# Homelab Ansible

Infrastructure automation for a hybrid Proxmox/Kubernetes homelab running on bare metal.

## What This Does

This repository manages the complete lifecycle of homelab infrastructure:
- Proxmox bare metal node management
- Ubuntu VM provisioning via cloud-init templates
- k3s Kubernetes cluster deployment over Tailscale
- Service deployment and configuration
- Security hardening
- Backup and snapshot automation

---

## Architecture

```
[ Proxmox Cluster — bare metal nodes ]
  ├── LXC Containers (existing services)
  │     ├── Gitea (source of truth)
  │     ├── Mattermost
  │     ├── n8n
  │     └── ansible-ctrl (Ansible control node)
  └── VMs (k3s cluster)
        ├── k3s-master
        ├── k3s-worker-1
        ├── k3s-worker-2
        ├── k3s-worker-3
        └── k3s-worker-4

[ Tailscale Mesh — encrypted WireGuard overlay ]
[ Cloud Edge Node — public ingress ]
```

---

## Prerequisites

### Control Node Requirements
- Ubuntu/Debian Linux (LXC or VM)
- Python 3.10+
- Ansible 2.17+
- SSH key pair

### Install Ansible

```bash
sudo apt update && sudo apt install pipx -y
pipx install --include-deps ansible
pipx ensurepath
# Restart terminal then verify
ansible --version
```

### Install Required Collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### Generate SSH Key

```bash
ssh-keygen -t ed25519 -C "ansible-control-node"
```

---

## Network Layout

Replace the placeholder values below with your actual network addresses before running any playbooks.

### Proxmox Hosts

| Variable | Description | Example |
|----------|-------------|---------|
| `MGMT_SUBNET` | Management subnet | `192.168.x.0/24` |
| `GATEWAY` | Default gateway | `192.168.x.1` |
| `NODE_1_IP` through `NODE_N_IP` | Proxmox host IPs | `192.168.x.9` |

### k3s Cluster

| Variable | Description | Example |
|----------|-------------|---------|
| `K3S_MASTER_LAN_IP` | Master LAN IP | `192.168.x.20` |
| `K3S_MASTER_TAILSCALE_IP` | Master Tailscale IP | `100.x.x.x` |
| `K3S_WORKER_N_LAN_IP` | Worker LAN IPs | `192.168.x.21-24` |
| `K3S_WORKER_N_TAILSCALE_IP` | Worker Tailscale IPs | `100.x.x.x` |

### Containers

| Variable | Description | Example |
|----------|-------------|---------|
| `ANSIBLE_CTRL_IP` | Ansible control node | `192.168.x.108` |
| `DNS_IP` | Pi-hole or AdGuard | `192.168.x.100` |

---

## Quick Start — New Node

```bash
# 1. Add your user to the node (as root on target)
useradd -m -s /bin/bash YOUR_USER
passwd YOUR_USER
usermod -aG sudo YOUR_USER
apt install sudo -y

# 2. Copy SSH key from control node
ssh-copy-id YOUR_USER@<node-ip>

# 3. Add node to inventory
nano inventory/hosts.ini

# 4. Bootstrap ansible service account
ansible-playbook playbooks/setup-ansible-user-dale.yml --limit <node-name>

# 5. Verify
ansible <node-name> -m ping

# 6. Commit
git add inventory/hosts.ini
git commit -m "Add <node-name> to inventory"
git push
```

---

## Playbook Reference — Full Deployment Order

### Phase 1 — Proxmox Node Management

```bash
# Validate all Proxmox nodes meet requirements
ansible-playbook playbooks/check-resources.yml

# Update all Proxmox nodes
ansible-playbook playbooks/update_nodes.yml

# Snapshot everything before making changes
ansible-playbook playbooks/snapshot-all.yml
```

### Phase 2 — k3s VM Provisioning

```bash
# Create Ubuntu 22.04 cloud-init template on each Proxmox node
# Requires Ubuntu 22.04 cloud image downloaded to /tmp/ on each node:
# wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
ansible-playbook playbooks/create-template-per-node.yml \
  --limit node-1,node-2,node-3,node-4,node-5

# Clone templates into k3s VMs
ansible-playbook playbooks/create-k3s-vms.yml

# Prepare VMs for k3s (disable swap, kernel modules, sysctl)
ansible-playbook playbooks/node-prep.yml \
  -e "target_hosts=k3s_masters,k3s_workers"
```

### Phase 3 — Tailscale

