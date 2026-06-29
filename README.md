# RKE2 Ansible Lab

Ansible project to install one RKE2 server and two RKE2 agents:

```txt
rke2-master-01   192.168.56.101
rke2-worker-01   192.168.56.102
rke2-worker-02   192.168.56.103
```

## Prerequisites

- Three Ubuntu VMs are already created.
- Static IPs are already configured.
- `openssh-server` is installed on all VMs.
- The Ansible control machine can SSH to all VMs.
- Update `ansible_user` in `inventory.ini` if your Ubuntu user is not `duc`.

## Validate SSH

With password auth:

```bash
ansible -i inventory.ini all -m ping --ask-pass --ask-become-pass
```

With SSH key auth:

```bash
ansible -i inventory.ini all -m ping
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

## Verify

On the master node:

```bash
sudo /var/lib/rancher/rke2/bin/kubectl get nodes -o wide \
  --kubeconfig /etc/rancher/rke2/rke2.yaml
```

Check services if needed:

```bash
sudo journalctl -u rke2-server -f
sudo journalctl -u rke2-agent -f
```
