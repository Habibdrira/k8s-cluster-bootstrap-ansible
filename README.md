# Kubernetes Cluster Automation with Ansible

Déploiement automatisé d'un cluster Kubernetes (1 Master + Workers) avec Ansible.

## 🚀 Ce qui est installé

- **Docker** avec configuration optimisée (cgroupdriver systemd)
- **cri-dockerd** v0.3.15 (téléchargé depuis GitHub, configuré avec `--network-plugin=cni`)
- **CNI plugins** v1.5.0
- **kubeadm, kubelet, kubectl** v1.29
- **Calico** network CNI (CIDR des pods : `192.168.0.0/16`, compatible Calico)
- **Local Path Storage Provisioner** (storage class par défaut)

## 📋 Prérequis

- **Machines Ubuntu 22.04** (ou 20.04)
- **Accès SSH** à tous les nœuds
- **Ansible** installé sur la machine de contrôle (version 2.9+)
- **Clé SSH** configurée (`~/.ssh/id_ed25519`)
- **Accès à Internet** depuis chaque VM pendant le déploiement (pour télécharger cri-dockerd, CNI plugins, Calico et local-path-provisioner)

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

> ⚠️ **Important** : Les adresses IP `192.168.1.10`, `192.168.1.20` et `192.168.1.30` sont des **exemples**. Remplacez-les par les vraies IPs de vos VMs dans `inventory.ini` **et** dans `ssh_config`.

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

## ✅ Vérification post-déploiement

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

# Vérifier tous les pods système
kubectl get pods -n kube-system

# Vérifier la storage class par défaut
kubectl get storageclass

# Vérifier la version de Kubernetes
kubectl version

# Vérifier les infos du cluster
kubectl cluster-info
```

## 🐳 Docker Compose

Docker Compose peut être installé **sans aucun conflit** avec Kubernetes. C'est un outil indépendant qui n'interfère pas avec kubelet, cri-dockerd ni Calico.

Pour l'installer manuellement sur un nœud :

```bash
# Créer le répertoire des plugins Docker CLI
sudo mkdir -p /usr/local/lib/docker/cli-plugins

# Télécharger Docker Compose v2
sudo curl -SL https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose

# Rendre exécutable
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose

# Vérifier
docker compose version
```

## 🗑️ Désinstallation

Pour désinstaller complètement le cluster, utilisez le playbook `uninstall.yml` inclus dans le dépôt :

```bash
# Désinstaller le cluster
ansible-playbook -i inventory.ini uninstall.yml
```

## 🔍 Dépannage / Troubleshooting

### Problème 1 — Les nœuds restent en état `NotReady`

```bash
# Vérifier les pods Calico
kubectl get pods -n kube-system -l k8s-app=calico-node
# Vérifier les logs
kubectl logs -n kube-system -l k8s-app=calico-node
```

### Problème 2 — `kubeadm init` échoue avec une erreur CRI

```bash
# Vérifier que cri-dockerd tourne
sudo systemctl status cri-docker.service
sudo systemctl status cri-docker.socket
# Relancer si nécessaire
sudo systemctl restart cri-docker.socket cri-docker.service
```

### Problème 3 — Les workers ne rejoignent pas le cluster

```bash
# Vérifier que kubelet tourne sur le worker
sudo systemctl status kubelet
# Vérifier les logs kubelet
sudo journalctl -u kubelet -n 50
```

### Problème 4 — Erreur de connexion SSH Ansible

```bash
# Tester la connectivité
ansible all -i inventory.ini -m ping
# Vérifier les clés SSH
ssh -i ~/.ssh/id_ed25519 ubuntu@<IP_VM>
```

### Problème 5 — Réinitialiser un nœud manuellement

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd ~/.kube
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
