# Cluster Architecture - Practice Questions

> **Domain Weight:** 25% | **Estimated Questions:** 4-5

---

## Question 1: RBAC - Create Role and RoleBinding
**Difficulty:** Easy | **Time:** 4 minutes

Create a Role named `pod-reader` in namespace `development` that allows `get`, `list`, and `watch` on pods. Create a RoleBinding named `dev-pod-reader` that binds this role to user `jane`.

<details>
<summary>💡 Solution</summary>

```bash
# Create namespace
kubectl create namespace development

# Create Role
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n development

# Create RoleBinding
kubectl create rolebinding dev-pod-reader \
  --role=pod-reader \
  --user=jane \
  -n development

# Verify
kubectl auth can-i get pods --as=jane -n development
```
</details>

---

## Question 2: RBAC - ServiceAccount Permissions
**Difficulty:** Medium | **Time:** 5 minutes

Create a ServiceAccount named `deploy-sa` in namespace `ci-cd`. Create a ClusterRole named `deployment-manager` that can create, update, and delete deployments. Bind this ClusterRole to the ServiceAccount using a RoleBinding in namespace `production`.

<details>
<summary>💡 Solution</summary>

```bash
# Create namespaces
kubectl create namespace ci-cd
kubectl create namespace production

# Create ServiceAccount
kubectl create serviceaccount deploy-sa -n ci-cd

# Create ClusterRole
kubectl create clusterrole deployment-manager \
  --verb=create,update,delete,get,list \
  --resource=deployments

# Create RoleBinding (binds ClusterRole to SA in production namespace)
kubectl create rolebinding deploy-sa-binding \
  --clusterrole=deployment-manager \
  --serviceaccount=ci-cd:deploy-sa \
  -n production

# Verify
kubectl auth can-i create deployments \
  --as=system:serviceaccount:ci-cd:deploy-sa \
  -n production
```
</details>

---

## Question 3: Cluster Upgrade
**Difficulty:** Hard | **Time:** 8 minutes

You need to upgrade the control plane node from Kubernetes v1.34.0 to v1.35.0. Perform the upgrade following best practices.

<details>
<summary>💡 Solution</summary>

```bash
# 1. Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.35.0-*
sudo apt-mark hold kubeadm

# 2. Verify upgrade plan
sudo kubeadm upgrade plan

# 3. Apply upgrade
sudo kubeadm upgrade apply v1.35.0

# 4. Drain the control plane node
kubectl drain <control-plane-node> --ignore-daemonsets

# 5. Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get update && sudo apt-get install -y kubelet=1.35.0-* kubectl=1.35.0-*
sudo apt-mark hold kubelet kubectl

# 6. Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 7. Uncordon the node
kubectl uncordon <control-plane-node>

# 8. Verify
kubectl get nodes
```
</details>

---

## Question 4: etcd Backup
**Difficulty:** Medium | **Time:** 5 minutes

Create a backup of etcd and save it to `/opt/etcd-backup.db`. The etcd is running on the control plane with the following:
- Endpoint: https://127.0.0.1:2379
- CA cert: /etc/kubernetes/pki/etcd/ca.crt
- Server cert: /etc/kubernetes/pki/etcd/server.crt
- Server key: /etc/kubernetes/pki/etcd/server.key

<details>
<summary>💡 Solution</summary>

```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db --write-out=table
```
</details>

---

## Question 5: etcd Restore
**Difficulty:** Hard | **Time:** 8 minutes

Restore etcd from backup `/opt/etcd-backup.db` to a new data directory `/var/lib/etcd-restored`. Update the etcd configuration to use the new data directory.

<details>
<summary>💡 Solution</summary>

```bash
# 1. Restore snapshot to new directory
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored

# 2. Update etcd manifest
sudo vi /etc/kubernetes/manifests/etcd.yaml

# Change the hostPath for etcd-data volume:
# volumes:
# - hostPath:
#     path: /var/lib/etcd-restored    # Changed from /var/lib/etcd
#     type: DirectoryOrCreate
#   name: etcd-data

# Also update --data-dir flag if present:
# - --data-dir=/var/lib/etcd-restored

# 3. Wait for etcd to restart (automatic when manifest changes)
# 4. Verify
kubectl get pods -n kube-system | grep etcd
```
</details>

---

## Question 6: Helm Installation
**Difficulty:** Easy | **Time:** 4 minutes

Install nginx using Helm from the bitnami repository with the following requirements:
- Release name: `web-server`
- Namespace: `web` (create if not exists)
- Set replica count to 3
- Set service type to NodePort

<details>
<summary>💡 Solution</summary>

```bash
# Add bitnami repo (if not already added)
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install
helm install web-server bitnami/nginx \
  --namespace web \
  --create-namespace \
  --set replicaCount=3 \
  --set service.type=NodePort

# Verify
helm list -n web
kubectl get all -n web
```
</details>

---

## Question 7: Helm Upgrade and Rollback
**Difficulty:** Medium | **Time:** 5 minutes

Upgrade the Helm release `web-server` in namespace `web` to set replicas to 5. Then rollback to the previous version.

