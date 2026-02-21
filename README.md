# ☸️ Kubernetes Cluster Bootstrap with Ansible

> Automated deployment of a production-ready Kubernetes cluster (1 Master + N Workers) using Ansible — **100% idempotent, modular, tested in real conditions**.

---

## 🚀 What Gets Installed

| Component | Version | Description |
|-----------|---------|-------------|
| **Docker** | latest via `docker.io` | Container runtime with systemd cgroupdriver |
| **cri-dockerd** | `0.3.15` (configurable) | CRI shim for Docker with `--network-plugin=cni` |
| **CNI plugins** | `1.5.0` (configurable) | Container network interface plugins |
| **kubeadm / kubelet / kubectl** | `1.32.3` (configurable) | Kubernetes core tooling |
| **Calico** | `v3.28.0` | Pod network CNI — CIDR `10.244.0.0/16` |
| **Local Path Provisioner** | `v0.0.28` | Default storage class (auto-provisioning) |
| **Docker Compose** | `2.27.0` (configurable) | Docker Compose v2 CLI plugin |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────┐
│               Control Machine                    │
│        (master node / your workstation)          │
│           ansible-playbook site.yml              │
└───────────────┬─────────────────────────────────┘
                │ SSH
    ┌───────────┼───────────────────┐
    ▼           ▼                   ▼
┌─────────┐ ┌─────────┐       ┌─────────┐
│ master1 │ │ worker1 │  ...  │ worker2 │
│ control │ │         │       │         │
│  plane  │ │  node   │       │  node   │
└─────────┘ └─────────┘       └─────────┘
192.168.1.10  192.168.1.20    192.168.1.30
              (example IPs — replace with yours)
```

> ✅ **CIDR**: Pod network is `10.244.0.0/16` — does **not** overlap with typical `192.168.x.x` VM networks.

---

## 📋 Prerequisites

### Control machine (where you run Ansible)
- **Ubuntu 22.04** (tested) or any Linux/macOS
- **Python 3.10+** and **pip3**
- **Ansible ≥ 2.14** installed via pip (see installation below — **do NOT use `apt`**)
- **SSH key** (`~/.ssh/id_ed25519`) with access to all nodes

### Target nodes (master + workers)
- **Ubuntu 22.04** (or 20.04)
- **Passwordless sudo** configured
- **Internet access** during deployment

---

## 🐍 Ansible Installation (Control Machine)

> ⚠️ **Important**: Ubuntu 22.04 ships Ansible `2.12.0` via `apt` which is **too old** and causes `ansible-galaxy` errors. You **must** install Ansible via `pip3`.

### Step 1 — Install pip3

```bash
sudo apt update && sudo apt install -y python3-pip
```

### Step 2 — Install Ansible via pip

```bash
pip3 install --upgrade ansible
# Installs ansible-core 2.17.x
```

### Step 3 — Add pip binaries to PATH

```bash
export PATH="$HOME/.local/bin:$PATH"
```

Make it permanent:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Step 4 — Verify

```bash
ansible --version
# ansible [core 2.17.x]
# executable location = /home/<user>/.local/bin/ansible
```

### Step 5 — Install required Ansible collections

```bash
ansible-galaxy collection install -r requirements.yml
# Starting galaxy collection install process
# Nothing to do. All requested collections are already installed.
```

---

## 🔧 Step-by-Step Configuration

### Step 1 — Configure static IPs on your VMs

On each VM, configure a static IP with Netplan (`/etc/netplan/00-installer-config.yaml`).

### Step 2 — Configure SSH

```bash
# Generate an SSH key (on control machine)
ssh-keygen -t ed25519 -C "k8s-cluster"

# Copy to all nodes (replace IPs with yours)
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.10
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.20
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.30
```

### Step 3 — Configure passwordless sudo

On **each** node:
```bash
sudo visudo
# Add this line:
ubuntu ALL=(ALL) NOPASSWD:ALL
```

### Step 4 — ⚠️ Update IPs (REQUIRED)

> ⚠️ **Two files must be updated** with your actual VM IP addresses:

**`inventory.ini`**:
```
[masters]
master1 ansible_host=192.168.1.10   # ← replace with your master IP

[workers]
worker1 ansible_host=192.168.1.20   # ← replace with your worker IP
worker2 ansible_host=192.168.1.30   # ← replace with your worker IP

[all:vars]
ansible_user=ubuntu
ansible_become=true
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

**`ssh_config`**:
```
Host master1
    HostName 192.168.1.10   # ← replace with your master IP
Host worker1
    HostName 192.168.1.20   # ← replace
Host worker2
    HostName 192.168.1.30   # ← replace
```

### Step 5 — Customize versions (optional)

Edit `group_vars/all.yml` to change Kubernetes version, CIDR, or component versions.

---

## 🚀 Deployment

```bash
# 1. Test connectivity
ansible all -m ping

# 2. Check syntax
ansible-playbook site.yml --syntax-check

# 3. Deploy the cluster
ansible-playbook site.yml
```

---

## ✅ Post-deployment Verification

