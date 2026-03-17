# Services & Networking - Practice Questions

> **Domain Weight:** 20% | **Estimated Questions:** 3-4

---

## Question 1: Create ClusterIP Service
**Difficulty:** Easy | **Time:** 3 minutes

Create a ClusterIP service named `backend-svc` that exposes deployment `backend` on port 80, targeting container port 8080.

<details>
<summary>💡 Solution</summary>

```bash
kubectl expose deployment backend \
  --name=backend-svc \
  --port=80 \
  --target-port=8080

# Verify
kubectl get svc backend-svc
kubectl describe svc backend-svc
```
</details>

---

## Question 2: Create NodePort Service
**Difficulty:** Easy | **Time:** 3 minutes

Create a NodePort service named `web-external` for deployment `web` with:
- Service port: 80
- Target port: 8080
- NodePort: 30080

<details>
<summary>💡 Solution</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-external
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

```bash
kubectl apply -f svc.yaml
kubectl get svc web-external
```
</details>

---

## Question 3: Network Policy - Default Deny
**Difficulty:** Medium | **Time:** 4 minutes

Create a NetworkPolicy named `deny-all` in namespace `secure` that denies all ingress and egress traffic to all pods.

<details>
<summary>💡 Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: secure
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```
</details>

---

## Question 4: Network Policy - Allow Specific Traffic
**Difficulty:** Medium | **Time:** 5 minutes

Create a NetworkPolicy named `api-policy` in namespace `app` that:
- Applies to pods with label `role=api`
- Allows ingress only from pods with label `role=frontend` on port 8080
- Allows egress only to pods with label `role=database` on port 5432

<details>
<summary>💡 Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: app
spec:
  podSelector:
    matchLabels:
      role: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: database
    ports:
    - protocol: TCP
      port: 5432
```
</details>

---

## Question 5: Ingress with Path Routing
**Difficulty:** Medium | **Time:** 5 minutes

Create an Ingress named `app-ingress` that:
- Uses ingress class `nginx`
- Routes `/api` to service `api-svc` on port 8080
- Routes `/web` to service `web-svc` on port 80

<details>
<summary>💡 Solution</summary>

```yaml
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
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 8080
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
```
</details>

---

## Question 6: Ingress with TLS
**Difficulty:** Medium | **Time:** 5 minutes

Create an Ingress named `secure-ingress` that:
- Routes host `secure.example.com` to service `secure-svc` on port 443
- Uses TLS with secret `tls-secret`
- Uses ingress class `nginx`

<details>
<summary>💡 Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.example.com
    secretName: tls-secret
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-svc
            port:
              number: 443
```
</details>

---

## Question 7: DNS Troubleshooting
**Difficulty:** Medium | **Time:** 5 minutes

A pod cannot resolve service names. Debug and identify the issue.

<details>
<summary>💡 Solution</summary>

```bash
# 1. Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 2. Check CoreDNS service
kubectl get svc -n kube-system kube-dns

# 3. Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# 4. Test DNS from a pod
kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes

# 5. Check pod's resolv.conf
kubectl exec <pod> -- cat /etc/resolv.conf

# 6. Check if NetworkPolicy is blocking DNS
kubectl get networkpolicy -A
```
</details>

---

## Question 8: Cross-Namespace Service Access
**Difficulty:** Easy | **Time:** 3 minutes

From a pod in namespace `frontend`, access service `api` in namespace `backend` on port 8080.

<details>
<summary>💡 Solution</summary>

```bash
# DNS format: <service>.<namespace>.svc.cluster.local
curl http://api.backend.svc.cluster.local:8080

# Or shorter
curl http://api.backend:8080
```
</details>

---

## Question 9: Network Policy - Allow DNS
**Difficulty:** Medium | **Time:** 5 minutes

In namespace `restricted`, there's a default-deny policy. Create a NetworkPolicy named `allow-dns` that allows all pods to reach CoreDNS for DNS resolution.

<details>
<summary>💡 Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: restricted
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```
</details>

---

## Question 10: Service Endpoints
**Difficulty:** Easy | **Time:** 3 minutes

Service `my-svc` is not routing traffic to pods. Debug the issue.

<details>
<summary>💡 Solution</summary>

```bash
# 1. Check service
kubectl describe svc my-svc

# 2. Check endpoints (if empty, selector doesn't match)
kubectl get endpoints my-svc

# 3. Check pod labels
kubectl get pods --show-labels

# 4. Compare service selector with pod labels
kubectl get svc my-svc -o yaml | grep -A5 selector

# 5. Fix: Update selector or pod labels
kubectl patch svc my-svc -p '{"spec":{"selector":{"app":"correct-label"}}}'
```
</details>

---

## Question 11: Gateway API HTTPRoute
**Difficulty:** Hard | **Time:** 6 minutes

Create an HTTPRoute named `app-route` that:
- References Gateway `my-gateway`
- Routes host `app.example.com` path `/api` to service `api-svc` port 8080
- Routes path `/` to service `web-svc` port 80

<details>
<summary>💡 Solution</summary>

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "app.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-svc
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-svc
      port: 80
```
</details>

---

## Question 12: Network Policy - Cross Namespace
**Difficulty:** Hard | **Time:** 6 minutes

Create a NetworkPolicy named `allow-monitoring` in namespace `app` that allows ingress from any pod in namespace `monitoring` on port 9090.

<details>
<summary>💡 Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
    ports:
    - protocol: TCP
      port: 9090
```
</details>

---

## Quick Reference

```bash
# Services
kubectl expose deployment NAME --port=PORT --target-port=PORT --type=TYPE
kubectl get svc
kubectl get endpoints

# Network Policies
kubectl get netpol -A
kubectl describe netpol NAME

# Ingress
kubectl get ingress
kubectl describe ingress NAME

# DNS Testing
kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- nslookup SERVICE

# CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

---

## DNS Quick Reference

```
Service DNS:
<service>.<namespace>.svc.cluster.local

Pod DNS:
<pod-ip-dashed>.<namespace>.pod.cluster.local

Same namespace: curl http://service-name
Cross namespace: curl http://service-name.namespace
```
