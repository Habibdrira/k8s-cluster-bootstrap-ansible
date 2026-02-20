# Kubernetes Cluster Automation with Ansible

Déploiement automatisé d'un cluster Kubernetes (1 Master + Workers) avec Ansible.

## 🚀 Ce qui est installé

- **Docker** avec configuration optimisée (cgroupdriver systemd)
- **cri-dockerd** v0.3.15 (téléchargé depuis GitHub)
- **CNI plugins** v1.5.0
- **kubeadm, kubelet, kubectl** v1.29
- **Calico** network CNI
- **Local Path Storage Provisioner** (storage class par défaut)

## 📋 Prérequis

- **Machines Ubuntu 22.04** (ou 20.04)
- **Accès SSH** à tous les nœuds
- **Ansible** installé sur la machine de contrôle (version 2.9+)
- **Clé SSH** configurée (`~/.ssh/id_ed25519`)

## 🔧 Configuration

### 1. Configurer les adresses IP statiques sur vos VMs

Configurez les IPs statiques avec Netplan sur chaque VM (fichier `/etc/netplan/00-installer-config.yaml`).

### 2. Configurer SSH

```bash
# Générer une clé SSH
ssh-keygen -t ed25519 -C "k8s-cluster"

# Copier sur toutes les VMs
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.10
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.20
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.30
```

### 3. Configurer sudo sans mot de passe

Sur chaque VM :
```bash
sudo visudo
# Ajouter : ubuntu ALL=(ALL) NOPASSWD:ALL
```

### 4. Modifier l'inventaire

Éditez `inventory.ini` avec vos adresses IP :

```ini
[masters]
master1 ansible_host=192.168.1.10

[workers]
worker1 ansible_host=192.168.1.20
worker2 ansible_host=192.168.1.30

[all:vars]
ansible_user=ubuntu
ansible_become=true
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

### 5. Personnaliser les variables (optionnel)

Éditez `group_vars/all.yml` pour modifier :
- Version de Kubernetes
- Réseau des pods
- URL Calico

## 🚀 Déploiement

```bash
# Tester la connectivité
ansible all -i inventory.ini -m ping

# Tester sudo
ansible all -i inventory.ini -m shell -a "sudo whoami"

# Vérifier la syntaxe
ansible-playbook site.yml --syntax-check

# Déployer le cluster (mode dry-run)
ansible-playbook -i inventory.ini site.yml --check

# Déployer le cluster
ansible-playbook -i inventory.ini site.yml
```

## ✅ Vérification

```bash
# Se connecter au master
ssh ubuntu@192.168.1.10

# Vérifier les nœuds
kubectl get nodes

# Doit afficher :
# NAME      STATUS   ROLES           AGE   VERSION
# master1   Ready    control-plane   5m    v1.29.x
# worker1   Ready    <none>          4m    v1.29.x
# worker2   Ready    <none>          4m    v1.29.x
```

## 🗂️ Structure du projet

```
.
├── ansible.cfg
├── inventory.ini
├── site.yml
├── uninstall.yml
├── ssh_config
├── group_vars/
│   └── all.yml
└── roles/
    ├── common/
    │   └── tasks/main.yml
    ├── docker/
    │   ├── handlers/main.yml
    │   └── tasks/main.yml
    ├── kubernetes/
    │   └── tasks/main.yml
    ├── master/
    │   ├── tasks/main.yml
    │   └── templates/kubeadm-config.yaml.j2
    └── worker/
        └── tasks/main.yml
```