```bash
# Connect to the master
ssh ubuntu@192.168.1.10

# Check nodes (all must be Ready)
kubectl get nodes
# NAME      STATUS   ROLES           AGE   VERSION
# master1   Ready    control-plane   5m    v1.32.3
# worker1   Ready    <none>          4m    v1.32.3
# worker2   Ready    <none>          4m    v1.32.3

# Check all system pods
kubectl get pods -n kube-system

# Check default storage class
kubectl get storageclass

# Check Kubernetes version
kubectl version

# Check cluster info
kubectl cluster-info
```

---

## 🐳 Docker Compose

Docker Compose v2 is **automatically installed** by the `docker` role on all nodes. It does not interfere with Kubernetes (kubelet, cri-dockerd, Calico).

```bash
# Verify installation
docker compose version
# Docker Compose version v2.27.0
```

---

## ♻️ Idempotency

The `site.yml` playbook is **fully idempotent**: it can be run multiple times on an already-deployed cluster without errors or unintended changes. The following checks ensure idempotency:

- **cri-dockerd**: checked via `stat` before download
- **Docker Compose**: checked via `stat` before download
- **kubeadm init**: checked via presence of `/etc/kubernetes/admin.conf`
- **Calico / storage provisioner**: applied only on first cluster initialization
- **kubeadm config**: `/tmp/kubeadm-config.yaml` removed immediately after `kubeadm init`
- **apt-mark hold**: managed via `dpkg_selections` (natively idempotent)

---

## 🗑️ Uninstall

```bash
# Completely remove the cluster and all components
ansible-playbook -i inventory.ini uninstall.yml
```

The `uninstall.yml` playbook removes:
- kubelet, docker, cri-docker services
- Packages (docker.io, kubelet, kubeadm, kubectl)
- Kubernetes, Docker, CNI directories
- Kubernetes APT keyrings and sources
- Kube configs (`/root/.kube`, `/home/<user>/.kube`)
- Temporary files (`/tmp/cri-dockerd*`, `/tmp/cni.tgz`, etc.)
- Docker daemon configuration (`/etc/docker/daemon.json`)

---

## 🔍 Troubleshooting

### Nodes in `NotReady` state

```bash
kubectl get pods -n kube-system -l k8s-app=calico-node
kubectl logs -n kube-system -l k8s-app=calico-node
```

Verify that the pod CIDR (`10.244.0.0/16`) does not overlap with your VM network.

### `kubeadm init` fails with CRI error

```bash
sudo systemctl status cri-docker.service
sudo systemctl status cri-docker.socket
sudo systemctl restart cri-docker.socket cri-docker.service
```

### Workers cannot join the cluster

```bash
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 50
```

### Ansible SSH connection error

```bash
ansible all -m ping
# Verify IPs in inventory.ini AND ssh_config match actual VM IPs
ssh -i ~/.ssh/id_ed25519 ubuntu@<VM_IP>
```

### Manually reset a node

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd /root/.kube
```

### Docker Compose not found

```bash
ls -la /usr/local/lib/docker/cli-plugins/docker-compose
docker compose version
```

---

## 🗂️ Project Structure

```
.
├── ansible.cfg                          # Ansible configuration
├── inventory.ini                        # ⚠️ Update with your IPs
├── requirements.yml                     # Ansible Galaxy collection requirements
├── site.yml                             # Main playbook
├── uninstall.yml                        # Uninstall playbook
├── ssh_config                           # ⚠️ Update with your IPs
├── group_vars/
│   └── all.yml                          # Global variables
└── roles/
    ├── common/
    │   └── tasks/main.yml               # Swap, kernel modules, sysctl
    ├── docker/
    │   ├── handlers/main.yml            # Docker restart handler
    │   └── tasks/main.yml               # Docker + cri-dockerd + CNI + Compose
    ├── kubernetes/
    │   ├── handlers/main.yml            # kubelet restart handler
    │   └── tasks/main.yml               # kubeadm/kubelet/kubectl
    ├── master/
    │   ├── tasks/main.yml               # Cluster init, Calico, storage
    │   └── templates/kubeadm-config.yaml.j2
    └── worker/
        └── tasks/main.yml               # Join the cluster
```

---

## ⚙️ Available Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `kubernetes_version` | `"1.32.3"` | Kubernetes version (full semver required by kubeadm) |
| `pod_network_cidr` | `"10.244.0.0/16"` | Pod network CIDR for Calico (no overlap with `192.168.x.x`) |
| `cri_socket` | `"unix:///var/run/cri-dockerd.sock"` | CRI Docker socket path |
| `cri_dockerd_version` | `"0.3.15"` | cri-dockerd version |
| `cni_plugins_version` | `"1.5.0"` | CNI plugins version |
| `calico_manifest_url` | Calico v3.28.0 URL | Calico manifest URL |
| `docker_compose_version` | `"2.27.0"` | Docker Compose v2 version |
| `cluster_user` | `"{{ ansible_user }}"` | Cluster user |
| `ansible_become` | `true` | Automatic privilege escalation |
