# Helm and Kustomize

> **CKA Domain:** Cluster Architecture, Installation & Configuration (25%)
> **Official Docs:** 
> - https://helm.sh/docs/
> - https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/

---

## Table of Contents

1. [Helm Overview](#1-helm-overview)
2. [Helm Commands](#2-helm-commands)
3. [Working with Helm Charts](#3-working-with-helm-charts)
4. [Kustomize Overview](#4-kustomize-overview)
5. [Kustomize Structure](#5-kustomize-structure)
6. [Kustomize Commands](#6-kustomize-commands)
7. [Practice Exercises](#7-practice-exercises)
8. [Exam Tips](#8-exam-tips)

---

## 1. Helm Overview

**Helm** is the package manager for Kubernetes. It uses **Charts** to define, install, and upgrade applications.

### Key Concepts

| Term | Description |
|------|-------------|
| **Chart** | Package of pre-configured Kubernetes resources |
| **Release** | Instance of a chart running in the cluster |
| **Repository** | Collection of charts |
| **Values** | Configuration for a chart |

### Helm Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Chart     │ ──► │   Helm      │ ──► │  Kubernetes │
│  Repository │     │   Client    │     │   Cluster   │
└─────────────┘     └─────────────┘     └─────────────┘
```

---

## 2. Helm Commands

### Repository Management

```bash
# Add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable

# List repositories
helm repo list

# Update repositories
helm repo update

# Search for charts
helm search repo nginx
helm search hub wordpress    # Search Artifact Hub
```

### Install/Upgrade/Uninstall

```bash
# Install a chart
helm install my-release bitnami/nginx

# Install with custom values
helm install my-release bitnami/nginx --set service.type=NodePort

# Install with values file
helm install my-release bitnami/nginx -f values.yaml

# Install in specific namespace
helm install my-release bitnami/nginx -n my-namespace --create-namespace

# Upgrade a release
helm upgrade my-release bitnami/nginx --set replicaCount=3

# Install or upgrade (if exists)
helm upgrade --install my-release bitnami/nginx

# Uninstall
helm uninstall my-release
helm uninstall my-release -n my-namespace
```

### List and Status

```bash
# List releases
helm list
helm list -A              # All namespaces
helm list -n my-namespace

# Get release status
helm status my-release

# Get release history
helm history my-release

# Rollback to previous version
helm rollback my-release 1
```

### Inspect Charts

```bash
# Show chart info
helm show chart bitnami/nginx

# Show default values
helm show values bitnami/nginx

# Show all info
helm show all bitnami/nginx

# Download chart locally
helm pull bitnami/nginx
helm pull bitnami/nginx --untar
```

### Template and Debug

```bash
# Render templates locally (dry-run)
helm template my-release bitnami/nginx

# Dry-run install (shows what would be created)
helm install my-release bitnami/nginx --dry-run

# Debug
helm install my-release bitnami/nginx --debug --dry-run
```

---

## 3. Working with Helm Charts

### Chart Structure

```
mychart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration values
├── charts/             # Dependency charts
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl    # Template helpers
│   └── NOTES.txt       # Post-install notes
└── README.md
```

### Chart.yaml Example

```yaml
apiVersion: v2
name: mychart
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### values.yaml Example

```yaml
replicaCount: 1

image:
  repository: nginx
  tag: "1.21"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
```

### Template Example

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nginx
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
      - name: nginx
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: 80
```

---

## 4. Kustomize Overview

**Kustomize** is a template-free way to customize Kubernetes configurations using overlays and patches.

### Key Concepts

| Term | Description |
|------|-------------|
| **Base** | Original/common configuration |
| **Overlay** | Environment-specific customizations |
| **Patch** | Modifications to apply |
| **kustomization.yaml** | Configuration file |

### Kustomize vs Helm

| Feature | Helm | Kustomize |
|---------|------|-----------|
| Templating | Yes (Go templates) | No |
| Package management | Yes | No |
| Built into kubectl | No | Yes |
| Learning curve | Higher | Lower |

---

## 5. Kustomize Structure

### Basic Structure

```
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

### Base kustomization.yaml

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

### Overlay kustomization.yaml

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: production

namePrefix: prod-

commonLabels:
  env: production

replicas:
  - name: my-app
    count: 5

images:
  - name: nginx
    newTag: "1.21-prod"
```

### Common Kustomization Fields

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Include resources
resources:
  - deployment.yaml
  - service.yaml
  - ../../base

# Set namespace for all resources
namespace: my-namespace

# Add prefix/suffix to names
namePrefix: dev-
nameSuffix: -v1

# Add labels to all resources
commonLabels:
  app: myapp
  env: dev

# Add annotations
commonAnnotations:
  owner: team-a

# Change image tags
images:
  - name: nginx
    newName: my-registry/nginx
    newTag: "2.0"

# Change replica count
replicas:
  - name: my-deployment
    count: 3

# Apply patches
patches:
  - path: patch.yaml
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: my-deployment
      spec:
        replicas: 5

# Generate ConfigMaps
configMapGenerator:
  - name: my-config
    literals:
      - KEY=value
    files:
      - config.properties

# Generate Secrets
secretGenerator:
  - name: my-secret
    literals:
      - password=secret123
```

---

## 6. Kustomize Commands

### Using kubectl

```bash
# View generated manifests
kubectl kustomize ./overlays/prod

# Apply kustomization
kubectl apply -k ./overlays/prod

# Delete resources
kubectl delete -k ./overlays/prod

# Diff before applying
kubectl diff -k ./overlays/prod
```

### Using kustomize CLI

```bash
# Build manifests
kustomize build ./overlays/prod

# Build and apply
kustomize build ./overlays/prod | kubectl apply -f -
```

---

## 7. Practice Exercises

### Exercise 1: Helm - Install and Customize

```bash
# Add bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install nginx with custom values
helm install my-nginx bitnami/nginx \
  --set replicaCount=2 \
  --set service.type=NodePort \
  -n web --create-namespace

# Verify
helm list -n web
kubectl get all -n web

# Upgrade
helm upgrade my-nginx bitnami/nginx \
  --set replicaCount=3 \
  -n web

# Rollback
helm rollback my-nginx 1 -n web

# Uninstall
helm uninstall my-nginx -n web
```

### Exercise 2: Kustomize - Create Overlays

```bash
# Create directory structure
mkdir -p kustomize-demo/base kustomize-demo/overlays/{dev,prod}

# Create base deployment
cat > kustomize-demo/base/deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
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
EOF

# Create base kustomization
cat > kustomize-demo/base/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
EOF

# Create prod overlay
cat > kustomize-demo/overlays/prod/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namespace: production
namePrefix: prod-
replicas:
  - name: nginx
    count: 5
images:
  - name: nginx
    newTag: "1.21-alpine"
EOF

# Preview
kubectl kustomize kustomize-demo/overlays/prod

# Apply
kubectl apply -k kustomize-demo/overlays/prod
```

### Exercise 3: Helm - Show Values and Template

```bash
# Show default values
helm show values bitnami/nginx > nginx-values.yaml

# Edit values
vi nginx-values.yaml

# Template with custom values (dry-run)
helm template my-nginx bitnami/nginx -f nginx-values.yaml

# Install with values file
helm install my-nginx bitnami/nginx -f nginx-values.yaml
```

---

## 8. Exam Tips

### Helm Quick Reference

```bash
# Install
helm install NAME CHART [--set key=val] [-f values.yaml] [-n namespace]

# Upgrade
helm upgrade NAME CHART [--set key=val]

# List
helm list [-A] [-n namespace]

# Uninstall
helm uninstall NAME [-n namespace]

# Search
helm search repo KEYWORD

# Show values
helm show values CHART
```

### Kustomize Quick Reference

```bash
# Preview
kubectl kustomize ./path

# Apply
kubectl apply -k ./path

# Delete
kubectl delete -k ./path
```

### Key Points

- Helm uses Go templating, Kustomize uses overlays
- `kubectl apply -k` is built-in for Kustomize
- Helm manages releases with history and rollback
- Kustomize is template-free, uses patches

---

**Next:** [05-CRDs-Operators.md](./05-CRDs-Operators.md)
