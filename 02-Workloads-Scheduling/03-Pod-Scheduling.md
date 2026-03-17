# Pod Scheduling

> **CKA Domain:** Workloads & Scheduling (15%)
> **Official Docs:** 
> - https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
> - https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

---

## Table of Contents

1. [Scheduling Overview](#1-scheduling-overview)
2. [Node Selectors](#2-node-selectors)
3. [Node Affinity](#3-node-affinity)
4. [Taints and Tolerations](#4-taints-and-tolerations)
5. [Pod Affinity/Anti-Affinity](#5-pod-affinityanti-affinity)
6. [Resource Requests and Limits](#6-resource-requests-and-limits)
7. [Priority and Preemption](#7-priority-and-preemption)
8. [Essential Commands](#8-essential-commands)
9. [Practice Exercises](#9-practice-exercises)

---

## 1. Scheduling Overview

The **kube-scheduler** assigns pods to nodes based on:
- Resource requirements
- Node selectors and affinity
- Taints and tolerations
- Pod affinity/anti-affinity

```
Pod Created ──► Scheduler ──► Filtering ──► Scoring ──► Binding
                    │              │            │           │
                    ▼              ▼            ▼           ▼
               Watches for    Finds nodes   Ranks      Assigns pod
               unscheduled    that meet     suitable   to best node
               pods           requirements  nodes
```

---

## 2. Node Selectors

Simple way to constrain pods to nodes with specific labels.

### Label Nodes

```bash
kubectl label nodes node1 disktype=ssd
kubectl label nodes node2 disktype=hdd
```

### Use nodeSelector

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: nginx
    image: nginx
```

---

## 3. Node Affinity

More expressive than nodeSelector with required/preferred rules.

### Required (Hard) Affinity

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
            - nvme
  containers:
  - name: nginx
    image: nginx
```

### Preferred (Soft) Affinity

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      - weight: 20
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-east-1a
  containers:
  - name: nginx
    image: nginx
```

### Operators

| Operator | Description |
|----------|-------------|
| `In` | Value is in the list |
| `NotIn` | Value is not in the list |
| `Exists` | Key exists (any value) |
| `DoesNotExist` | Key does not exist |
| `Gt` | Greater than (numeric) |
| `Lt` | Less than (numeric) |

---

## 4. Taints and Tolerations

**Taints** repel pods from nodes. **Tolerations** allow pods to schedule on tainted nodes.

### Taint a Node

```bash
# Add taint
kubectl taint nodes node1 key=value:NoSchedule
kubectl taint nodes node1 dedicated=gpu:NoExecute

# Remove taint
kubectl taint nodes node1 key=value:NoSchedule-
```

### Taint Effects

| Effect | Description |
|--------|-------------|
| `NoSchedule` | New pods won't schedule (existing stay) |
| `PreferNoSchedule` | Soft version - avoid if possible |
| `NoExecute` | Evict existing pods without toleration |

### Toleration in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  # Or tolerate all taints with a key
  - key: "dedicated"
    operator: "Exists"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx
```

### Tolerate All Taints

```yaml
tolerations:
- operator: "Exists"  # Tolerates everything
```

### Common Use Cases

```bash
# GPU nodes
kubectl taint nodes gpu-node1 dedicated=gpu:NoSchedule

# Maintenance
kubectl taint nodes node1 maintenance=true:NoExecute

# Control plane (already tainted)
# node-role.kubernetes.io/control-plane:NoSchedule
```

---

## 5. Pod Affinity/Anti-Affinity

Schedule pods based on labels of **other pods** already running.

### Pod Affinity (Co-locate)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: cache
        topologyKey: kubernetes.io/hostname
  containers:
  - name: web
    image: nginx
```

### Pod Anti-Affinity (Spread)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: web
        topologyKey: kubernetes.io/hostname
  containers:
  - name: web
    image: nginx
```

### topologyKey

| Key | Scope |
|-----|-------|
| `kubernetes.io/hostname` | Same node |
| `topology.kubernetes.io/zone` | Same zone |
| `topology.kubernetes.io/region` | Same region |

---

## 6. Resource Requests and Limits

### Requests vs Limits

| Type | Description |
|------|-------------|
| **Requests** | Guaranteed resources (used for scheduling) |
| **Limits** | Maximum resources (enforced at runtime) |

### Pod with Resources

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### CPU Units

| Value | Meaning |
|-------|---------|
| `1` | 1 CPU core |
| `500m` | 0.5 CPU core (500 millicores) |
| `100m` | 0.1 CPU core |

### LimitRange

Set default/min/max for namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "64Mi"
    max:
      cpu: "2"
      memory: "1Gi"
    min:
      cpu: "50m"
      memory: "32Mi"
    type: Container
```

### ResourceQuota

Limit total resources in namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
```

---

## 7. Priority and Preemption

### PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "High priority for critical workloads"
```

### Use PriorityClass

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx
```

---

## 8. Essential Commands

```bash
# Node labels
kubectl label nodes NODE KEY=VALUE
kubectl label nodes NODE KEY-              # Remove label
kubectl get nodes --show-labels

# Taints
kubectl taint nodes NODE KEY=VALUE:EFFECT
kubectl taint nodes NODE KEY=VALUE:EFFECT-  # Remove
kubectl describe node NODE | grep Taint

# View scheduling
kubectl get pods -o wide                   # Shows node assignment
kubectl describe pod POD | grep -A5 Events # Scheduling events

# Resources
kubectl top nodes
kubectl top pods
kubectl describe node NODE | grep -A5 "Allocated resources"
```

---

## 9. Practice Exercises

### Exercise 1: Node Selector

```bash
# Label node
kubectl label nodes <node-name> env=production

# Create pod with nodeSelector
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: prod-pod
spec:
  nodeSelector:
    env: production
  containers:
  - name: nginx
    image: nginx
EOF

# Verify
kubectl get pod prod-pod -o wide
```

### Exercise 2: Taints and Tolerations

```bash
# Taint node
kubectl taint nodes <node-name> dedicated=special:NoSchedule

# Create pod without toleration (won't schedule)
kubectl run test-pod --image=nginx

# Create pod with toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: tolerant-pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "special"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx
EOF

# Cleanup
kubectl taint nodes <node-name> dedicated=special:NoSchedule-
```

### Exercise 3: Node Affinity

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
  containers:
  - name: nginx
    image: nginx
EOF
```

---

**Next:** [Practice-Questions.md](./Practice-Questions.md)
