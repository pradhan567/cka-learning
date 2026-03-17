# Kubeadm Cluster Setup & Management

> **CKA Domain:** Cluster Architecture, Installation & Configuration (25%)
> **Official Docs:** https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/

---

## Table of Contents

1. [What is kubeadm?](#1-what-is-kubeadm)
2. [Prerequisites](#2-prerequisites)
3. [Cluster Installation](#3-cluster-installation)
4. [Join Worker Nodes](#4-join-worker-nodes)
5. [Cluster Lifecycle Management](#5-cluster-lifecycle-management)
6. [Cluster Upgrades](#6-cluster-upgrades)
7. [Backup and Restore etcd](#7-backup-and-restore-etcd)
8. [Essential Commands](#8-essential-commands)
9. [Practice Exercises](#9-practice-exercises)
10. [Exam Tips](#10-exam-tips)

---

## 1. What is kubeadm?

**kubeadm** is a tool that helps you bootstrap a minimum viable Kubernetes cluster that conforms to best practices.

### kubeadm Commands Overview

| Command | Purpose |
|---------|---------|
| `kubeadm init` | Initialize control plane node |
| `kubeadm join` | Join worker/control plane nodes |
| `kubeadm upgrade` | Upgrade cluster version |
| `kubeadm reset` | Revert changes made by init/join |
| `kubeadm token` | Manage bootstrap tokens |
| `kubeadm certs` | Manage certificates |
| `kubeadm config` | Manage kubeadm configuration |

---

## 2. Prerequisites

### System Requirements

| Component | Requirement |
|-----------|-------------|
| OS | Linux (Ubuntu, CentOS, etc.) |
| RAM | 2 GB minimum |
| CPU | 2 cores minimum |
| Network | Full connectivity between nodes |
| Swap | Disabled |
| Ports | Various (see below) |

### Required Ports

**Control Plane:**
| Port | Protocol | Purpose |
|------|----------|---------|
| 6443 | TCP | Kubernetes API server |
| 2379-2380 | TCP | etcd server client API |
| 10250 | TCP | Kubelet API |
| 10259 | TCP | kube-scheduler |
| 10257 | TCP | kube-controller-manager |

**Worker Nodes:**
| Port | Protocol | Purpose |
|------|----------|---------|
| 10250 | TCP | Kubelet API |
| 30000-32767 | TCP | NodePort Services |

### Pre-installation Steps

```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Install container runtime (containerd)
sudo apt-get update
sudo apt-get install -y containerd

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

# Install kubeadm, kubelet, kubectl
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## 3. Cluster Installation

### Initialize Control Plane

```bash
# Basic initialization
sudo kubeadm init

# With specific pod network CIDR (required for some CNIs)
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# With specific API server address
sudo kubeadm init --apiserver-advertise-address=192.168.1.100

# With specific Kubernetes version
sudo kubeadm init --kubernetes-version=v1.35.0

# Full example
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.1.100 \
  --control-plane-endpoint=k8s-api.example.com:6443
```

### Post-Initialization Setup

```bash
# Configure kubectl for regular user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Or for root user
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### Install CNI (Pod Network)

```bash
# Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Flannel
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Weave
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

### Verify Installation

```bash
# Check nodes
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system

# Check component status
kubectl get componentstatuses  # Deprecated but may still work
kubectl get --raw='/readyz?verbose'
```

---

## 4. Join Worker Nodes

### Get Join Command

```bash
# On control plane, get join command
kubeadm token create --print-join-command

# Output example:
# kubeadm join 192.168.1.100:6443 --token abcdef.0123456789abcdef \
#   --discovery-token-ca-cert-hash sha256:xxx
```

### Join Worker Node

```bash
# On worker node
sudo kubeadm join 192.168.1.100:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:xxx
```

### Token Management

```bash
# List tokens
kubeadm token list

# Create new token
kubeadm token create

# Create token with specific TTL
kubeadm token create --ttl 2h

# Delete token
kubeadm token delete <token>

# Get CA cert hash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | sed 's/^.* //'
```

---

## 5. Cluster Lifecycle Management

### Drain Node (Maintenance)

```bash
# Drain node (evict pods)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Drain with grace period
kubectl drain <node-name> --ignore-daemonsets --grace-period=60

# Force drain (delete pods that don't have controller)
kubectl drain <node-name> --ignore-daemonsets --force
```

### Cordon/Uncordon Node

```bash
# Mark node as unschedulable (no new pods)
kubectl cordon <node-name>

# Mark node as schedulable again
kubectl uncordon <node-name>
```

### Remove Node from Cluster

```bash
# On control plane
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <node-name>

# On the node being removed
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube
```

---

## 6. Cluster Upgrades

### Upgrade Strategy

```
1. Upgrade control plane nodes (one at a time if HA)
2. Upgrade worker nodes (one at a time)
3. Upgrade kubelet and kubectl on all nodes
```

### Upgrade Control Plane

```bash
# Check available versions
sudo apt update
apt-cache madison kubeadm

# Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.35.0-*
sudo apt-mark hold kubeadm

# Verify upgrade plan
sudo kubeadm upgrade plan

# Apply upgrade (first control plane node)
sudo kubeadm upgrade apply v1.35.0

# For additional control plane nodes
sudo kubeadm upgrade node

# Drain the node
kubectl drain <node-name> --ignore-daemonsets

# Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get update && sudo apt-get install -y kubelet=1.35.0-* kubectl=1.35.0-*
sudo apt-mark hold kubelet kubectl

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Uncordon the node
kubectl uncordon <node-name>
```

### Upgrade Worker Nodes

```bash
# On control plane: drain worker
kubectl drain <worker-node> --ignore-daemonsets

# On worker node: upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.35.0-*
sudo apt-mark hold kubeadm

# Upgrade node config
sudo kubeadm upgrade node

# Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get update && sudo apt-get install -y kubelet=1.35.0-* kubectl=1.35.0-*
sudo apt-mark hold kubelet kubectl

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# On control plane: uncordon worker
kubectl uncordon <worker-node>
```

---

## 7. Backup and Restore etcd

### etcd Overview

etcd stores all cluster data. Backing it up is critical!

```
/etc/kubernetes/manifests/etcd.yaml  - etcd pod manifest
/etc/kubernetes/pki/etcd/            - etcd certificates
/var/lib/etcd/                       - etcd data directory
```

### Backup etcd

```bash
# Using etcdctl
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table
```

### Restore etcd

```bash
# Stop kube-apiserver (move manifest)
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Restore snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Update etcd manifest to use new data directory
sudo vi /etc/kubernetes/manifests/etcd.yaml
# Change: --data-dir=/var/lib/etcd-restored
# Change volume hostPath to /var/lib/etcd-restored

# Restart etcd (it will restart automatically when manifest changes)

# Restore kube-apiserver
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Verify
kubectl get pods -n kube-system
```

### etcdctl Environment Variables

```bash
# Set these for easier etcdctl usage
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key

# Now you can run
etcdctl snapshot save /backup/snapshot.db
etcdctl member list
```

---

## 8. Essential Commands

### kubeadm Commands

```bash
# Initialize cluster
kubeadm init --pod-network-cidr=10.244.0.0/16

# Get join command
kubeadm token create --print-join-command

# Upgrade cluster
kubeadm upgrade plan
kubeadm upgrade apply v1.35.0

# Reset node
kubeadm reset

# Certificate management
kubeadm certs check-expiration
kubeadm certs renew all
```

### Node Management

```bash
# Drain node
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# Cordon/uncordon
kubectl cordon <node>
kubectl uncordon <node>

# Delete node
kubectl delete node <node>
```

### etcd Commands

```bash
# Backup
ETCDCTL_API=3 etcdctl snapshot save /backup/snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /backup/snapshot.db \
  --data-dir=/var/lib/etcd-new

# Check members
ETCDCTL_API=3 etcdctl member list
```

---

## 9. Practice Exercises

### Exercise 1: Cluster Upgrade Simulation

**Task:** Upgrade control plane from v1.34.0 to v1.35.0 (simulation steps)

```bash
# 1. Check current version
kubectl get nodes

# 2. Check available versions
apt-cache madison kubeadm | head -5

# 3. Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.35.0-*
sudo apt-mark hold kubeadm

# 4. Plan upgrade
sudo kubeadm upgrade plan

# 5. Apply upgrade
sudo kubeadm upgrade apply v1.35.0

# 6. Drain node
kubectl drain <control-plane-node> --ignore-daemonsets

# 7. Upgrade kubelet/kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.35.0-* kubectl=1.35.0-*
sudo apt-mark hold kubelet kubectl

# 8. Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 9. Uncordon
kubectl uncordon <control-plane-node>

# 10. Verify
kubectl get nodes
```

### Exercise 2: etcd Backup and Restore

**Task:** Backup etcd and restore to a new directory

```bash
# Backup
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db --write-out=table

# Restore (to new directory)
ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored
```

### Exercise 3: Node Maintenance

**Task:** Perform maintenance on worker node `worker-1`

```bash
# 1. Cordon node (prevent new pods)
kubectl cordon worker-1

# 2. Drain node (evict existing pods)
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data

# 3. Perform maintenance...

# 4. Uncordon node
kubectl uncordon worker-1

# 5. Verify
kubectl get nodes
```

### Exercise 4: Generate New Join Token

**Task:** Create a new join token valid for 4 hours

```bash
# Create token
kubeadm token create --ttl 4h --print-join-command

# List tokens
kubeadm token list
```

---

## 10. Exam Tips

### Key Files to Know

```
/etc/kubernetes/manifests/     - Static pod manifests
/etc/kubernetes/admin.conf     - Admin kubeconfig
/etc/kubernetes/pki/           - Certificates
/var/lib/etcd/                 - etcd data
```

### Upgrade Order

```
1. kubeadm (control plane)
2. kubeadm upgrade apply
3. Drain node
4. kubelet + kubectl
5. Restart kubelet
6. Uncordon
7. Repeat for workers
```

### etcd Backup Checklist

- [ ] Know certificate paths
- [ ] Know `ETCDCTL_API=3`
- [ ] Know snapshot save/restore commands
- [ ] Know how to verify backup

### Common Mistakes

1. Forgetting `--ignore-daemonsets` when draining
2. Wrong certificate paths for etcdctl
3. Forgetting to restart kubelet after upgrade
4. Not uncordoning after maintenance

---

**Next:** [03-HA-Control-Plane.md](./03-HA-Control-Plane.md)
