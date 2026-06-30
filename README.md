# RKE2 Ansible Lab

Ansible project to install one RKE2 server, two RKE2 agents, and one load balancer VM running HAProxy plus optional Cloudflare Tunnel:

```txt
rke2-lb-01       192.168.56.100
rke2-master-01   192.168.56.101
rke2-worker-01   192.168.56.102
rke2-worker-02   192.168.56.103
```

## Prerequisites

- Four Ubuntu VMs are already created.
- Static IPs are already configured.
- `openssh-server` is installed on all VMs.
- The Ansible control machine can SSH to all VMs.
- The VMs have outbound internet access through the hypervisor network, for example NAT.
- Update `ansible_user` in `inventory.ini` if your Ubuntu user is not `duc`.
- Set `vault_cloudflare_tunnel_token` in an encrypted `group_vars/all/vault.yml` file when you want Ansible to install and start the Cloudflare Tunnel service.

## Network Exposure

Only `rke2-lb-01` should receive application traffic from outside the lab, through Cloudflare Tunnel and HAProxy. Master and worker nodes do not need public IP addresses.

## VM Networking

Ansible writes netplan files inside the VMs, but it cannot create missing virtual NICs unless the hypervisor is managed separately. In the simple lab setup, keep each VM on the NAT/private network that already lets the nodes reach each other and reach the internet.

The default vars assume VMware NAT-style routing:

```txt
VM subnet:       192.168.56.0/24
Default gateway: 192.168.56.2
Interface:       ens33
```

If your hypervisor uses a different gateway or interface name, update `lab_gateway_ip` and `lab_netplan_hosts` in `group_vars/all/vars.yml` before applying netplan. For Cloudflare Tunnel, `rke2-lb-01` must be on a NAT or bridged network with outbound internet access, not a host-only network.

Check interface names first:

```bash
ansible -i inventory.ini all_lab -m command -a "ip -br link" --ask-pass --ask-become-pass
```

Then update `lab_netplan_hosts` in `group_vars/all/vars.yml` if your interface name is not `ens33`. Netplan is enabled by default:

```yaml
lab_configure_netplan: true
lab_gateway_ip: "192.168.56.2"
lab_netplan_disable_existing: true
```

When `lab_netplan_disable_existing` is enabled, the netplan role renders `/etc/netplan/99-rke2-lab.yaml`, backs up other `/etc/netplan/*.yaml` and `/etc/netplan/*.yml` files into `/etc/netplan/ansible-backup`, then removes those unmanaged files before `netplan generate` and `netplan apply`. This prevents old DHCP/static definitions for the same interface from being merged with the lab route.

Run only the network step first:

```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass --ask-pass --ask-become-pass --tags network
```

You can also disable the final ping check without changing routing:

```yaml
lab_check_internet_connectivity: false
```

## Validate SSH

With password auth:

```bash
ansible -i inventory.ini all_lab -m ping --ask-pass --ask-become-pass
```

With SSH key auth:

```bash
ansible -i inventory.ini all_lab -m ping
```

## Install RKE2

With password auth:

```bash
ansible-playbook -i inventory.ini site.yml --ask-pass --ask-become-pass
```

With SSH key auth:

```bash
ansible-playbook -i inventory.ini site.yml
```

With encrypted secrets:

```bash
cp -n group_vars/all/vault.yml.example group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml
ansible-playbook -i inventory.ini site.yml --ask-vault-pass --ask-pass --ask-become-pass
```

If `group_vars/all/vault.yml` already exists, edit it instead of copying the example over it:

```bash
ansible-vault edit group_vars/all/vault.yml
```

Set at least these secret values:

```yaml
vault_cloudflare_tunnel_token: "..."
vault_rancher_bootstrap_password: "..."
```

## Rancher

Set the Rancher hostname and Let's Encrypt email in `group_vars/all/vars.yml` before running the playbook:

```yaml
rancher_hostname: "rancher.example.com"
letsencrypt_email: "admin@example.com"
```

The playbook keeps TLS responsibilities separate:

```txt
helm          -> installs Helm on the master node
cert_manager  -> installs cert-manager with Helm
cert_issuer   -> creates the Let's Encrypt ClusterIssuer
rancher       -> installs Rancher with Helm using external TLS termination
```

After the playbook completes, add a Cloudflare Tunnel public hostname for the same Rancher hostname and point it to the local HAProxy frontend on the load balancer:

```txt
Type: HTTP
URL: http://localhost:80
```

Rancher is installed with:

```txt
tls=external
```

so public TLS is terminated by Cloudflare and the tunnel forwards HTTP to HAProxy.

For Let's Encrypt HTTP-01 validation, the Cloudflare public hostname must route to the cluster ingress. If the certificate stays pending, confirm the public hostname exists before rerunning the `cert_issuer` and `rancher` roles.

Because Cloudflare terminates public TLS before forwarding traffic through the tunnel, the HTTP frontend on HAProxy sets the forwarded protocol headers before sending traffic to ingress-nginx:

```haproxy
http-request set-header X-Forwarded-Proto https
http-request set-header X-Forwarded-Ssl on
```

These headers belong in HAProxy rather than the Rancher Ingress because HAProxy is the first local hop after `cloudflared` and knows that public client traffic arrived over HTTPS.

The playbook also patches the RKE2 ingress-nginx controller ConfigMap with:

```yaml
use-forwarded-headers: "true"
compute-full-forwarded-for: "true"
```

so ingress-nginx forwards the HAProxy headers to Rancher instead of replacing them with its local HTTP scheme.

Then open:

```txt
https://rancher.example.com
```

## Verify

On the master node:

```bash
sudo /var/lib/rancher/rke2/bin/kubectl get nodes -o wide \
  --kubeconfig /etc/rancher/rke2/rke2.yaml

sudo /var/lib/rancher/rke2/bin/kubectl get svc -n kube-system ingress-nginx-nodeport \
  --kubeconfig /etc/rancher/rke2/rke2.yaml

sudo /var/lib/rancher/rke2/bin/kubectl get endpoints -n kube-system ingress-nginx-nodeport \
  --kubeconfig /etc/rancher/rke2/rke2.yaml
```

On the load balancer node:

```bash
curl -I http://192.168.56.100
sudo systemctl status haproxy
sudo systemctl status cloudflared
```

Check services if needed:

```bash
sudo journalctl -u rke2-server -f
sudo journalctl -u rke2-agent -f
sudo journalctl -u haproxy -f
sudo journalctl -u cloudflared -f
```

HAProxy stats are available at:

```txt
http://192.168.56.100:8404/stats
```
