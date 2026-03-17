# High Availability Control Plane

> **CKA Domain:** Cluster Architecture, Installation & Configuration (25%)
> **Official Docs:** https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/

---

## Table of Contents

1. [HA Concepts](#1-ha-concepts)
2. [HA Topologies](#2-ha-topologies)
3. [Setting Up HA Cluster](#3-setting-up-ha-cluster)
4. [Load Balancer Configuration](#4-load-balancer-configuration)
5. [Adding Control Plane Nodes](#5-adding-control-plane-nodes)
6. [Practice Exercises](#6-practice-exercises)

---

## 1. HA Concepts

### Why High Availability?

Single control plane = Single Point of Failure (SPOF)

```
Single Control Plane:           HA Control Plane:
┌─────────────────┐            ┌─────────────────┐
│  Control Plane  │            │   Load Balancer │
│    (SPOF!)      │            └────────┬────────┘
└────────┬────────┘                     │
         │                    ┌─────────┼─────────┐
    ┌────┴────┐               │         │         │
    │         │            ┌──┴──┐   ┌──┴──┐   ┌──┴──┐
 Worker    Worker          │ CP1 │   │ CP2 │   │ CP3 │
                           └─────┘   └─────┘   └─────┘
                              │         │         │
                           ┌──┴─────────┴─────────┴──┐
                           │      Worker Nodes       │
                           └─────────────────────────┘
```

### HA Components

| Component | HA Strategy |
|-----------|-------------|
| **kube-apiserver** | Multiple instances behind load balancer |
| **etcd** | Clustered (3 or 5 nodes for quorum) |
| **kube-controller-manager** | Leader election (only one active) |
| **kube-scheduler** | Leader election (only one active) |

---

## 2. HA Topologies

### Stacked etcd Topology

etcd runs on the same nodes as control plane components.

```
┌─────────────────────────────────────────────────────┐
│                   Load Balancer                      │
└─────────────────────┬───────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
   ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
   │  Node 1 │   │  Node 2 │   │  Node 3 │
   │─────────│   │─────────│   │─────────│
   │ API     │   │ API     │   │ API     │
   │ Sched   │   │ Sched   │   │ Sched   │
   │ CM      │   │ CM      │   │ CM      │
   │ etcd ◄──┼───┼─► etcd ◄┼───┼─► etcd  │
   └─────────┘   └─────────┘   └─────────┘
```

**Pros:**
- Simpler setup
- Fewer nodes required

**Cons:**
- Coupled failure (losing a node loses both control plane and etcd member)

### External etcd Topology

etcd runs on separate dedicated nodes.

```
┌─────────────────────────────────────────────────────┐
│                   Load Balancer                      │
└─────────────────────┬───────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
   ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
   │  CP 1   │   │  CP 2   │   │  CP 3   │
   │─────────│   │─────────│   │─────────│
   │ API     │   │ API     │   │ API     │
   │ Sched   │   │ Sched   │   │ Sched   │
   │ CM      │   │ CM      │   │ CM      │
   └────┬────┘   └────┬────┘   └────┬────┘
        │             │             │
        └─────────────┼─────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
   ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
   │ etcd 1  │◄──┼─► etcd 2│◄──┼─► etcd 3│
   └─────────┘   └─────────┘   └─────────┘
```

**Pros:**
- Decoupled failure domains
- Can scale etcd independently

**Cons:**
- More complex setup
- More nodes required

---

## 3. Setting Up HA Cluster

### Prerequisites

- 3+ control plane nodes
- Load balancer (HAProxy, nginx, cloud LB)
- Shared endpoint for API server

### Initialize First Control Plane

```bash
sudo kubeadm init \
  --control-plane-endpoint "LOAD_BALANCER_DNS:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16
```

**Key flags:**
- `--control-plane-endpoint`: Load balancer address (REQUIRED for HA)
- `--upload-certs`: Upload certificates to kubeadm-certs secret

### Save Join Commands

After init, you'll see two join commands:

```bash
# For control plane nodes:
kubeadm join LOAD_BALANCER:6443 --token xxx \
  --discovery-token-ca-cert-hash sha256:xxx \
  --control-plane --certificate-key xxx

# For worker nodes:
kubeadm join LOAD_BALANCER:6443 --token xxx \
  --discovery-token-ca-cert-hash sha256:xxx
```

---

## 4. Load Balancer Configuration

### HAProxy Example

```bash
# /etc/haproxy/haproxy.cfg

frontend kubernetes-frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server cp1 192.168.1.101:6443 check fall 3 rise 2
    server cp2 192.168.1.102:6443 check fall 3 rise 2
    server cp3 192.168.1.103:6443 check fall 3 rise 2
```

### Nginx Example

```nginx
# /etc/nginx/nginx.conf

stream {
    upstream kubernetes {
        server 192.168.1.101:6443;
        server 192.168.1.102:6443;
        server 192.168.1.103:6443;
    }

    server {
        listen 6443;
        proxy_pass kubernetes;
    }
}
```

---

## 5. Adding Control Plane Nodes

### Join Additional Control Plane

```bash
# On additional control plane nodes
sudo kubeadm join LOAD_BALANCER:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key>
```

### If Certificate Key Expired

```bash
# On first control plane, regenerate certificate key
sudo kubeadm init phase upload-certs --upload-certs

# Use new certificate key in join command
```

### Verify HA Setup

```bash
# Check all control plane nodes
kubectl get nodes

# Check etcd cluster health
kubectl -n kube-system exec -it etcd-<node> -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list

# Check component status
kubectl get pods -n kube-system | grep -E 'etcd|apiserver|scheduler|controller'
```

---

## 6. Practice Exercises

### Exercise 1: Understand HA Topology

**Question:** In a stacked etcd topology with 3 control plane nodes, what happens if 2 nodes fail?

**Answer:** 
- etcd loses quorum (needs majority: 2 of 3)
- Cluster becomes read-only
- No new pods can be scheduled
- Existing workloads continue running

### Exercise 2: Calculate etcd Quorum

| Cluster Size | Quorum | Fault Tolerance |
|--------------|--------|-----------------|
| 1 | 1 | 0 |
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

**Formula:** Quorum = (n/2) + 1

---

**Next:** [04-Helm-Kustomize.md](./04-Helm-Kustomize.md)
