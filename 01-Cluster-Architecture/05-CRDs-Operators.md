# CRDs and Operators

> **CKA Domain:** Cluster Architecture, Installation & Configuration (25%)
> **Official Docs:** 
> - https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
> - https://kubernetes.io/docs/concepts/extend-kubernetes/operator/

---

## Table of Contents

1. [Custom Resource Definitions (CRDs)](#1-custom-resource-definitions-crds)
2. [Creating CRDs](#2-creating-crds)
3. [Working with Custom Resources](#3-working-with-custom-resources)
4. [Operators](#4-operators)
5. [Extension Interfaces (CNI, CSI, CRI)](#5-extension-interfaces-cni-csi-cri)
6. [Practice Exercises](#6-practice-exercises)
7. [Exam Tips](#7-exam-tips)

---

## 1. Custom Resource Definitions (CRDs)

### What are CRDs?

CRDs extend the Kubernetes API by defining new resource types.

```
Built-in Resources:              Custom Resources (via CRD):
- Pod                            - Certificate (cert-manager)
- Deployment                     - VirtualService (Istio)
- Service                        - PostgresCluster (operator)
- ConfigMap                      - MyApp (your custom resource)
```

### Why Use CRDs?

- Extend Kubernetes with domain-specific resources
- Store and retrieve structured data in Kubernetes API
- Use kubectl to manage custom resources
- Leverage Kubernetes RBAC for access control

---

## 2. Creating CRDs

### Basic CRD Structure

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com    # plural.group
spec:
  group: stable.example.com            # API group
  versions:
    - name: v1
      served: true                     # Enable this version
      storage: true                    # Storage version
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  scope: Namespaced                    # or Cluster
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
      - ct
```

### CRD with Validation

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.mycompany.com
spec:
  group: mycompany.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          required:
            - spec
          properties:
            spec:
              type: object
              required:
                - image
                - replicas
              properties:
                image:
                  type: string
                  pattern: '^[a-z0-9]+/[a-z0-9]+:[a-z0-9.]+$'
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
                environment:
                  type: string
                  enum:
                    - dev
                    - staging
                    - prod
  scope: Namespaced
  names:
    plural: applications
    singular: application
    kind: Application
    shortNames:
      - app
```

### Apply CRD

```bash
kubectl apply -f crd.yaml

# Verify CRD is created
kubectl get crd
kubectl describe crd crontabs.stable.example.com
```

---

## 3. Working with Custom Resources

### Create Custom Resource

```yaml
apiVersion: stable.example.com/v1
kind: CronTab
metadata:
  name: my-cron-job
spec:
  cronSpec: "* * * * */5"
  image: my-cron-image
  replicas: 3
```

### Manage Custom Resources

```bash
# Create
kubectl apply -f my-crontab.yaml

# List
kubectl get crontabs
kubectl get ct                    # Using shortName

# Describe
kubectl describe crontab my-cron-job

# Delete
kubectl delete crontab my-cron-job

# Delete CRD (deletes all custom resources too!)
kubectl delete crd crontabs.stable.example.com
```

---

## 4. Operators

### What is an Operator?

An Operator is a method of packaging, deploying, and managing a Kubernetes application using custom resources and controllers.

```
┌─────────────────────────────────────────────────────────┐
│                      Operator                            │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │     CRD     │ +  │ Controller  │ =  │  Automated  │  │
│  │ (Schema)    │    │  (Logic)    │    │  Management │  │
│  └─────────────┘    └─────────────┘    └─────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Operator Pattern

```
User creates CR ──► Controller watches ──► Controller reconciles
     │                    │                       │
     ▼                    ▼                       ▼
PostgresCluster      Detects change         Creates/Updates:
  replicas: 3                               - StatefulSet
  version: 14                               - Services
                                            - ConfigMaps
                                            - Secrets
```

### Popular Operators

| Operator | Purpose |
|----------|---------|
| **Prometheus Operator** | Monitoring |
| **cert-manager** | TLS certificates |
| **Strimzi** | Kafka |
| **PostgreSQL Operator** | PostgreSQL databases |
| **Elastic Cloud on K8s** | Elasticsearch |

### Installing Operators

```bash
# Using kubectl (operator manifests)
kubectl apply -f https://example.com/operator.yaml

# Using Helm
helm install my-operator operator-repo/operator

# Using OLM (Operator Lifecycle Manager)
kubectl apply -f operator-subscription.yaml
```

### Example: cert-manager Operator

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Verify
kubectl get pods -n cert-manager

# Create a Certificate (Custom Resource)
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-cert
  namespace: default
spec:
  secretName: my-cert-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - example.com
EOF
```

---

## 5. Extension Interfaces (CNI, CSI, CRI)

### Container Network Interface (CNI)

Plugins that configure pod networking.

| CNI Plugin | Features |
|------------|----------|
| **Calico** | Network policies, BGP |
| **Cilium** | eBPF, advanced policies |
| **Flannel** | Simple overlay network |
| **Weave** | Mesh networking |

```bash
# Check CNI
ls /etc/cni/net.d/
cat /etc/cni/net.d/10-calico.conflist

# CNI binaries
ls /opt/cni/bin/
```

### Container Storage Interface (CSI)

Plugins that provide storage to pods.

```bash
# List CSI drivers
kubectl get csidrivers

# List CSI nodes
kubectl get csinodes
```

### Container Runtime Interface (CRI)

Interface between kubelet and container runtime.

| Runtime | Description |
|---------|-------------|
| **containerd** | Industry standard |
| **CRI-O** | Lightweight for Kubernetes |
| **Docker** | Via cri-dockerd shim |

```bash
# Check container runtime
kubectl get nodes -o wide  # Look at CONTAINER-RUNTIME column

# Using crictl
crictl ps
crictl images
```

---

## 6. Practice Exercises

### Exercise 1: Create a CRD

```bash
# Create CRD
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webapps.mycompany.com
spec:
  group: mycompany.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                image:
                  type: string
                port:
                  type: integer
  scope: Namespaced
  names:
    plural: webapps
    singular: webapp
    kind: WebApp
    shortNames:
      - wa
EOF

# Verify
kubectl get crd webapps.mycompany.com

# Create custom resource
cat <<EOF | kubectl apply -f -
apiVersion: mycompany.com/v1
kind: WebApp
metadata:
  name: my-webapp
spec:
  image: nginx:1.21
  port: 8080
EOF

# List
kubectl get webapps
kubectl get wa

# Cleanup
kubectl delete webapp my-webapp
kubectl delete crd webapps.mycompany.com
```

### Exercise 2: Explore Existing CRDs

```bash
# List all CRDs in cluster
kubectl get crd

# Describe a CRD
kubectl describe crd <crd-name>

# List custom resources of a CRD
kubectl get <resource-name> -A
```

---

## 7. Exam Tips

### CRD Quick Reference

```bash
# List CRDs
kubectl get crd

# Describe CRD
kubectl describe crd <name>

# Get custom resources
kubectl get <plural-name>

# API resources (shows CRDs too)
kubectl api-resources | grep <group>
```

### Key Points

- CRD name format: `plural.group`
- Deleting CRD deletes all its custom resources
- Operators = CRD + Controller
- Know CNI, CSI, CRI purposes

### Extension Interfaces Summary

| Interface | Purpose | Examples |
|-----------|---------|----------|
| **CNI** | Pod networking | Calico, Cilium, Flannel |
| **CSI** | Storage | AWS EBS, GCE PD, NFS |
| **CRI** | Container runtime | containerd, CRI-O |

---

**Next:** [Practice-Questions.md](./Practice-Questions.md)
