# Deployments and Rolling Updates

> **CKA Domain:** Workloads & Scheduling (15%)
> **Official Docs:** https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

---

## Table of Contents

1. [Deployment Overview](#1-deployment-overview)
2. [Creating Deployments](#2-creating-deployments)
3. [Rolling Updates](#3-rolling-updates)
4. [Rollbacks](#4-rollbacks)
5. [Scaling](#5-scaling)
6. [Deployment Strategies](#6-deployment-strategies)
7. [Essential Commands](#7-essential-commands)
8. [Practice Exercises](#8-practice-exercises)

---

## 1. Deployment Overview

A **Deployment** provides declarative updates for Pods and ReplicaSets.

```
Deployment
    │
    ├── ReplicaSet (current)
    │       ├── Pod
    │       ├── Pod
    │       └── Pod
    │
    └── ReplicaSet (previous - for rollback)
            ├── Pod (scaled to 0)
            └── ...
```

### Key Features

- Declarative updates
- Rolling updates and rollbacks
- Scaling
- Pause and resume
- Revision history

---

## 2. Creating Deployments

### Imperative Command

```bash
# Create deployment
kubectl create deployment nginx --image=nginx:1.21

# Create with replicas
kubectl create deployment nginx --image=nginx:1.21 --replicas=3

# Generate YAML
kubectl create deployment nginx --image=nginx:1.21 --dry-run=client -o yaml > deploy.yaml
```

### Declarative YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

### Deployment with Strategy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Max pods above desired during update
      maxUnavailable: 0    # Max pods unavailable during update
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
```

---

## 3. Rolling Updates

### Update Image

```bash
# Update image
kubectl set image deployment/nginx-deployment nginx=nginx:1.22

# Update using edit
kubectl edit deployment nginx-deployment

# Update using patch
kubectl patch deployment nginx-deployment -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.22"}]}}}}'
```

### Monitor Rollout

```bash
# Watch rollout status
kubectl rollout status deployment/nginx-deployment

# Check rollout history
kubectl rollout history deployment/nginx-deployment

# Check specific revision
kubectl rollout history deployment/nginx-deployment --revision=2
```

### Rolling Update Process

```
Initial State (v1):     During Update:           Final State (v2):
┌─────┐ ┌─────┐ ┌─────┐  ┌─────┐ ┌─────┐ ┌─────┐  ┌─────┐ ┌─────┐ ┌─────┐
│ v1  │ │ v1  │ │ v1  │  │ v1  │ │ v1  │ │ v2  │  │ v2  │ │ v2  │ │ v2  │
└─────┘ └─────┘ └─────┘  └─────┘ └─────┘ └─────┘  └─────┘ └─────┘ └─────┘
                              ▲
                         New pod created,
                         old pod terminated
```

---

## 4. Rollbacks

### Rollback to Previous Version

```bash
# Rollback to previous revision
kubectl rollout undo deployment/nginx-deployment

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

### View Revision History

```bash
# List all revisions
kubectl rollout history deployment/nginx-deployment

# View specific revision details
kubectl rollout history deployment/nginx-deployment --revision=3
```

### Record Changes (for better history)

```bash
# Record the command in revision history
kubectl set image deployment/nginx-deployment nginx=nginx:1.22 --record

# Or use annotation
kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="Updated to nginx 1.22"
```

---

## 5. Scaling

### Manual Scaling

```bash
# Scale deployment
kubectl scale deployment nginx-deployment --replicas=5

# Scale multiple deployments
kubectl scale deployment nginx-deployment web-deployment --replicas=3
```

### Autoscaling (HPA)

```bash
# Create HPA
kubectl autoscale deployment nginx-deployment --min=2 --max=10 --cpu-percent=80

# View HPA
kubectl get hpa
```

### HPA YAML

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## 6. Deployment Strategies

### RollingUpdate (Default)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # or absolute number
      maxUnavailable: 25%  # or absolute number
```

| Parameter | Description |
|-----------|-------------|
| `maxSurge` | Max pods above desired count during update |
| `maxUnavailable` | Max pods that can be unavailable during update |

### Recreate

All existing pods killed before new ones created.

```yaml
spec:
  strategy:
    type: Recreate
```

**Use case:** When you can't run multiple versions simultaneously (e.g., database migrations)

### Strategy Comparison

| Strategy | Downtime | Resource Usage | Use Case |
|----------|----------|----------------|----------|
| RollingUpdate | No | Higher during update | Most applications |
| Recreate | Yes | Same | Breaking changes |

---

## 7. Essential Commands

```bash
# Create
kubectl create deployment NAME --image=IMAGE --replicas=N

# Update image
kubectl set image deployment/NAME CONTAINER=IMAGE

# Scale
kubectl scale deployment/NAME --replicas=N

# Rollout status
kubectl rollout status deployment/NAME

# Rollout history
kubectl rollout history deployment/NAME

# Rollback
kubectl rollout undo deployment/NAME
kubectl rollout undo deployment/NAME --to-revision=N

# Pause/Resume
kubectl rollout pause deployment/NAME
kubectl rollout resume deployment/NAME

# Restart (trigger new rollout)
kubectl rollout restart deployment/NAME
```

---

## 8. Practice Exercises

### Exercise 1: Create and Update Deployment

```bash
# Create deployment
kubectl create deployment web --image=nginx:1.20 --replicas=3

# Update image
kubectl set image deployment/web nginx=nginx:1.21

# Watch rollout
kubectl rollout status deployment/web

# Check history
kubectl rollout history deployment/web
```

### Exercise 2: Rollback

```bash
# Update to bad image
kubectl set image deployment/web nginx=nginx:invalid

# Check status (will show issues)
kubectl rollout status deployment/web

# Rollback
kubectl rollout undo deployment/web

# Verify
kubectl get pods
```

### Exercise 3: Scaling

```bash
# Scale up
kubectl scale deployment/web --replicas=5

# Create HPA
kubectl autoscale deployment/web --min=2 --max=10 --cpu-percent=50

# Check HPA
kubectl get hpa
```

---

**Next:** [02-ConfigMaps-Secrets.md](./02-ConfigMaps-Secrets.md)
