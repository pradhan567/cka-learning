# CKA Network Policies Guide

> **CKA Exam Weight:** Part of Services & Networking (20%) | **Difficulty:** Medium-High

---

## Table of Contents

1. [What is a Network Policy?](#1-what-is-a-network-policy)
2. [Prerequisites](#2-prerequisites)
3. [Default Behavior](#3-default-behavior)
4. [NetworkPolicy Structure](#4-networkpolicy-structure)
5. [The 4 Selector Types](#5-the-4-selector-types)
6. [YAML Syntax Traps](#6-yaml-syntax-traps)
7. [Default Policy Templates](#7-default-policy-templates)
8. [Port Ranges](#8-port-ranges)
9. [Targeting Namespaces](#9-targeting-namespaces)
10. [Real-World Examples](#10-real-world-examples)
11. [Essential Commands](#11-essential-commands)
12. [Practice Exercises](#12-practice-exercises)
13. [Troubleshooting](#13-troubleshooting)
14. [Limitations](#14-limitations)
15. [Exam Tips](#15-exam-tips)

---

## 1. What is a Network Policy?

A **NetworkPolicy** is a Kubernetes resource that controls traffic flow at the IP address or port level (OSI Layer 3/4). Think of it as a **firewall for your Pods**.

### Visual Representation

```
WITHOUT Network Policy:          WITH Network Policy:
┌─────────┐    ┌─────────┐       ┌─────────┐    ┌─────────┐
│ Pod A   │◄──►│ Pod B   │       │ Pod A   │──X─│ Pod B   │  (blocked)
└─────────┘    └─────────┘       └─────────┘    └─────────┘
     ▲              ▲                 ▲              
     │              │                 │              
     ▼              ▼                 ▼              
┌─────────┐    ┌─────────┐       ┌─────────┐    ┌─────────┐
│ Pod C   │◄──►│ Pod D   │       │ Pod C   │◄──►│ Pod D   │  (allowed)
└─────────┘    └─────────┘       └─────────┘    └─────────┘

All pods can talk to all pods    Only explicitly allowed traffic flows
```

### Key Points

- NetworkPolicies are **namespaced** resources
- They use **labels** to select pods
- Policies are **additive** (they combine, not conflict)
- Both **ingress AND egress** must allow for connection to succeed

---

## 2. Prerequisites

### CNI Plugin Support

NetworkPolicies require a CNI (Container Network Interface) plugin that supports them:

| CNI Plugin | NetworkPolicy Support |
|------------|----------------------|
| **Calico** | ✅ Full support |
| **Cilium** | ✅ Full support |
| **Weave Net** | ✅ Full support |
| **Antrea** | ✅ Full support |
| **Flannel** | ❌ No support |
| **Kubenet** | ❌ No support |

⚠️ **Critical:** Creating a NetworkPolicy without a supporting CNI has **NO EFFECT**!

### Check Your CNI

```bash
# Check which CNI is installed
kubectl get pods -n kube-system | grep -E 'calico|cilium|weave|flannel'

# Check if NetworkPolicy controller is running
kubectl get pods -n kube-system -l k8s-app=calico-kube-controllers
```

---

## 3. Default Behavior

### Without Any NetworkPolicy

| Direction | Default Behavior |
|-----------|------------------|
| Ingress | **All allowed** - Any pod can receive traffic from anywhere |
| Egress | **All allowed** - Any pod can send traffic to anywhere |

### When NetworkPolicy Selects a Pod

Once a NetworkPolicy selects a pod for a direction:

1. The pod becomes **isolated** for that direction
2. **Only traffic explicitly allowed** by NetworkPolicy rules can flow
3. All other traffic is **denied by default**

### The Two Types of Isolation

| Type | Meaning |
|------|---------|
| **Isolated for Ingress** | Pod selected by a policy with `Ingress` in policyTypes |
| **Isolated for Egress** | Pod selected by a policy with `Egress` in policyTypes |

### Connection Requirements

For a connection to succeed:
```
Source Pod ──► [Egress Policy Must Allow] ──► [Ingress Policy Must Allow] ──► Destination Pod
```

Both sides must permit the connection!

---

## 4. NetworkPolicy Structure

### Complete YAML Template

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-policy
  namespace: default              # NetworkPolicies are namespaced
spec:
  podSelector:                    # WHO does this policy apply to?
    matchLabels:
      app: database
  
  policyTypes:                    # WHAT directions are controlled?
    - Ingress
    - Egress
  
  ingress:                        # ALLOW incoming traffic FROM...
    - from:
        - podSelector:
            matchLabels:
              app: backend
        - namespaceSelector:
            matchLabels:
              env: production
        - ipBlock:
            cidr: 10.0.0.0/8
            except:
              - 10.0.1.0/24
      ports:
        - protocol: TCP
          port: 5432
  
  egress:                         # ALLOW outgoing traffic TO...
    - to:
        - podSelector:
            matchLabels:
              app: cache
      ports:
        - protocol: TCP
          port: 6379
```

### Spec Fields Explained

| Field | Description |
|-------|-------------|
| `podSelector` | Selects pods this policy applies to. Empty `{}` = all pods in namespace |
| `policyTypes` | List containing `Ingress`, `Egress`, or both |
| `ingress` | List of ingress rules (what incoming traffic to allow) |
| `egress` | List of egress rules (what outgoing traffic to allow) |

### policyTypes Behavior

| policyTypes | Effect |
|-------------|--------|
| Not specified | `Ingress` always set; `Egress` set only if egress rules exist |
| `[Ingress]` | Pod isolated for ingress only |
| `[Egress]` | Pod isolated for egress only |
| `[Ingress, Egress]` | Pod isolated for both directions |

---

## 5. The 4 Selector Types

### 1. podSelector

Selects pods **in the same namespace** as the NetworkPolicy.

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            role: frontend
```

**Meaning:** Allow traffic from pods with label `role=frontend` in the same namespace.

### 2. namespaceSelector

Selects **all pods** in namespaces matching the selector.

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            env: production
```

**Meaning:** Allow traffic from ANY pod in namespaces labeled `env=production`.

### 3. podSelector + namespaceSelector (Combined)

Selects **specific pods** in **specific namespaces**.

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            env: production
        podSelector:
          matchLabels:
            role: frontend
```

**Meaning:** Allow traffic from pods with `role=frontend` that are in namespaces with `env=production`.

### 4. ipBlock

Selects traffic by **IP CIDR range** (typically for external traffic).

```yaml
ingress:
  - from:
      - ipBlock:
          cidr: 172.17.0.0/16
          except:
            - 172.17.1.0/24
```

**Meaning:** Allow traffic from IPs in 172.17.0.0/16, except 172.17.1.0/24.

### Selector Summary Table

| Selector | Scope | Use Case |
|----------|-------|----------|
| `podSelector` | Same namespace | Allow specific pods |
| `namespaceSelector` | Cross-namespace | Allow entire namespaces |
| `podSelector` + `namespaceSelector` | Cross-namespace | Allow specific pods in specific namespaces |
| `ipBlock` | External IPs | Allow external traffic |

---

## 6. YAML Syntax Traps

### ⚠️ CRITICAL FOR CKA EXAM

The difference between AND and OR logic depends on YAML structure!

### AND Logic (Single Array Element)

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            user: alice
        podSelector:              # SAME array element = AND
          matchLabels:
            role: client
```

**Meaning:** Pods with `role=client` **AND** in namespaces with `user=alice`.

### OR Logic (Multiple Array Elements)

```yaml
ingress:
  - from:
      - namespaceSelector:        # First element
          matchLabels:
            user: alice
      - podSelector:              # Second element (note the dash!)
          matchLabels:
            role: client
```

**Meaning:** ANY pod in namespaces with `user=alice` **OR** pods with `role=client` in the same namespace.

### Visual Comparison

```
AND (single element):           OR (multiple elements):
- namespaceSelector:            - namespaceSelector:
    matchLabels:                    matchLabels:
      user: alice                     user: alice
  podSelector:          ←──     - podSelector:          ←── Note the dash!
    matchLabels:                    matchLabels:
      role: client                    role: client
```

### Debug Tip

```bash
# Always verify how Kubernetes interpreted your policy
kubectl describe networkpolicy <policy-name>
```

---

## 7. Default Policy Templates

### Deny All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}              # Empty = ALL pods in namespace
  policyTypes:
    - Ingress                  # No ingress rules = deny all incoming
```

### Deny All Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
    - Egress                   # No egress rules = deny all outgoing
```

### Deny All (Both Directions)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### Allow All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
spec:
  podSelector: {}
  ingress:
    - {}                       # Empty rule = allow everything
  policyTypes:
    - Ingress
```

### Allow All Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
spec:
  podSelector: {}
  egress:
    - {}                       # Empty rule = allow everything
  policyTypes:
    - Egress
```

---

## 8. Port Ranges

### Single Port

```yaml
ports:
  - protocol: TCP
    port: 80
```

### Port Range (K8s 1.25+)

```yaml
ports:
  - protocol: TCP
    port: 32000
    endPort: 32768             # Range: 32000-32768
```

### Rules for Port Ranges

- `endPort` must be >= `port`
- `endPort` requires `port` to be defined
- Both must be numeric (no named ports with ranges)

### Multiple Ports

```yaml
ports:
  - protocol: TCP
    port: 80
  - protocol: TCP
    port: 443
  - protocol: TCP
    port: 8080
```

---

## 9. Targeting Namespaces

### By Label

```yaml
namespaceSelector:
  matchLabels:
    env: production
```

### By Name (Using Built-in Label)

Kubernetes automatically adds `kubernetes.io/metadata.name` label to all namespaces:

```yaml
namespaceSelector:
  matchLabels:
    kubernetes.io/metadata.name: my-namespace
```

### Multiple Namespaces

```yaml
namespaceSelector:
  matchExpressions:
    - key: env
      operator: In
      values:
        - production
        - staging
```

### Label Your Namespaces

```bash
# Add labels to namespaces
kubectl label namespace frontend env=frontend
kubectl label namespace backend env=backend

# Verify
kubectl get namespaces --show-labels
```

---

## 10. Real-World Examples

### Example 1: Database Pod - Allow Only Backend

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5432
```

### Example 2: Frontend - Allow Internet Egress Only

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

### Example 3: Allow DNS for All Pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### Example 4: Multi-Tier Application

```yaml
# Frontend: Allow ingress from anywhere, egress to backend only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - {}                       # Allow all ingress
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - protocol: TCP
          port: 8080
---
# Backend: Allow from frontend, egress to database only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: database
      ports:
        - protocol: TCP
          port: 5432
---
# Database: Allow from backend only, no egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - protocol: TCP
          port: 5432
  # No egress rules = deny all egress
```

---

## 11. Essential Commands

### Create and Manage

```bash
# Create NetworkPolicy from YAML
kubectl apply -f networkpolicy.yaml

# List NetworkPolicies
kubectl get networkpolicy
kubectl get netpol              # Short form

# List in all namespaces
kubectl get netpol -A

# Describe (see how K8s interpreted it)
kubectl describe netpol <name>

# Delete
kubectl delete netpol <name>

# Get YAML
kubectl get netpol <name> -o yaml
```

### Quick Reference

```bash
# Explain NetworkPolicy spec
kubectl explain networkpolicy.spec
kubectl explain networkpolicy.spec.ingress
kubectl explain networkpolicy.spec.egress

# Watch NetworkPolicies
kubectl get netpol -w
```

### Testing Connectivity

```bash
# Create a test pod
kubectl run test-pod --image=busybox --rm -it --restart=Never -- sh

# From inside the pod, test connectivity
wget -qO- --timeout=2 http://<service-name>:<port>
nc -zv <pod-ip> <port>

# Test DNS
nslookup kubernetes.default
```

---

## 12. Practice Exercises

### Exercise 1: Default Deny All

**Task:** Create a policy that denies all ingress and egress traffic in the `secure` namespace.

```bash
# Create namespace
kubectl create namespace secure

# Create the policy
cat <<EOF | kubectl apply -f -
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
EOF

# Verify
kubectl get netpol -n secure
```

### Exercise 2: Allow Specific Pod Communication

**Task:** In namespace `app`, allow pods with label `role=api` to receive traffic only from pods with label `role=web` on port 8080.

```bash
kubectl create namespace app

cat <<EOF | kubectl apply -f -
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
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: web
      ports:
        - protocol: TCP
          port: 8080
EOF
```

### Exercise 3: Cross-Namespace Communication

**Task:** Allow pods in namespace `frontend` to communicate with pods labeled `app=backend` in namespace `backend` on port 3000.

```bash
# Create namespaces
kubectl create namespace frontend
kubectl create namespace backend
kubectl label namespace backend name=backend

# Create policy in backend namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: frontend
      ports:
        - protocol: TCP
          port: 3000
EOF
```

### Exercise 4: Allow DNS Egress

**Task:** Create a policy that denies all egress except DNS (UDP/TCP 53 to kube-system).

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress-allow-dns
  namespace: default
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
EOF
```

### Exercise 5: Timed Exam Simulation (5 minutes)

**Task:** Complete within 5 minutes.

1. Create namespace `exam-ns`
2. Create a NetworkPolicy named `exam-policy` that:
   - Applies to pods with label `app=secure`
   - Allows ingress from pods with label `app=trusted` on port 443
   - Allows egress to CIDR `10.0.0.0/8` on port 5432
   - Denies all other traffic

---

## 13. Troubleshooting

### Policy Not Working?

| Symptom | Possible Cause | Solution |
|---------|----------------|----------|
| Policy has no effect | CNI doesn't support NetworkPolicy | Use Calico, Cilium, or Weave |
| Pods can still communicate | Policy not selecting pods | Check `podSelector` labels |
| All traffic blocked | Missing allow rules | Add specific ingress/egress rules |
| DNS not working | Egress blocking DNS | Add DNS egress rule |
| Cross-namespace blocked | Missing namespaceSelector | Add proper namespace selection |

### Debug Commands

```bash
# Check if policy is applied
kubectl describe netpol <name>

# Verify pod labels
kubectl get pods --show-labels

# Verify namespace labels
kubectl get namespaces --show-labels

# Test connectivity from a pod
kubectl exec -it <pod> -- wget -qO- --timeout=2 http://<target>

# Check CNI pods
kubectl get pods -n kube-system | grep -E 'calico|cilium|weave'
```

### Common Mistakes

1. **Forgetting DNS egress** - Pods can't resolve service names
2. **Wrong namespace** - Policy in wrong namespace doesn't affect target pods
3. **Label mismatch** - Selector doesn't match pod labels
4. **AND vs OR confusion** - YAML syntax creates wrong logic
5. **Missing policyTypes** - Direction not isolated

---

## 14. Limitations

### What NetworkPolicy CAN'T Do

| Limitation | Workaround |
|------------|------------|
| Force traffic through gateway | Use Service Mesh |
| TLS/encryption | Use Service Mesh or Ingress |
| Target nodes by name | Use CIDR notation |
| Target Services by name | Target pods by labels |
| Log blocked connections | Use CNI-specific features |
| Explicitly DENY rules | Use deny-all + allow rules |
| Block localhost traffic | Not possible |
| Block node-to-pod traffic | Not possible |

### Protocol Support

| Protocol | Support |
|----------|---------|
| TCP | ✅ Fully supported |
| UDP | ✅ Fully supported |
| SCTP | ✅ Supported (if CNI supports) |
| ICMP | ⚠️ Undefined behavior |
| Other | ⚠️ Varies by CNI |

---

## 15. Exam Tips

### Speed Tips

```bash
# Use short form
kubectl get netpol

# Quick YAML generation (then edit)
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

# Use kubectl explain
kubectl explain netpol.spec.ingress.from
```

### Exam Day Checklist

- [ ] Know default deny templates by heart
- [ ] Understand AND vs OR YAML syntax
- [ ] Remember: empty `podSelector: {}` = all pods
- [ ] Remember: empty rule `- {}` = allow all
- [ ] Know the 4 selector types
- [ ] Always verify with `kubectl describe`

### Common Exam Scenarios

1. **Create default deny policy** - Most common
2. **Allow specific pod-to-pod communication** - Use podSelector
3. **Cross-namespace communication** - Use namespaceSelector
4. **Allow external traffic** - Use ipBlock
5. **Allow DNS** - Remember UDP 53

### Time Management

- NetworkPolicy questions: **3-5 minutes each**
- Start with `podSelector` and `policyTypes`
- Add rules incrementally
- Always verify with `describe`

---

## Quick Reference Card

```
DEFAULTS:
  No policy     → All traffic allowed
  Policy exists → Selected pods isolated for specified direction

SELECTORS:
  podSelector       → Same namespace pods
  namespaceSelector → All pods in matching namespaces
  Both combined     → Specific pods in specific namespaces
  ipBlock           → External IP ranges

YAML TRAPS:
  Single element (AND):    Two elements (OR):
  - namespaceSelector:     - namespaceSelector:
      ...                      ...
    podSelector:           - podSelector:
      ...                      ...

EMPTY VALUES:
  podSelector: {}  → All pods in namespace
  ingress: [{}]    → Allow all ingress
  egress: [{}]     → Allow all egress

MUST REMEMBER:
  - Policies are ADDITIVE (union of allows)
  - Both egress AND ingress must allow
  - CNI must support NetworkPolicy
  - DNS needs explicit allow if egress denied
```

---

**Good luck with your CKA exam! 🎯**
