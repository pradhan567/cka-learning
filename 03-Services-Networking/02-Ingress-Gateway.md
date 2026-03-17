# Ingress and Gateway API

> **CKA Domain:** Services & Networking (20%)
> **Official Docs:** 
> - https://kubernetes.io/docs/concepts/services-networking/ingress/
> - https://kubernetes.io/docs/concepts/services-networking/gateway/

---

## Table of Contents

1. [Ingress Overview](#1-ingress-overview)
2. [Ingress Controllers](#2-ingress-controllers)
3. [Ingress Resources](#3-ingress-resources)
4. [Path-Based Routing](#4-path-based-routing)
5. [Host-Based Routing](#5-host-based-routing)
6. [TLS/HTTPS](#6-tlshttps)
7. [Gateway API](#7-gateway-api)
8. [Essential Commands](#8-essential-commands)
9. [Practice Exercises](#9-practice-exercises)

---

## 1. Ingress Overview

**Ingress** exposes HTTP/HTTPS routes from outside the cluster to services within.

```
Internet ──► Ingress Controller ──► Ingress Rules ──► Services ──► Pods
                    │
                    ├── /api  ──► api-service
                    ├── /web  ──► web-service
                    └── /blog ──► blog-service
```

### Ingress vs Service

| Feature | Service (NodePort/LB) | Ingress |
|---------|----------------------|---------|
| Protocol | TCP/UDP | HTTP/HTTPS |
| Routing | Port-based | Path/Host-based |
| TLS | Manual | Built-in |
| Cost | Multiple LBs | Single entry point |

---

## 2. Ingress Controllers

Ingress resources require an **Ingress Controller** to function.

### Popular Controllers

| Controller | Provider |
|------------|----------|
| **ingress-nginx** | Kubernetes community |
| **Traefik** | Traefik Labs |
| **HAProxy** | HAProxy Technologies |
| **AWS ALB** | Amazon |
| **GCE** | Google Cloud |

### Install ingress-nginx

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.0/deploy/static/provider/cloud/deploy.yaml

# Verify
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

---

## 3. Ingress Resources

### Basic Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Path Types

| Type | Description |
|------|-------------|
| `Prefix` | Matches URL path prefix |
| `Exact` | Exact URL path match |
| `ImplementationSpecific` | Controller-dependent |

---

## 4. Path-Based Routing

Route different paths to different services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

---

## 5. Host-Based Routing

Route different hostnames to different services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

---

## 6. TLS/HTTPS

### Create TLS Secret

```bash
kubectl create secret tls my-tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

### Ingress with TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.example.com
    secretName: my-tls-secret
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-service
            port:
              number: 443
```

---

## 7. Gateway API

**Gateway API** is the successor to Ingress with more features and flexibility.

### Key Resources

| Resource | Description |
|----------|-------------|
| **GatewayClass** | Defines the controller (like IngressClass) |
| **Gateway** | Defines listeners (ports, protocols, TLS) |
| **HTTPRoute** | Defines HTTP routing rules |

### GatewayClass

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: my-gateway-class
spec:
  controllerName: example.com/gateway-controller
```

### Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: my-gateway-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
```

### HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-service
      port: 80
```

### Gateway API vs Ingress

| Feature | Ingress | Gateway API |
|---------|---------|-------------|
| Role separation | No | Yes (infra vs app teams) |
| Protocol support | HTTP/HTTPS | HTTP, HTTPS, TCP, UDP, gRPC |
| Traffic splitting | Limited | Native support |
| Header matching | Annotation-based | Native support |

---

## 8. Essential Commands

```bash
# Ingress
kubectl get ingress
kubectl describe ingress NAME
kubectl get ingressclass

# Gateway API
kubectl get gateways
kubectl get httproutes
kubectl get gatewayclasses

# Debug
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

---

## 9. Practice Exercises

### Exercise 1: Basic Ingress

```bash
# Create deployments and services
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80

kubectl create deployment api --image=nginx
kubectl expose deployment api --port=80

# Create Ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 80
EOF

# Verify
kubectl get ingress
kubectl describe ingress app-ingress
```

### Exercise 2: Host-Based Ingress

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: web.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
  - host: api.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 80
EOF
```

---

**Next:** [03-CoreDNS.md](./03-CoreDNS.md)
