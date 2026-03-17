# Cluster and Node Troubleshooting

> **CKA Domain:** Troubleshooting (30%) - Highest Weight!
> **Official Docs:** https://kubernetes.io/docs/tasks/debug/debug-cluster/

---

## Table of Contents

1. [Troubleshooting Approach](#1-troubleshooting-approach)
2. [Node Issues](#2-node-issues)
3. [Control Plane Issues](#3-control-plane-issues)
4. [kubelet Issues](#4-kubelet-issues)
5. [etcd Issues](#5-etcd-issues)
6. [Certificate Issues](#6-certificate-issues)
7. [Essential Commands](#7-essential-commands)
8. [Practice Scenarios](#8-practice-scenarios)

---

## 1. Troubleshooting Approach

### Systematic Debug Flow

```
1. Identify the symptom
       │
       ▼
2. Check cluster health
       │
       ▼
3. Check node status
       │
       ▼
4. Check component logs
       │
       ▼
5. Check events
       │
       ▼
6. Fix and verify
```

### Quick Health Check

```bash
# Cluster info
kubectl cluster-info

# Node status
kubectl get nodes

# Component status
kubectl get componentstatuses   # May be deprecated
kubectl get --raw='/readyz?verbose'

# All pods in kube-system
kubectl get pods -n kube-system
```

---

## 2. Node Issues

### Node Status

| Status | Meaning |
|--------|---------|
| `Ready` | Node is healthy |
| `NotReady` | Node has issues |
| `SchedulingDisabled` | Node is cordoned |
| `Unknown` | Node lost contact |

### Debug NotReady Node

```bash
# Check node status
kubectl get nodes
kubectl describe node <node-name>

# Look for:
# - Conditions (Ready, MemoryPressure, DiskPressure, PIDPressure)
# - Events at the bottom

# SSH to node and check kubelet
systemctl status kubelet
journalctl -u kubelet -f

# Check container runtime
systemctl status containerd
crictl ps
```

### Common Node Issues

| Issue | Symptoms | Fix |
|-------|----------|-----|
| kubelet not running | Node NotReady | `systemctl start kubelet` |
| Container runtime down | Pods not starting | `systemctl start containerd` |
| Disk pressure | DiskPressure condition | Free disk space |
| Memory pressure | MemoryPressure condition | Free memory or add resources |
| Network issues | Node Unknown | Check network connectivity |

### Fix kubelet

```bash
# Check kubelet status
systemctl status kubelet

# Check kubelet logs
journalctl -u kubelet -n 100

# Restart kubelet
systemctl restart kubelet

# Enable kubelet on boot
systemctl enable kubelet
```

---

## 3. Control Plane Issues

### Control Plane Components

| Component | Location | Purpose |
|-----------|----------|---------|
| kube-apiserver | Static pod | API endpoint |
| kube-controller-manager | Static pod | Controllers |
| kube-scheduler | Static pod | Pod scheduling |
| etcd | Static pod | Data store |

### Static Pod Manifests

```bash
# Location
ls /etc/kubernetes/manifests/

# Files
# - etcd.yaml
# - kube-apiserver.yaml
# - kube-controller-manager.yaml
# - kube-scheduler.yaml
```

### Debug Control Plane

```bash
# Check static pods
kubectl get pods -n kube-system

# Check specific component
kubectl logs -n kube-system kube-apiserver-<node>
kubectl logs -n kube-system kube-controller-manager-<node>
kubectl logs -n kube-system kube-scheduler-<node>
kubectl logs -n kube-system etcd-<node>

# If kubectl doesn't work, use crictl
crictl ps
crictl logs <container-id>
```

### Common Control Plane Issues

| Issue | Symptoms | Debug |
|-------|----------|-------|
| API server down | kubectl not working | Check apiserver pod/logs |
| Scheduler down | Pods stuck Pending | Check scheduler logs |
| Controller manager down | No scaling, no endpoints | Check cm logs |
| etcd down | Cluster read-only | Check etcd logs |

### Fix Static Pod Issues

```bash
# Check manifest syntax
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Common issues:
# - Wrong image
# - Wrong volume mounts
# - Wrong certificate paths
# - Syntax errors

# After fixing manifest, kubelet auto-restarts the pod
# Watch for pod to restart
watch kubectl get pods -n kube-system
```

---

## 4. kubelet Issues

### kubelet Configuration

```bash
# kubelet config
cat /var/lib/kubelet/config.yaml

# kubelet flags
cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Check kubelet is running
systemctl status kubelet
```

### Common kubelet Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Not starting | Wrong config | Check config.yaml |
| Can't reach API | Wrong API address | Check kubeconfig |
| Certificate errors | Expired certs | Renew certificates |
| Container runtime | Runtime not running | Start containerd |

### Debug kubelet

```bash
# Check status
systemctl status kubelet

# View logs
journalctl -u kubelet -f
journalctl -u kubelet --since "10 minutes ago"

# Check kubelet process
ps aux | grep kubelet

# Restart
systemctl daemon-reload
systemctl restart kubelet
```

---

## 5. etcd Issues

### etcd Health Check

```bash
# Check etcd pod
kubectl get pods -n kube-system | grep etcd

# Check etcd logs
kubectl logs -n kube-system etcd-<node>

# etcdctl commands
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Member list
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

### Common etcd Issues

| Issue | Symptoms | Fix |
|-------|----------|-----|
| etcd not running | Cluster down | Check manifest, restart |
| Disk full | etcd crashes | Free disk space |
| Certificate expired | Connection refused | Renew certs |
| Data corruption | Cluster inconsistent | Restore from backup |

---

## 6. Certificate Issues

### Check Certificate Expiration

```bash
# Using kubeadm
kubeadm certs check-expiration

# Manual check
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
```

### Renew Certificates

```bash
# Renew all certificates
kubeadm certs renew all

# Renew specific certificate
kubeadm certs renew apiserver

# After renewal, restart control plane components
# (kubelet will auto-restart static pods)
```

### Certificate Locations

```
/etc/kubernetes/pki/
├── apiserver.crt
├── apiserver.key
├── apiserver-kubelet-client.crt
├── apiserver-kubelet-client.key
├── ca.crt
├── ca.key
├── front-proxy-ca.crt
├── front-proxy-ca.key
├── front-proxy-client.crt
├── front-proxy-client.key
├── sa.key
├── sa.pub
└── etcd/
    ├── ca.crt
    ├── ca.key
    ├── server.crt
    ├── server.key
    ├── peer.crt
    ├── peer.key
    ├── healthcheck-client.crt
    └── healthcheck-client.key
```

---

## 7. Essential Commands

```bash
# Cluster health
kubectl cluster-info
kubectl get nodes
kubectl get pods -n kube-system

# Node debug
kubectl describe node <node>
ssh <node> "systemctl status kubelet"
ssh <node> "journalctl -u kubelet"

# Control plane logs
kubectl logs -n kube-system <pod>
crictl logs <container-id>

# etcd
ETCDCTL_API=3 etcdctl endpoint health --endpoints=...

# Certificates
kubeadm certs check-expiration
kubeadm certs renew all
```

---

## 8. Practice Scenarios

### Scenario 1: Node NotReady

**Symptom:** Node shows NotReady status

```bash
# Debug steps
kubectl describe node <node>
# Look at Conditions and Events

# SSH to node
ssh <node>
systemctl status kubelet
journalctl -u kubelet | tail -50

# Common fixes
systemctl start kubelet
systemctl start containerd
```

### Scenario 2: API Server Down

**Symptom:** kubectl commands fail

```bash
# Check if API server pod is running
crictl ps | grep apiserver

# Check logs
crictl logs <apiserver-container-id>

# Check manifest
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Common issues:
# - Wrong certificate paths
# - Wrong etcd endpoints
# - Port conflicts
```

### Scenario 3: Scheduler Not Working

**Symptom:** Pods stuck in Pending

```bash
# Check scheduler
kubectl get pods -n kube-system | grep scheduler
kubectl logs -n kube-system kube-scheduler-<node>

# Check pod events
kubectl describe pod <pending-pod>

# Common issues:
# - Scheduler not running
# - No nodes available
# - Resource constraints
```

### Scenario 4: etcd Issues

**Symptom:** Cluster read-only or down

```bash
# Check etcd
kubectl get pods -n kube-system | grep etcd
kubectl logs -n kube-system etcd-<node>

# Check etcd health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

**Next:** [02-Application-Troubleshooting.md](./02-Application-Troubleshooting.md)