```bash
# Store Tailscale auth key in vault first
ansible-vault create group_vars/k3s.yml
# Add: vault_tailscale_authkey: "tskey-auth-YOUR-KEY"

# Install and authenticate Tailscale on all k3s VMs
ansible-playbook playbooks/install-tailscale.yml --ask-vault-pass
```

### Phase 4 — k3s Deployment

```bash
# Final pre-flight check — all nodes must show READY FOR K3S
ansible-playbook playbooks/check-resources.yml \
  -e "target_hosts=k3s_masters,k3s_workers"

# Deploy k3s master (saves node token to /tmp/k3s-node-token)
ansible-playbook playbooks/k3s-master.yml

# Join worker nodes to cluster
ansible-playbook playbooks/k3s-workers.yml

# Verify cluster is healthy
kubectl get nodes -o wide
```

### Phase 5 — Management Tools

```bash
# Set up kubectl on your Ansible control node
ansible-playbook playbooks/setup-kubectl.yml --limit ansible-ctrl

# Verify from control node
kubectl get nodes -o wide
kubectl get pods -A
```

### Daily Operations

```bash
# Update Proxmox hosts
ansible-playbook playbooks/update_nodes.yml

# Update k3s VMs
ansible-playbook playbooks/update_nodes.yml \
  -e "target_hosts=k3s_masters,k3s_workers"

# Snapshot before any major change
ansible-playbook playbooks/snapshot-all.yml \
  -e "snapshot_name=pre-change-$(date +%Y-%m-%d-%H%M)"

# Check cluster health
kubectl get nodes -o wide
kubectl get pods -A
```

---

## Inventory Structure

```ini
# inventory/hosts.ini

[proxmox_nodes]
node-1  ansible_host=YOUR_NODE_1_IP
node-2  ansible_host=YOUR_NODE_2_IP
# ... add all Proxmox hosts

[containers]
ansible-ctrl  ansible_host=YOUR_CTRL_IP
dns           ansible_host=YOUR_DNS_IP
# ... add all LXC containers

[vms]
pbs  ansible_host=YOUR_PBS_IP

[k3s_masters]
k3s-master  ansible_host=YOUR_K3S_MASTER_LAN_IP  tailscale_ip=YOUR_K3S_MASTER_TS_IP

[k3s_workers]
k3s-worker-1  ansible_host=YOUR_WORKER_1_LAN_IP  tailscale_ip=YOUR_WORKER_1_TS_IP
k3s-worker-2  ansible_host=YOUR_WORKER_2_LAN_IP  tailscale_ip=YOUR_WORKER_2_TS_IP
k3s-worker-3  ansible_host=YOUR_WORKER_3_LAN_IP  tailscale_ip=YOUR_WORKER_3_TS_IP
k3s-worker-4  ansible_host=YOUR_WORKER_4_LAN_IP  tailscale_ip=YOUR_WORKER_4_TS_IP

[oracle_edge]
# oracle-1  ansible_host=YOUR_ORACLE_TAILSCALE_IP

[all:vars]
ansible_user=ansible
ansible_python_interpreter=/usr/bin/python3

[proxmox_nodes:vars]
ansible_become=true
ansible_become_method=sudo

[containers:vars]
ansible_become=true
ansible_become_method=sudo

[vms:vars]
ansible_become=true
ansible_become_method=sudo

[k3s_masters:vars]
ansible_become=true
ansible_become_method=sudo

[k3s_workers:vars]
ansible_become=true
ansible_become_method=sudo

[oracle_edge:vars]
ansible_user=ubuntu
ansible_become=true
ansible_become_method=sudo
```

---

## Variable Files

```
group_vars/
├── all.yml          ← shared variables for all hosts
└── k3s.yml          ← k3s secrets (encrypted, NOT in git)

host_vars/
└── <hostname>/
    ├── vars.yml     ← per-host variables (safe to commit)
    └── vault.yml    ← per-host secrets (encrypted, NOT in git)
```

### group_vars/all.yml — Customize These

```yaml
# group_vars/all.yml
ansible_user: ansible
ansible_python_interpreter: /usr/bin/python3

ntp_server: YOUR_DNS_IP
dns_servers:
  - YOUR_DNS_IP
  - 8.8.8.8

gitea_url: http://YOUR_GITEA_IP
timezone: YOUR_TIMEZONE   # e.g. America/Chicago

common_packages:
  - curl
  - wget
  - git
  - vim
  - htop
  - tmux
  - net-tools
```

