# CKA Certification Roadmap 2025

> **Exam Version:** Kubernetes v1.35 | **Duration:** 2 hours | **Passing Score:** 66% | **Format:** Performance-based (hands-on)

---

## Official CKA Exam Domains

| Domain | Weight | Topics |
|--------|--------|--------|
| **1. Cluster Architecture, Installation & Configuration** | 25% | RBAC, kubeadm, HA, Helm, Kustomize, CRDs, Operators |
| **2. Workloads & Scheduling** | 15% | Deployments, ConfigMaps, Secrets, Autoscaling, Scheduling |
| **3. Services & Networking** | 20% | Services, Network Policies, Ingress, Gateway API, CoreDNS |
| **4. Storage** | 10% | StorageClasses, PV, PVC, Volume Types |
| **5. Troubleshooting** | 30% | Cluster, Node, Application, Networking troubleshooting |

---

## Folder Structure

```
K8S Learning/
├── CKA-Roadmap.md                    (This file - Overview)
├── CKA-Storage-Guide.md              (Already created)
│
├── 01-Cluster-Architecture/          (25%)
│   ├── 01-RBAC.md
│   ├── 02-Kubeadm-Cluster-Setup.md
│   ├── 03-Cluster-Lifecycle.md
│   ├── 04-HA-Control-Plane.md
│   ├── 05-Helm-Kustomize.md
│   ├── 06-CRDs-Operators.md
│   ├── 07-Extension-Interfaces.md
│   └── Practice-Questions.md
│
├── 02-Workloads-Scheduling/          (15%)
│   ├── 01-Deployments-Rollouts.md
│   ├── 02-ConfigMaps-Secrets.md
│   ├── 03-Autoscaling.md
│   ├── 04-Self-Healing-Apps.md
│   ├── 05-Pod-Scheduling.md
│   └── Practice-Questions.md
│
├── 03-Services-Networking/           (20%)
│   ├── 01-Pod-Networking.md
│   ├── 02-Services.md
│   ├── 03-Network-Policies.md        (Already created in Network/)
│   ├── 04-Ingress.md
│   ├── 05-Gateway-API.md
│   ├── 06-CoreDNS.md
│   └── Practice-Questions.md
│
├── 04-Storage/                       (10%)
│   ├── 01-Storage-Classes.md
│   ├── 02-PV-PVC.md
│   ├── 03-Volume-Types.md
│   └── Practice-Questions.md
│
├── 05-Troubleshooting/               (30%)
│   ├── 01-Cluster-Node-Troubleshooting.md
│   ├── 02-Component-Troubleshooting.md
│   ├── 03-Resource-Monitoring.md
│   ├── 04-Container-Logs.md
│   ├── 05-Service-Network-Troubleshooting.md
│   └── Practice-Questions.md
│
└── Network/                          (Existing)
    └── CKA-Network-Policies-Guide.md
```

---

## 4-Week Study Plan

### Week 1: Cluster Architecture (25%)
| Day | Topic | Time |
|-----|-------|------|
| 1-2 | RBAC (Roles, ClusterRoles, Bindings) | 4 hrs |
| 3-4 | kubeadm cluster setup & lifecycle | 4 hrs |
| 5 | High Availability control plane | 2 hrs |
| 6 | Helm & Kustomize | 3 hrs |
| 7 | CRDs, Operators, Extension interfaces | 3 hrs |

### Week 2: Workloads & Scheduling (15%) + Services & Networking (20%)
| Day | Topic | Time |
|-----|-------|------|
| 1 | Deployments, Rolling updates, Rollbacks | 2 hrs |
| 2 | ConfigMaps & Secrets | 2 hrs |
| 3 | Autoscaling (HPA) & Self-healing | 2 hrs |
| 4 | Pod Scheduling (affinity, taints, tolerations) | 3 hrs |
| 5 | Services (ClusterIP, NodePort, LoadBalancer) | 3 hrs |
| 6 | Network Policies | 2 hrs |
| 7 | Ingress & Gateway API | 3 hrs |

### Week 3: Storage (10%) + Troubleshooting (30%)
| Day | Topic | Time |
|-----|-------|------|
| 1 | Storage Classes & Dynamic Provisioning | 2 hrs |
| 2 | PV, PVC, Volume Types | 2 hrs |
| 3 | Cluster & Node Troubleshooting | 3 hrs |
| 4 | Component Troubleshooting (etcd, API server) | 3 hrs |
| 5 | Application & Container Troubleshooting | 3 hrs |
| 6 | Service & Network Troubleshooting | 3 hrs |
| 7 | Review & Practice | 3 hrs |

### Week 4: Practice & Mock Exams
| Day | Activity | Time |
|-----|----------|------|
| 1-2 | Killer.sh Simulator Attempt 1 | 4 hrs |
| 3-4 | Review weak areas, practice more | 4 hrs |
| 5-6 | Killer.sh Simulator Attempt 2 | 4 hrs |
| 7 | Final review, rest before exam | 2 hrs |

---

## Official Kubernetes Documentation Links

### Cluster Architecture
- [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [HA Topology](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)
- [Helm](https://helm.sh/docs/)
- [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)

### Workloads & Scheduling
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Node Affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Taints & Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

### Services & Networking
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)
- [DNS for Services](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

### Storage
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Volume Types](https://kubernetes.io/docs/concepts/storage/volumes/)

### Troubleshooting
- [Troubleshoot Clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/)
- [Troubleshoot Applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Debug Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)

---

## Exam Environment Tips

### Allowed Resources During Exam
- https://kubernetes.io/docs/
- https://kubernetes.io/blog/
- https://helm.sh/docs/

### Essential Aliases (Set at exam start)
```bash
alias k=kubectl
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'
alias kd='kubectl describe'
alias kaf='kubectl apply -f'
export do='--dry-run=client -o yaml'
```

### Vim Settings
```bash
echo "set tabstop=2 shiftwidth=2 expandtab" >> ~/.vimrc
```

### Quick YAML Generation
```bash
# Pod
kubectl run nginx --image=nginx $do > pod.yaml

# Deployment
kubectl create deployment nginx --image=nginx $do > deploy.yaml

# Service
kubectl expose deployment nginx --port=80 --target-port=8080 $do > svc.yaml

# ConfigMap
kubectl create configmap my-config --from-literal=key=value $do > cm.yaml

# Secret
kubectl create secret generic my-secret --from-literal=pass=123 $do > secret.yaml
```

---

## Progress Tracker

| Domain | Status | Confidence |
|--------|--------|------------|
| Cluster Architecture (25%) | ⬜ Not Started | ⬜⬜⬜⬜⬜ |
| Workloads & Scheduling (15%) | ⬜ Not Started | ⬜⬜⬜⬜⬜ |
| Services & Networking (20%) | 🟨 In Progress | ⬜⬜⬜⬜⬜ |
| Storage (10%) | ✅ Completed | ⬜⬜⬜⬜⬜ |
| Troubleshooting (30%) | ⬜ Not Started | ⬜⬜⬜⬜⬜ |

---

**Let's start creating the content for each domain!**
