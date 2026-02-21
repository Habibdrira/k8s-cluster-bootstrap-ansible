# Kubernetes Cluster Bootstrap avec Ansible

Déploiement automatisé d'un cluster Kubernetes (1 Master + Workers) avec Ansible — **100% idempotent, modulaire et sans erreurs**.

---

## 🚀 Ce qui est installé

| Composant | Version | Description |
|-----------|---------|-------------|
| **Docker** | dernière via `docker.io` | Runtime de conteneurs avec cgroupdriver systemd |
| **cri-dockerd** | v0.3.15 | Shim CRI pour Docker, configuré avec `--network-plugin=cni` |
| **CNI plugins** | v1.5.0 | Plugins réseau pour les conteneurs |
| **kubeadm / kubelet / kubectl** | **1.32** | Outils Kubernetes |
| **Calico** | v3.28.0 | CNI réseau des pods (CIDR : `10.244.0.0/16`) |
| **Local Path Provisioner** | v0.0.28 | Storage class par défaut |
| **Docker Compose** | v2.27.0 | Plugin CLI Docker Compose v2 (installé automatiquement) |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────┐
│                  Machine de contrôle             │
│              (votre poste / CI)                  │
│         ansible-playbook site.yml                │
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
              (IPs d'exemple — à remplacer)
```

> ⚠️ **CIDR important** : Le sous-réseau des VMs est `192.168.1.x`. Calico utilise donc `10.244.0.0/16` pour les pods afin d'éviter tout conflit de routage.

---

## 📋 Prérequis

- **Machines Ubuntu 22.04** (ou 20.04) — 1 master + N workers
- **Accès SSH** à tous les nœuds avec clé ed25519 (`~/.ssh/id_ed25519`)
- **Ansible** ≥ 2.9 installé sur la machine de contrôle
- **Accès Internet** depuis chaque VM pendant le déploiement

---

## 🔧 Configuration pas à pas

### Étape 1 — Configurer les IPs statiques sur vos VMs

Sur chaque VM, configurez une IP statique avec Netplan (`/etc/netplan/00-installer-config.yaml`).

### Étape 2 — Configurer SSH

```bash
# Générer une clé SSH
ssh-keygen -t ed25519 -C "k8s-cluster"

# Copier sur toutes les VMs (remplacez les IPs par les vôtres)
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.10
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.20
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.30
```

### Étape 3 — Configurer sudo sans mot de passe

Sur chaque VM :
```bash
sudo visudo
# Ajouter : ubuntu ALL=(ALL) NOPASSWD:ALL
```

### Étape 4 — ⚠️ Mettre à jour les IPs (OBLIGATOIRE)

> ⚠️ **Double mise à jour obligatoire** : Vous devez mettre à jour les adresses IP dans **deux fichiers** :

**`inventory.ini`** :
```ini
[masters]
master1 ansible_host=192.168.1.10   # ← remplacer par votre IP

[workers]
worker1 ansible_host=192.168.1.20   # ← remplacer par votre IP
worker2 ansible_host=192.168.1.30   # ← remplacer par votre IP

[all:vars]
ansible_user=ubuntu
ansible_become=true
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

**`ssh_config`** :
```
Host master1
    HostName 192.168.1.10   # ← remplacer par votre IP
...
```

### Étape 5 — Personnaliser les variables (optionnel)

Éditez `group_vars/all.yml` si vous souhaitez modifier les versions ou le CIDR.

---

## 🚀 Déploiement

```bash
# 1. Tester la connectivité
ansible all -m ping

# 2. Vérifier la syntaxe
ansible-playbook site.yml --syntax-check

# 3. Déployer le cluster
ansible-playbook site.yml
```

---

## ✅ Vérification post-déploiement

```bash
# Se connecter au master
ssh ubuntu@192.168.1.10

# Vérifier les nœuds (tous doivent être Ready)
kubectl get nodes
# NAME      STATUS   ROLES           AGE   VERSION
# master1   Ready    control-plane   5m    v1.32.x
# worker1   Ready    <none>          4m    v1.32.x
# worker2   Ready    <none>          4m    v1.32.x

# Vérifier tous les pods système
kubectl get pods -n kube-system

# Vérifier la storage class par défaut
kubectl get storageclass

# Vérifier la version de Kubernetes
kubectl version

# Vérifier les infos du cluster
kubectl cluster-info
```

---

## 🐳 Docker Compose

Docker Compose v2 est **installé automatiquement** par le rôle `docker` sur tous les nœuds. Il n'interfère pas avec Kubernetes (kubelet, cri-dockerd, Calico).

```bash
# Vérifier l'installation
docker compose version
# Docker Compose version v2.27.0
```

---

## ♻️ Idempotence

Le playbook `site.yml` est **entièrement idempotent** : il peut être relancé plusieurs fois sur un cluster déjà déployé sans causer d'erreurs ni de modifications non désirées. Les vérifications suivantes garantissent l'idempotence :

- **cri-dockerd** : vérifié via `stat` avant téléchargement
- **Docker Compose** : vérifié via `stat` avant téléchargement
- **kubeadm init** : vérifié via la présence de `/etc/kubernetes/admin.conf`
- **Calico / storage provisioner** : appliqués uniquement si le cluster vient d'être initialisé
- **apt-mark hold** : géré via `dpkg_selections` (idempotent nativement)

---

## 🗑️ Désinstallation

```bash
# Désinstaller complètement le cluster et tous les composants
ansible-playbook -i inventory.ini uninstall.yml
```

Le playbook `uninstall.yml` supprime :
- Les services kubelet, docker, cri-docker
- Les packages (docker.io, kubelet, kubeadm, kubectl)
- Les répertoires Kubernetes, Docker, CNI
- Les keyrings et sources APT Kubernetes
- Les configs kube (`~/.kube`) pour root et l'utilisateur
- Les fichiers temporaires (`/tmp/cri-dockerd*`, `/tmp/cni.tgz`, etc.)
- La configuration Docker daemon (`/etc/docker/daemon.json`)

---

## 🔍 Dépannage

### Nœuds en état `NotReady`

```bash
kubectl get pods -n kube-system -l k8s-app=calico-node
kubectl logs -n kube-system -l k8s-app=calico-node
```

Vérifiez que le CIDR des pods ne chevauche pas le réseau des VMs.

### `kubeadm init` échoue avec une erreur CRI

```bash
sudo systemctl status cri-docker.service
sudo systemctl status cri-docker.socket
sudo systemctl restart cri-docker.socket cri-docker.service
```

### Les workers ne rejoignent pas le cluster

```bash
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 50
```

### Erreur de connexion SSH Ansible

```bash
ansible all -m ping
# Vérifier que les IPs dans inventory.ini ET ssh_config correspondent aux vraies IPs
ssh -i ~/.ssh/id_ed25519 ubuntu@<IP_VM>
```

### Réinitialiser un nœud manuellement

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd ~/.kube
```

### Docker Compose non trouvé

```bash
ls -la /usr/local/lib/docker/cli-plugins/docker-compose
docker compose version
```

---

## 🗂️ Structure du projet

```
.
├── ansible.cfg                          # Configuration Ansible
├── inventory.ini                        # ⚠️ À mettre à jour avec vos IPs
├── site.yml                             # Playbook principal
├── uninstall.yml                        # Playbook de désinstallation
├── ssh_config                           # ⚠️ À mettre à jour avec vos IPs
├── group_vars/
│   └── all.yml                          # Variables globales
└── roles/
    ├── common/
    │   └── tasks/main.yml               # Swap, modules kernel, sysctl
    ├── docker/
    │   ├── handlers/main.yml            # Handler restart Docker
    │   └── tasks/main.yml               # Docker + cri-dockerd + CNI + Compose
    ├── kubernetes/
    │   ├── handlers/main.yml            # Handler restart kubelet
    │   └── tasks/main.yml               # kubeadm/kubelet/kubectl
    ├── master/
    │   ├── tasks/main.yml               # Init cluster, Calico, storage
    │   └── templates/kubeadm-config.yaml.j2
    └── worker/
        └── tasks/main.yml               # Rejoindre le cluster
```

---

## ⚙️ Variables disponibles

| Variable | Valeur par défaut | Description |
|----------|-------------------|-------------|
| `kubernetes_version` | `"1.32"` | Version de Kubernetes à installer |
| `pod_network_cidr` | `"10.244.0.0/16"` | CIDR réseau des pods (Calico) |
| `cri_socket` | `"unix:///var/run/cri-dockerd.sock"` | Socket CRI Docker |
| `calico_manifest_url` | URL Calico v3.28.0 | URL du manifest Calico |
| `docker_compose_version` | `"2.27.0"` | Version de Docker Compose v2 |
| `cluster_user` | `"{{ ansible_user }}"` | Utilisateur du cluster |
| `ansible_become` | `true` | Élévation de privilèges automatique |