### group_vars/k3s.yml — Secrets (Encrypted)

```yaml
# group_vars/k3s.yml — NEVER commit unencrypted
vault_tailscale_authkey: "tskey-auth-YOUR-KEY-HERE"
```

---

## Secrets Management

All secrets use Ansible Vault — never stored in plaintext:

```bash
# Create new encrypted vault file
ansible-vault create group_vars/k3s.yml

# Edit existing vault file
ansible-vault edit group_vars/k3s.yml

# View without decrypting to disk
ansible-vault view group_vars/k3s.yml

# Run playbook with vault
ansible-playbook playbooks/install-tailscale.yml --ask-vault-pass

# Or use a vault password file (add to ansible.cfg, never commit)
echo "your-vault-password" > ~/.vault_pass
chmod 600 ~/.vault_pass
# Then add to ansible.cfg:
# vault_password_file = ~/.vault_pass
```

---

## Playbook Directory

| Playbook | Purpose | When to Run |
|----------|---------|-------------|
| `check-resources.yml` | Pre-flight validation | Before any major change |
| `snapshot-all.yml` | Proxmox CT/VM snapshots | Before any major change |
| `update_nodes.yml` | Update all packages | Regularly |
| `setup-ansible-user.yml` | Create ansible service account | New nodes (ansible SSH already works) |
| `setup-ansible-user-dale.yml` | Bootstrap as your user | New nodes (manual SSH works) |
| `create-template-per-node.yml` | Ubuntu 22.04 cloud-init templates | One time per node |
| `create-k3s-vms.yml` | Clone VMs for k3s | Initial k3s setup |
| `node-prep.yml` | Prepare nodes for k3s | Before k3s install |
| `install-tailscale.yml` | Tailscale on k3s VMs | Before k3s install |
| `k3s-master.yml` | Deploy k3s master | Initial k3s setup |
| `k3s-workers.yml` | Join k3s workers | Initial k3s setup |
| `setup-kubectl.yml` | kubectl on management nodes | After k3s is running |
| `harden-nodes.yml` | Security hardening | After services are stable |

---

## ansible.cfg Reference

```ini
[defaults]
inventory = inventory/hosts.ini
remote_user = ansible
host_key_checking = False
timeout = 30
forks = 10
# vault_password_file = ~/.vault_pass
# log_path = ~/ansible/ansible.log

[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = false

[ssh_connection]
pipelining = true
ssh_args = -o StrictHostKeyChecking=no -o ControlMaster=auto -o ControlPersist=60s
```

---

## Common Commands

```bash
# Test connectivity to all hosts
ansible all -m ping

# Run ad-hoc command across all nodes
ansible proxmox_nodes -m command -a "uptime"

# Check disk space across cluster
ansible proxmox_nodes -m command -a "df -h /" --become

# List all VMs on all Proxmox nodes
ansible proxmox_nodes -m shell -a "qm list" --become

# List all containers on all Proxmox nodes
ansible proxmox_nodes -m shell -a "pct list" --become

# Dry run any playbook
ansible-playbook playbooks/update_nodes.yml --check --diff

# Target specific hosts
ansible-playbook playbooks/update_nodes.yml --limit node-1,node-3

# Run with verbose output
ansible-playbook playbooks/update_nodes.yml -vv

# Check playbook syntax
ansible-playbook playbooks/update_nodes.yml --syntax-check
```

---

## kubectl Quick Reference

```bash
# Cluster status
kubectl get nodes -o wide
kubectl get pods -A
kubectl get services -A

# Useful aliases (added to .bashrc by setup-kubectl.yml)
k        # kubectl
kgn      # kubectl get nodes -o wide
kgp      # kubectl get pods -A
kgs      # kubectl get services -A
kgd      # kubectl get deployments -A
kga      # kubectl get all -A
kdp      # kubectl describe pod
kdn      # kubectl describe node
klog     # kubectl logs
```

---

## Security Notes

- Root SSH login is disabled on all nodes
- Ansible uses a dedicated service account (`ansible`) with NOPASSWD sudo
- All secrets encrypted with Ansible Vault
- k3s cluster traffic routes over encrypted Tailscale WireGuard mesh
- Vault files are excluded from git via `.gitignore`

---

## See Also

- `ANSIBLE.md` — Complete Ansible reference, module guide, and 25 mastery tasks
- `requirements.yml` — Collection dependencies

---

*Homelab Ansible — Infrastructure as Code for self-hosted homelab environments*
