# Kubernetes Services

> **CKA Domain:** Services & Networking (20%)
> **Official Docs:** https://kubernetes.io/docs/concepts/services-networking/service/

---

## Table of Contents

1. [Service Overview](#1-service-overview)
2. [Service Types](#2-service-types)
3. [ClusterIP](#3-clusterip)
4. [NodePort](#4-nodeport)
5. [LoadBalancer](#5-loadbalancer)
6. [Endpoints](#6-endpoints)
7. [Essential Commands](#7-essential-commands)
8. [Practice Exercises](#8-practice-exercises)

---

## 1. Service Overview

A **Service** is an abstraction that defines a logical set of Pods and a policy to access them.

```
┌─────────────────────────────────────────────────────────┐
│                       Service                            │
│                    (Stable IP/DNS)                       │
└─────────────────────┬───────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
   ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
   │  Pod 1  │   │  Pod 2  │   │  Pod 3  │
   │ app=web │   │ app=web │   │ app=web │
   └─────────┘   └─────────┘   └─────────┘
```

### Why Services?

- Pods are ephemeral (IPs change)
- Services provide stable networking
- Load balancing across pods
- Service discovery via DNS

---

## 2. Service Types

| Type | Description | Use Case |
|------|-------------|----------|
| **ClusterIP** | Internal cluster IP (default) | Internal communication |
| **NodePort** | Exposes on each node's IP | External access (dev/test) |
| **LoadBalancer** | Cloud provider load balancer | Production external access |
| **ExternalName** | Maps to external DNS | External service integration |

---

## 3. ClusterIP

Default type. Only accessible within the cluster.

### Create ClusterIP Service

```bash
# Imperative
kubectl expose deployment nginx --port=80 --target-port=8080

# With specific name
kubectl expose deployment nginx --name=nginx-svc --port=80
```

### YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP          # Default, can be omitted
  selector:
    app: nginx
  ports:
  - port: 80               # Service port
    targetPort: 8080       # Container port
    protocol: TCP
```

### Headless Service

No cluster IP assigned. Used for StatefulSets.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None          # Headless
  selector:
    app: nginx
  ports:
  - port: 80
```

---

## 4. NodePort

Exposes service on each node's IP at a static port.

```
External Traffic ──► NodeIP:NodePort ──► Service ──► Pods
```

### Create NodePort Service

```bash
kubectl expose deployment nginx --type=NodePort --port=80
```

### YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80               # Service port
    targetPort: 8080       # Container port
    nodePort: 30080        # Node port (30000-32767)
```

### Access

```bash
# Access via any node IP
curl http://<node-ip>:30080
```

---

## 5. LoadBalancer

Provisions external load balancer (cloud providers).

### YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 8080
```

### Check External IP

```bash
kubectl get svc nginx-lb
# EXTERNAL-IP will show the load balancer IP
```

---

## 6. Endpoints

Endpoints are automatically created for services with selectors.

```bash
# View endpoints
kubectl get endpoints
kubectl get ep nginx-service

# Describe
kubectl describe endpoints nginx-service
```

### Manual Endpoints (No Selector)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  ports:
  - port: 3306
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-db
subsets:
- addresses:
  - ip: 192.168.1.100
  ports:
  - port: 3306
```

---

## 7. Essential Commands

```bash
# Create service
kubectl expose deployment NAME --port=PORT --target-port=PORT --type=TYPE

# List services
kubectl get svc
kubectl get svc -A

# Describe
kubectl describe svc NAME

# Get endpoints
kubectl get endpoints

# Delete
kubectl delete svc NAME

# Test service (from within cluster)
kubectl run test --image=busybox --rm -it --restart=Never -- wget -qO- http://SERVICE:PORT
```

---

## 8. Practice Exercises

### Exercise 1: ClusterIP Service

```bash
# Create deployment
kubectl create deployment web --image=nginx --replicas=3

# Expose as ClusterIP
kubectl expose deployment web --port=80

# Test
kubectl run test --image=busybox --rm -it --restart=Never -- wget -qO- http://web
```

### Exercise 2: NodePort Service

```bash
# Create NodePort service
kubectl expose deployment web --type=NodePort --port=80 --name=web-nodeport

# Get NodePort
kubectl get svc web-nodeport

# Access externally
curl http://<node-ip>:<nodeport>
```

### Exercise 3: Service with Specific Ports

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-custom
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 8080
    targetPort: 80
    nodePort: 30088
EOF
```

---

**Next:** [02-Ingress.md](./02-Ingress.md)
