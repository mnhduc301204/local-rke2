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
- The load balancer VM has internet access. By default Ansible lets private RKE2 nodes initiate outbound internet traffic through the load balancer, while blocking internet-initiated forwarded traffic back to those private nodes.
- Update `ansible_user` in `inventory.ini` if your Ubuntu user is not `duc`.
- Set `vault_cloudflare_tunnel_token` in an encrypted `group_vars/all/vault.yml` file when you want Ansible to install and start the Cloudflare Tunnel service.

## Network Exposure

Only `rke2-lb-01` should be exposed outside the lab network. Master and worker nodes stay on the private `192.168.56.0/24` network.

Traffic inside `192.168.56.0/24` is allowed normally. The NAT rules only let private nodes initiate outbound internet connections through the load balancer and block internet-initiated forwarded traffic to those private nodes.

Set this in `group_vars/all/vars.yml` when you want to control whether private nodes can initiate outbound internet connections through the load balancer:

```yaml
lab_allow_private_nodes_outbound_internet: true
```

Set it to `false` to avoid adding the NAT/default-route rules. In that mode, online installs that use `apt` or `https://get.rke2.io` need another package source or an offline install path.

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