<details>
<summary>💡 Solution</summary>

```bash
# Upgrade
helm upgrade web-server bitnami/nginx \
  --namespace web \
  --set replicaCount=5

# Check history
helm history web-server -n web

# Rollback to previous version
helm rollback web-server 1 -n web

# Verify
helm list -n web
kubectl get deployment -n web
```
</details>

---

## Question 8: Kustomize
**Difficulty:** Medium | **Time:** 5 minutes

Given the following base configuration in `/opt/kustomize/base/`, create an overlay in `/opt/kustomize/overlays/prod/` that:
- Sets namespace to `production`
- Adds prefix `prod-` to all resource names
- Changes replica count to 5
- Changes image tag to `1.21-alpine`

<details>
<summary>💡 Solution</summary>

```bash
# Create overlay directory
mkdir -p /opt/kustomize/overlays/prod

# Create kustomization.yaml
cat > /opt/kustomize/overlays/prod/kustomization.yaml <<EOF
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
kubectl kustomize /opt/kustomize/overlays/prod

# Apply
kubectl apply -k /opt/kustomize/overlays/prod
```
</details>

---

## Question 9: CRD
**Difficulty:** Medium | **Time:** 5 minutes

Create a CustomResourceDefinition named `backups.data.example.com` with:
- Group: `data.example.com`
- Version: `v1`
- Kind: `Backup`
- Scope: Namespaced
- Plural: `backups`
- Singular: `backup`
- ShortName: `bk`

<details>
<summary>💡 Solution</summary>

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.data.example.com
spec:
  group: data.example.com
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
  scope: Namespaced
  names:
    plural: backups
    singular: backup
    kind: Backup
    shortNames:
      - bk
```

```bash
kubectl apply -f crd.yaml
kubectl get crd backups.data.example.com
```
</details>

---

## Question 10: Node Maintenance
**Difficulty:** Easy | **Time:** 3 minutes

Node `worker-2` needs maintenance. Safely evict all pods from the node and mark it as unschedulable. After maintenance, make it schedulable again.

<details>
<summary>💡 Solution</summary>

```bash
# Drain node (evict pods and mark unschedulable)
kubectl drain worker-2 --ignore-daemonsets --delete-emptydir-data

# Perform maintenance...

# Make node schedulable again
kubectl uncordon worker-2

# Verify
kubectl get nodes
```
</details>

---

## Question 11: Check User Permissions
**Difficulty:** Easy | **Time:** 2 minutes

Check if user `developer` can:
1. Create deployments in namespace `app`
2. Delete pods in namespace `app`
3. Get secrets in all namespaces

<details>
<summary>💡 Solution</summary>

```bash
# Check create deployments
kubectl auth can-i create deployments --as=developer -n app

# Check delete pods
kubectl auth can-i delete pods --as=developer -n app

# Check get secrets cluster-wide
kubectl auth can-i get secrets --as=developer --all-namespaces

# List all permissions for user
kubectl auth can-i --list --as=developer -n app
```
</details>

---

## Question 12: Join Token
**Difficulty:** Easy | **Time:** 2 minutes

Generate a new join token for adding worker nodes to the cluster. The token should be valid for 4 hours.

<details>
<summary>💡 Solution</summary>

```bash
# Create token with TTL
kubeadm token create --ttl 4h --print-join-command

# List tokens
kubeadm token list
```
</details>

---

## Exam Simulation: Timed Practice

Complete the following tasks within **25 minutes**:

1. **(3 min)** Create namespace `exam-ns`
2. **(4 min)** Create Role `exam-role` in `exam-ns` allowing get, list on pods and deployments
3. **(3 min)** Create ServiceAccount `exam-sa` in `exam-ns`
4. **(3 min)** Bind `exam-role` to `exam-sa`
5. **(5 min)** Backup etcd to `/tmp/etcd-exam-backup.db`
6. **(4 min)** Install nginx via Helm in namespace `exam-web` with 2 replicas
7. **(3 min)** Drain node `worker-1` for maintenance

---

## Quick Reference

```bash
# RBAC
kubectl create role NAME --verb=VERBS --resource=RESOURCES -n NS
kubectl create rolebinding NAME --role=ROLE --user=USER -n NS
kubectl auth can-i VERB RESOURCE --as=USER -n NS

# Cluster Management
kubectl drain NODE --ignore-daemonsets --delete-emptydir-data
kubectl cordon NODE
kubectl uncordon NODE
kubeadm token create --print-join-command

# etcd
ETCDCTL_API=3 etcdctl snapshot save FILE --endpoints=EP --cacert=CA --cert=CERT --key=KEY
ETCDCTL_API=3 etcdctl snapshot restore FILE --data-dir=DIR

# Helm
helm install NAME CHART --set key=val -n NS --create-namespace
helm upgrade NAME CHART --set key=val -n NS
helm rollback NAME REVISION -n NS
helm uninstall NAME -n NS

# Kustomize
kubectl kustomize ./path
kubectl apply -k ./path
```
