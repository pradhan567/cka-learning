# RBAC - Role Based Access Control

> **CKA Domain:** Cluster Architecture, Installation & Configuration (25%)
> **Official Docs:** https://kubernetes.io/docs/reference/access-authn-authz/rbac/

---

## Table of Contents

1. [What is RBAC?](#1-what-is-rbac)
2. [RBAC API Objects](#2-rbac-api-objects)
3. [Roles and ClusterRoles](#3-roles-and-clusterroles)
4. [RoleBindings and ClusterRoleBindings](#4-rolebindings-and-clusterrolebindings)
5. [Subjects](#5-subjects)
6. [Common RBAC Patterns](#6-common-rbac-patterns)
7. [Essential Commands](#7-essential-commands)
8. [Practice Exercises](#8-practice-exercises)
9. [Exam Tips](#9-exam-tips)

---

## 1. What is RBAC?

**Role-Based Access Control (RBAC)** regulates access to Kubernetes resources based on the roles of individual users or service accounts.

### Key Concepts

```
WHO (Subject)  +  WHAT (Role)  =  Access via (Binding)
     │                │                    │
     ▼                ▼                    ▼
  User            Permissions         RoleBinding/
  Group           on Resources         ClusterRoleBinding
  ServiceAccount
```

### RBAC Components

| Component | Scope | Purpose |
|-----------|-------|---------|
| **Role** | Namespace | Define permissions within a namespace |
| **ClusterRole** | Cluster-wide | Define permissions cluster-wide |
| **RoleBinding** | Namespace | Grant Role/ClusterRole to subjects in namespace |
| **ClusterRoleBinding** | Cluster-wide | Grant ClusterRole to subjects cluster-wide |

---

## 2. RBAC API Objects

### API Group

All RBAC resources are in the `rbac.authorization.k8s.io` API group.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role | ClusterRole | RoleBinding | ClusterRoleBinding
```

### Verbs (Actions)

| Verb | Description |
|------|-------------|
| `get` | Read a specific resource |
| `list` | List resources |
| `watch` | Watch for changes |
| `create` | Create new resources |
| `update` | Modify existing resources |
| `patch` | Partially modify resources |
| `delete` | Delete resources |
| `deletecollection` | Delete multiple resources |

### Common Verb Combinations

| Use Case | Verbs |
|----------|-------|
| Read-only | `get`, `list`, `watch` |
| Read-write | `get`, `list`, `watch`, `create`, `update`, `patch`, `delete` |
| Full access | `*` (all verbs) |

---

## 3. Roles and ClusterRoles

### Role (Namespaced)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]           # "" = core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

### ClusterRole (Cluster-wide)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch"]
```

### Rules Structure

```yaml
rules:
- apiGroups: [""]                    # Core API group (pods, services, etc.)
  resources: ["pods", "services"]
  verbs: ["get", "list"]
  
- apiGroups: ["apps"]                # apps API group (deployments, etc.)
  resources: ["deployments"]
  verbs: ["get", "list", "create"]
  
- apiGroups: [""]
  resources: ["pods/log"]            # Subresource
  verbs: ["get"]
  
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-config"]       # Specific resource by name
  verbs: ["get", "update"]
```

### Common API Groups

| API Group | Resources |
|-----------|-----------|
| `""` (core) | pods, services, configmaps, secrets, nodes, namespaces, pv, pvc |
| `apps` | deployments, replicasets, statefulsets, daemonsets |
| `batch` | jobs, cronjobs |
| `networking.k8s.io` | networkpolicies, ingresses |
| `rbac.authorization.k8s.io` | roles, clusterroles, rolebindings, clusterrolebindings |
| `storage.k8s.io` | storageclasses |

### Find API Group for a Resource

```bash
kubectl api-resources | grep -i deployment
# NAME          SHORTNAMES   APIVERSION   NAMESPACED   KIND
# deployments   deploy       apps/v1      true         Deployment
```

---

## 4. RoleBindings and ClusterRoleBindings

### RoleBinding

Grants permissions within a specific namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding

Grants permissions cluster-wide.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### RoleBinding with ClusterRole

A RoleBinding can reference a ClusterRole to grant permissions **only within the namespace**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: admin-in-dev
  namespace: dev
subjects:
- kind: User
  name: admin-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole        # Using ClusterRole
  name: admin              # Built-in admin ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

---

## 5. Subjects

### User

```yaml
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
```

### Group

```yaml
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
```

### ServiceAccount

```yaml
subjects:
- kind: ServiceAccount
  name: my-service-account
  namespace: default           # Required for ServiceAccount
```

### Multiple Subjects

```yaml
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
- kind: User
  name: john
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: build-bot
  namespace: ci-cd
```

---

## 6. Common RBAC Patterns

### Pattern 1: Read-Only Access to Pods

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: default
subjects:
- kind: User
  name: developer
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Pattern 2: Deployment Manager

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

### Pattern 3: ServiceAccount for CI/CD

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-cd-bot
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-cd-role
  namespace: default
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "update", "patch"]
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-cd-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: ci-cd-bot
  namespace: default
roleRef:
  kind: Role
  name: ci-cd-role
  apiGroup: rbac.authorization.k8s.io
```

### Pattern 4: Cluster-Wide Node Reader

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes-global
subjects:
- kind: Group
  name: ops-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

### Built-in ClusterRoles

| ClusterRole | Description |
|-------------|-------------|
| `cluster-admin` | Full access to everything |
| `admin` | Full access within a namespace |
| `edit` | Read/write access to most resources |
| `view` | Read-only access to most resources |

---

## 7. Essential Commands

### Create Role/ClusterRole

```bash
# Create Role imperatively
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n default

# Create ClusterRole
kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

# Create Role with specific resource names
kubectl create role configmap-updater \
  --verb=get,update \
  --resource=configmaps \
  --resource-name=my-config \
  -n default
```

### Create RoleBinding/ClusterRoleBinding

```bash
# RoleBinding to User
kubectl create rolebinding read-pods \
  --role=pod-reader \
  --user=jane \
  -n default

# RoleBinding to ServiceAccount
kubectl create rolebinding sa-binding \
  --role=pod-reader \
  --serviceaccount=default:my-sa \
  -n default

# ClusterRoleBinding
kubectl create clusterrolebinding read-nodes \
  --clusterrole=node-reader \
  --user=jane

# RoleBinding with ClusterRole
kubectl create rolebinding admin-binding \
  --clusterrole=admin \
  --user=jane \
  -n dev
```

### View and Debug

```bash
# List roles
kubectl get roles -A
kubectl get clusterroles

# List bindings
kubectl get rolebindings -A
kubectl get clusterrolebindings

# Describe
kubectl describe role pod-reader -n default
kubectl describe clusterrole cluster-admin

# Check permissions for a user
kubectl auth can-i get pods --as=jane
kubectl auth can-i create deployments --as=jane -n production
kubectl auth can-i '*' '*' --as=jane  # Check all permissions

# Check permissions for ServiceAccount
kubectl auth can-i get pods --as=system:serviceaccount:default:my-sa

# List all permissions for a user
kubectl auth can-i --list --as=jane
kubectl auth can-i --list --as=jane -n production
```

### Generate YAML

```bash
# Generate Role YAML
kubectl create role pod-reader \
  --verb=get,list \
  --resource=pods \
  --dry-run=client -o yaml > role.yaml

# Generate RoleBinding YAML
kubectl create rolebinding read-pods \
  --role=pod-reader \
  --user=jane \
  --dry-run=client -o yaml > rolebinding.yaml
```

---

## 8. Practice Exercises

### Exercise 1: Basic Role and RoleBinding

**Task:** Create a Role `pod-reader` in namespace `dev` that allows reading pods. Bind it to user `john`.

```bash
# Create namespace
kubectl create namespace dev

# Create Role
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n dev

# Create RoleBinding
kubectl create rolebinding john-pod-reader \
  --role=pod-reader \
  --user=john \
  -n dev

# Verify
kubectl auth can-i get pods --as=john -n dev
kubectl auth can-i create pods --as=john -n dev
```

### Exercise 2: ServiceAccount with Deployment Access

**Task:** Create a ServiceAccount `deploy-bot` in namespace `production`. Grant it permission to manage deployments.

```bash
kubectl create namespace production

# Create ServiceAccount
kubectl create serviceaccount deploy-bot -n production

# Create Role
kubectl create role deployment-manager \
  --verb=get,list,watch,create,update,patch,delete \
  --resource=deployments \
  -n production

# Create RoleBinding
kubectl create rolebinding deploy-bot-binding \
  --role=deployment-manager \
  --serviceaccount=production:deploy-bot \
  -n production

# Verify
kubectl auth can-i create deployments \
  --as=system:serviceaccount:production:deploy-bot \
  -n production
```

### Exercise 3: ClusterRole for Node Access

**Task:** Create a ClusterRole `node-viewer` that allows viewing nodes. Bind it to group `ops-team`.

```bash
# Create ClusterRole
kubectl create clusterrole node-viewer \
  --verb=get,list,watch \
  --resource=nodes

# Create ClusterRoleBinding
kubectl create clusterrolebinding ops-node-viewer \
  --clusterrole=node-viewer \
  --group=ops-team

# Verify
kubectl auth can-i get nodes --as-group=ops-team --as=dummy
```

### Exercise 4: Restrict to Specific ConfigMap

**Task:** Create a Role that allows user `app-admin` to only update ConfigMap named `app-config` in namespace `app`.

```bash
kubectl create namespace app

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-config-updater
  namespace: app
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config"]
  verbs: ["get", "update", "patch"]
EOF

kubectl create rolebinding app-admin-config \
  --role=app-config-updater \
  --user=app-admin \
  -n app
```

### Exercise 5: Timed Exam Simulation (5 minutes)

**Task:** Complete within 5 minutes.

1. Create namespace `secure`
2. Create ServiceAccount `audit-sa` in `secure`
3. Create a Role `audit-role` that can:
   - Get, list pods
   - Get pod logs (`pods/log`)
   - Get, list events
4. Bind the role to the ServiceAccount
5. Verify the ServiceAccount can get pod logs

---

## 9. Exam Tips

### Quick Commands

```bash
# Check if you can do something
kubectl auth can-i <verb> <resource> --as=<user>

# Generate YAML quickly
kubectl create role NAME --verb=VERBS --resource=RESOURCES --dry-run=client -o yaml

# Find API group
kubectl api-resources | grep <resource>
```

### Common Mistakes

1. **Forgetting apiGroup** - Use `""` for core resources
2. **Wrong namespace** - RoleBindings are namespaced
3. **ServiceAccount format** - `--serviceaccount=namespace:name`
4. **Subresources** - Use `pods/log`, `pods/exec` format

### Exam Checklist

- [ ] Know imperative commands for Role/RoleBinding
- [ ] Know how to use `kubectl auth can-i`
- [ ] Understand Role vs ClusterRole scope
- [ ] Know built-in ClusterRoles (admin, edit, view)
- [ ] Know how to find API groups

### Quick Reference

```
Role + RoleBinding           = Namespace permissions
ClusterRole + RoleBinding    = Namespace permissions (reusable ClusterRole)
ClusterRole + ClusterRoleBinding = Cluster-wide permissions
```

---

**Next:** [02-Kubeadm-Cluster-Setup.md](./02-Kubeadm-Cluster-Setup.md)
