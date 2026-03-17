# CoreDNS

> **CKA Domain:** Services & Networking (20%)
> **Official Docs:** https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/

---

## Table of Contents

1. [DNS in Kubernetes](#1-dns-in-kubernetes)
2. [CoreDNS Overview](#2-coredns-overview)
3. [Service DNS Records](#3-service-dns-records)
4. [Pod DNS Records](#4-pod-dns-records)
5. [DNS Policies](#5-dns-policies)
6. [Troubleshooting DNS](#6-troubleshooting-dns)
7. [Essential Commands](#7-essential-commands)
8. [Practice Exercises](#8-practice-exercises)

---

## 1. DNS in Kubernetes

Kubernetes creates DNS records for Services and Pods automatically.

```
┌─────────────────────────────────────────────────────────┐
│                      CoreDNS                             │
│              (kube-system namespace)                     │
└─────────────────────┬───────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
   Service DNS    Pod DNS      External DNS
   my-svc         10-0-0-1     google.com
   my-svc.ns      pod.ns       
   my-svc.ns.svc.cluster.local
```

---

## 2. CoreDNS Overview

### CoreDNS Components

```bash
# CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# CoreDNS service
kubectl get svc -n kube-system kube-dns

# CoreDNS ConfigMap
kubectl get configmap -n kube-system coredns
```

### CoreDNS ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

---

## 3. Service DNS Records

### DNS Format

```
<service-name>.<namespace>.svc.cluster.local
```

### Examples

| Service | Namespace | Full DNS Name |
|---------|-----------|---------------|
| `nginx` | `default` | `nginx.default.svc.cluster.local` |
| `api` | `production` | `api.production.svc.cluster.local` |
| `db` | `database` | `db.database.svc.cluster.local` |

### Short Names (Within Same Namespace)

```bash
# From pod in same namespace
curl http://nginx           # Works
curl http://nginx.default   # Works
curl http://nginx.default.svc.cluster.local  # Full name
```

### Cross-Namespace Access

```bash
# From pod in different namespace
curl http://nginx.default   # Need at least namespace
curl http://nginx.default.svc.cluster.local
```

### Headless Service DNS

For headless services (`clusterIP: None`), DNS returns pod IPs directly.

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

---

## 4. Pod DNS Records

### Pod DNS Format

```
<pod-ip-with-dashes>.<namespace>.pod.cluster.local
```

### Example

Pod IP: `10.244.1.5` in namespace `default`:
```
10-244-1-5.default.pod.cluster.local
```

### Pod DNS Policy

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  dnsPolicy: ClusterFirst    # Default
  containers:
  - name: app
    image: nginx
```

---

## 5. DNS Policies

| Policy | Description |
|--------|-------------|
| `ClusterFirst` | Use cluster DNS first, then upstream (default) |
| `Default` | Inherit from node |
| `ClusterFirstWithHostNet` | For hostNetwork pods |
| `None` | No DNS config, must specify dnsConfig |

### Custom DNS Config

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  dnsPolicy: None
  dnsConfig:
    nameservers:
      - 8.8.8.8
      - 8.8.4.4
    searches:
      - my-namespace.svc.cluster.local
      - svc.cluster.local
    options:
      - name: ndots
        value: "5"
  containers:
  - name: app
    image: nginx
```

---

## 6. Troubleshooting DNS

### Test DNS Resolution

```bash
# Run a debug pod
kubectl run dnstest --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes

# Test specific service
kubectl run dnstest --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx.default

# Check /etc/resolv.conf in pod
kubectl exec <pod> -- cat /etc/resolv.conf
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| DNS not resolving | CoreDNS not running | Check CoreDNS pods |
| Timeout | Network policy blocking | Check NetworkPolicy |
| Wrong namespace | Using short name | Use FQDN |
| NXDOMAIN | Service doesn't exist | Verify service exists |

### Debug CoreDNS

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Check CoreDNS service
kubectl get svc -n kube-system kube-dns

# Check endpoints
kubectl get endpoints -n kube-system kube-dns
```

---

## 7. Essential Commands

```bash
# DNS testing
kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- nslookup <service>

# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# View CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml

# Check pod DNS config
kubectl exec <pod> -- cat /etc/resolv.conf
```

---

## 8. Practice Exercises

### Exercise 1: Test Service DNS

```bash
# Create service
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80

# Test DNS
kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx

# Test with full name
kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx.default.svc.cluster.local
```

### Exercise 2: Cross-Namespace DNS

```bash
# Create service in different namespace
kubectl create namespace other
kubectl create deployment web --image=nginx -n other
kubectl expose deployment web --port=80 -n other

# Test from default namespace
kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- nslookup web.other.svc.cluster.local
```

### Exercise 3: Debug DNS Issues

```bash
# Check CoreDNS is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Verify DNS service
kubectl get svc kube-dns -n kube-system
```

---

**Next:** [Practice-Questions.md](./Practice-Questions.md)
