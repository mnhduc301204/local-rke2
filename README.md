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
- Update `ansible_user` in `inventory.ini` if your Ubuntu user is not `duc`.
- Set `vault_cloudflare_tunnel_token` in an encrypted Ansible Vault file when you want Ansible to install and start the Cloudflare Tunnel service.

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
cp group_vars/secrets.example.yml group_vars/vault.yml
ansible-vault encrypt group_vars/vault.yml
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
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
