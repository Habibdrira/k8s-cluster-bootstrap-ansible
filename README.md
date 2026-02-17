# Kubernetes Cluster Automation with Ansible

Simple Ansible project to automatically deploy a Kubernetes cluster (1 Master + Workers).

## What it installs

- Docker
- cri-dockerd
- kubeadm, kubelet, kubectl
- Calico network
- Local Path Storage

## Requirements

- Ubuntu servers
- SSH access to all nodes
- Ansible installed on control machine

## Configure inventory

Edit `inventory.ini` with your server IP addresses and SSH user.

## Deploy the cluster

ansible-playbook -i inventory.ini site.yml

## Verify
kubectl get nodes

Cluster should show all nodes in Ready state.



