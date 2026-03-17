# CKA Storage Mastery Guide (10% of Exam)

> **Target:** 1 Month | **Difficulty:** Ruthless | **Goal:** Pass CKA

---

## Table of Contents

1. [Persistent Volumes (PV)](#1-persistent-volumes-pv)
2. [Persistent Volume Claims (PVC)](#2-persistent-volume-claims-pvc)
3. [Access Modes](#3-access-modes)
4. [Reclaim Policies](#4-reclaim-policies)
5. [Storage Classes](#5-storage-classes)
6. [Dynamic Volume Provisioning](#6-dynamic-volume-provisioning)
7. [Volume Types](#7-volume-types)
8. [Pod with Volumes](#8-pod-with-volumes)
9. [StorageClass Internals & VolumeBindingMode](#9-storageclass-internals--volumebindingmode)
10. [CSI Drivers](#10-csi-drivers)
11. [StatefulSet Storage](#11-statefulset-storage)
12. [Essential Commands](#12-essential-commands)
13. [Practice Exercises](#13-practice-exercises)
14. [CKA Troubleshooting Scenarios](#14-cka-troubleshooting-scenarios)
15. [Exam Tips](#15-exam-tips)

---

## 1. Persistent Volumes (PV)

A **PersistentVolume (PV)** is a piece of storage in the cluster provisioned by an administrator or dynamically using Storage Classes.

### Key Characteristics

- Cluster-scoped resource (not namespaced)
- Lifecycle independent of any Pod
- Can be provisioned statically or dynamically

### PV YAML Template

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

### PV Spec Fields

| Field | Description |
|-------|-------------|
| `capacity.storage` | Size of the volume |
| `accessModes` | How the volume can be mounted |
| `persistentVolumeReclaimPolicy` | What happens when PVC is deleted |
| `storageClassName` | Links to StorageClass (empty = no class) |
| `hostPath/nfs/csi/etc` | The actual storage backend |

### PV Status Phases

| Phase | Description |
|-------|-------------|
| `Available` | Free, not bound to a PVC |
| `Bound` | Bound to a PVC |
| `Released` | PVC deleted, but resource not reclaimed |
| `Failed` | Automatic reclamation failed |

---

## 2. Persistent Volume Claims (PVC)

A **PersistentVolumeClaim (PVC)** is a request for storage by a user.

### Key Characteristics

- Namespace-scoped resource
- Binds to a PV that satisfies its requirements
- Pods use PVCs to request storage

### PVC YAML Template

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: manual
```

### PVC Spec Fields

| Field | Description |
|-------|-------------|
| `accessModes` | Must match or be subset of PV |
| `resources.requests.storage` | Minimum storage required |
| `storageClassName` | Must match PV's storageClassName |
| `selector` | Optional label selector for PV |

### PVC Status Phases

| Phase | Description |
|-------|-------------|
| `Pending` | No matching PV found |
| `Bound` | Successfully bound to a PV |
| `Lost` | Bound PV has been deleted |

---

## 3. Access Modes

### MEMORIZE THIS TABLE

| Mode | Abbreviation | Description | Use Case |
|------|--------------|-------------|----------|
| **ReadWriteOnce** | RWO | Single node read-write | Databases, single-instance apps |
| **ReadOnlyMany** | ROX | Multiple nodes read-only | Shared configs, static content |
| **ReadWriteMany** | RWX | Multiple nodes read-write | Shared file storage, CMS |
| **ReadWriteOncePod** | RWOP | Single pod read-write | Strict single-writer scenarios |

### Access Mode Support by Volume Type

| Volume Type | RWO | ROX | RWX |
|-------------|-----|-----|-----|
| hostPath | ✅ | ❌ | ❌ |
| NFS | ✅ | ✅ | ✅ |
| AWS EBS | ✅ | ❌ | ❌ |
| Azure Disk | ✅ | ❌ | ❌ |
| GCE PD | ✅ | ✅ | ❌ |

⚠️ **Exam Trap:** A PV can have multiple access modes, but a PVC binds using only ONE mode at a time.

---

## 4. Reclaim Policies

### Policy Comparison

| Policy | Behavior | When to Use |
|--------|----------|-------------|
| **Retain** | PV preserved, data intact, manual cleanup needed | Production data, compliance |
| **Delete** | PV and underlying storage deleted automatically | Dev/test, ephemeral data |
| **Recycle** | ⚠️ DEPRECATED - basic scrub (`rm -rf /volume/*`) | Don't use |

### Change Reclaim Policy

```bash
# Patch existing PV
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

### What Happens After PVC Deletion

1. **Retain:** PV status → `Released`, data preserved, must manually delete PV
2. **Delete:** PV and storage deleted automatically

---

## 5. Storage Classes

A **StorageClass** provides a way to describe different "classes" of storage.

### StorageClass YAML Template

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # Optional: make default
provisioner: kubernetes.io/no-provisioner
parameters:
  type: pd-ssd  # Provisioner-specific
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### StorageClass Fields

| Field | Description |
|-------|-------------|
| `provisioner` | Which volume plugin to use |
| `parameters` | Provisioner-specific settings |
| `reclaimPolicy` | Default reclaim policy for PVs |
| `volumeBindingMode` | When to bind PV to PVC |
| `allowVolumeExpansion` | Allow PVC resize |

### Volume Binding Modes

| Mode | Behavior |
|------|----------|
| `Immediate` | Bind PV as soon as PVC is created |
| `WaitForFirstConsumer` | Bind when Pod using PVC is scheduled |

**Best Practice:** Use `WaitForFirstConsumer` for topology-aware provisioning.

### Common Provisioners

| Provisioner | Platform |
|-------------|----------|
| `kubernetes.io/no-provisioner` | Local/Manual |
| `kubernetes.io/aws-ebs` | AWS |
| `kubernetes.io/gce-pd` | GCP |
| `kubernetes.io/azure-disk` | Azure |
| `rancher.io/local-path` | Rancher/K3s |

---

## 6. Dynamic Volume Provisioning

Dynamic provisioning automatically creates PVs when PVCs are created.

### How It Works

1. Admin creates StorageClass with provisioner
2. User creates PVC referencing StorageClass
3. Provisioner automatically creates PV
4. PVC binds to newly created PV

### PVC for Dynamic Provisioning

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: fast-storage  # References StorageClass
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

### Default StorageClass

```bash
# Check default StorageClass
kubectl get sc

# Set default StorageClass
kubectl patch storageclass <name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

If PVC has `storageClassName: ""` → No dynamic provisioning, only static binding.

---

## 7. Volume Types

### Volume Types for CKA Exam

| Type | Persistence | Use Case |
|------|-------------|----------|
| `emptyDir` | Pod lifetime | Temp storage, cache |
| `hostPath` | Node lifetime | Testing, single-node |
| `configMap` | ConfigMap lifetime | Config files |
| `secret` | Secret lifetime | Sensitive data |
| `persistentVolumeClaim` | PV lifetime | Persistent data |
| `nfs` | External | Shared storage |

### emptyDir Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-emptydir
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

### hostPath Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hostpath
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /data
      name: host-volume
  volumes:
  - name: host-volume
    hostPath:
      path: /mnt/data
      type: DirectoryOrCreate
```

### hostPath Types

| Type | Behavior |
|------|----------|
| `""` | No checks |
| `DirectoryOrCreate` | Create dir if not exists |
| `Directory` | Must exist |
| `FileOrCreate` | Create file if not exists |
| `File` | Must exist |

---

## 8. Pod with Volumes

### Pod Using PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pvc
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: web-storage
  volumes:
  - name: web-storage
    persistentVolumeClaim:
      claimName: pvc-example
```

### Pod with Multiple Volumes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-volume-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /data
      name: persistent-storage
    - mountPath: /cache
      name: cache-storage
    - mountPath: /config
      name: config-volume
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: my-pvc
  - name: cache-storage
    emptyDir: {}
  - name: config-volume
    configMap:
      name: my-config
```

---

## 9. StorageClass Internals & VolumeBindingMode

### StorageClass Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      StorageClass                           │
├─────────────────────────────────────────────────────────────┤
│  provisioner ──────► CSI Driver / In-tree Plugin            │
│  parameters ───────► Backend-specific config                │
│  reclaimPolicy ────► Delete / Retain                        │
│  volumeBindingMode → Immediate / WaitForFirstConsumer       │
│  allowVolumeExpansion → true / false                        │
│  mountOptions ─────► Mount flags for the volume             │
└─────────────────────────────────────────────────────────────┘
```

### VolumeBindingMode Deep Dive

#### Immediate Mode

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: immediate-sc
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: Immediate  # DEFAULT
```

**Behavior:**
- PV binds to PVC immediately upon PVC creation
- Does NOT consider Pod scheduling constraints
- Can cause issues in multi-zone clusters

**Problem Scenario:**
```
1. PVC created → Binds to PV in Zone-A
2. Pod scheduled → Lands on Node in Zone-B
3. Result: Pod stuck in Pending (volume not accessible)
```

#### WaitForFirstConsumer Mode

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: wait-sc
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

**Behavior:**
- PV binding delayed until Pod is scheduled
- Considers Pod's node affinity, taints, tolerations
- **Required for topology-aware provisioning**

**Flow:**
```
1. PVC created → Status: Pending (waiting)
2. Pod created referencing PVC
3. Scheduler finds suitable node
4. PV bound considering node topology
5. Pod starts successfully
```

### When to Use Each Mode

| Scenario | Recommended Mode |
|----------|------------------|
| Single-zone cluster | Either works |
| Multi-zone cluster | WaitForFirstConsumer |
| Local PVs | WaitForFirstConsumer (REQUIRED) |
| Network storage (NFS) | Immediate is fine |
| Cloud dynamic provisioning | WaitForFirstConsumer |

### Local Persistent Volumes (Exam Important!)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer  # MANDATORY for local PVs
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:  # REQUIRED for local PVs
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-node-1
```

### Volume Expansion

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-sc
provisioner: kubernetes.io/aws-ebs
allowVolumeExpansion: true  # Enable expansion
```

**Expand a PVC:**
```bash
# Edit PVC and increase storage request
kubectl edit pvc my-pvc
# Change: storage: 5Gi → storage: 10Gi

# Or patch it
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'
```

⚠️ **Exam Note:** Volume can only be EXPANDED, never shrunk!

---

## 10. CSI Drivers

### What is CSI?

**Container Storage Interface (CSI)** is a standard for exposing storage systems to containerized workloads.

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Kubelet    │────►│  CSI Driver  │────►│   Storage    │
│              │     │  (DaemonSet) │     │   Backend    │
└──────────────┘     └──────────────┘     └──────────────┘
```

### CSI Components

| Component | Description |
|-----------|-------------|
| **CSI Driver** | DaemonSet running on each node |
| **CSI Controller** | Deployment handling provisioning |
| **CSIDriver Object** | Cluster resource describing driver |
| **CSINode Object** | Node-specific CSI info |

### Check CSI Drivers

```bash
# List CSI drivers in cluster
kubectl get csidrivers

# Describe a CSI driver
kubectl describe csidriver <driver-name>

# Check CSI nodes
kubectl get csinodes
```

### Common CSI Drivers for CKA

| Driver | Use Case |
|--------|----------|
| `hostpath.csi.k8s.io` | Testing/Development |
| `ebs.csi.aws.com` | AWS EBS |
| `pd.csi.storage.gke.io` | GCP Persistent Disk |
| `disk.csi.azure.com` | Azure Disk |
| `nfs.csi.k8s.io` | NFS shares |

### CSI StorageClass Example

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-hostpath-sc
provisioner: hostpath.csi.k8s.io  # CSI driver name
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
parameters:
  # Driver-specific parameters
```

### CSI vs In-Tree Plugins

| Aspect | In-Tree | CSI |
|--------|---------|-----|
| Location | Built into K8s | External |
| Updates | K8s release cycle | Independent |
| Future | Being deprecated | Standard |
| Flexibility | Limited | Extensible |

⚠️ **Exam Note:** In-tree plugins (kubernetes.io/*) are being migrated to CSI. Know both!

---

## 11. StatefulSet Storage

### Why StatefulSet for Storage?

| Feature | Deployment | StatefulSet |
|---------|------------|-------------|
| Pod identity | Random | Stable, ordered |
| Storage | Shared PVC | Unique PVC per Pod |
| Scaling | Any order | Ordered |
| Use case | Stateless apps | Databases, queues |

### VolumeClaimTemplates

StatefulSets use `volumeClaimTemplates` to create unique PVCs for each Pod.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:  # Creates PVC per Pod
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

### What Gets Created

```
StatefulSet: mysql (replicas: 3)
    │
    ├── Pod: mysql-0 ──► PVC: data-mysql-0 ──► PV: pv-xxx
    ├── Pod: mysql-1 ──► PVC: data-mysql-1 ──► PV: pv-yyy
    └── Pod: mysql-2 ──► PVC: data-mysql-2 ──► PV: pv-zzz
```

### PVC Naming Convention

```
<volumeClaimTemplate-name>-<statefulset-name>-<ordinal>

Example: data-mysql-0, data-mysql-1, data-mysql-2
```

### StatefulSet Storage Behavior

**Scaling Up:**
```bash
kubectl scale sts mysql --replicas=5
# Creates: data-mysql-3, data-mysql-4 (new PVCs)
```

**Scaling Down:**
```bash
kubectl scale sts mysql --replicas=2
# Pods mysql-2, mysql-3, mysql-4 deleted
# PVCs data-mysql-2, data-mysql-3, data-mysql-4 RETAINED!
```

⚠️ **Critical:** PVCs are NOT deleted when scaling down! Manual cleanup required.

### Delete StatefulSet but Keep Data

```bash
# Delete StatefulSet only (keep PVCs)
kubectl delete sts mysql --cascade=orphan

# PVCs remain, can be reattached later
kubectl get pvc
```

### Headless Service for StatefulSet

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None  # Headless
  selector:
    app: mysql
  ports:
  - port: 3306
```

**DNS Resolution:**
```
mysql-0.mysql.default.svc.cluster.local
mysql-1.mysql.default.svc.cluster.local
mysql-2.mysql.default.svc.cluster.local
```

### Complete StatefulSet Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
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
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 1Gi
```

---

## 12. Essential Commands

### PV Commands

```bash
# List PVs
kubectl get pv

# Detailed PV info
kubectl describe pv <pv-name>

# Create PV from YAML
kubectl apply -f pv.yaml

# Delete PV
kubectl delete pv <pv-name>

# Get PV YAML
kubectl get pv <pv-name> -o yaml
```

### PVC Commands

```bash
# List PVCs (all namespaces)
kubectl get pvc -A

# List PVCs (specific namespace)
kubectl get pvc -n <namespace>

# Detailed PVC info
kubectl describe pvc <pvc-name>

# Create PVC from YAML
kubectl apply -f pvc.yaml

# Delete PVC
kubectl delete pvc <pvc-name>
```

### StorageClass Commands

```bash
# List StorageClasses
kubectl get sc

# Describe StorageClass
kubectl describe sc <sc-name>

# Get default StorageClass
kubectl get sc -o jsonpath='{.items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")].metadata.name}'
```

### Quick Reference Commands

```bash
# Explain PV spec
kubectl explain pv.spec

# Explain PVC spec
kubectl explain pvc.spec

# Explain StorageClass
kubectl explain sc

# Watch PV/PVC status
kubectl get pv,pvc -w
```

---

## 13. Practice Exercises

### Exercise 1: Basic PV and PVC Binding

**Task:** Create a PV and PVC that bind together.

```bash
# 1. Create PV
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: exercise1-pv
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/exercise1
EOF

# 2. Create PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: exercise1-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Mi
EOF

# 3. Verify binding
kubectl get pv,pvc

# 4. Cleanup
kubectl delete pvc exercise1-pvc
kubectl get pv  # Check status is Released
kubectl delete pv exercise1-pv
```

### Exercise 2: StorageClass with WaitForFirstConsumer

**Task:** Create a StorageClass and observe binding behavior.

```bash
# 1. Create StorageClass
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: delayed-binding
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF

# 2. Create PV with this StorageClass
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: exercise2-pv
spec:
  capacity:
    storage: 200Mi
  accessModes:
    - ReadWriteOnce
  storageClassName: delayed-binding
  hostPath:
    path: /mnt/exercise2
EOF

# 3. Create PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: exercise2-pvc
spec:
  storageClassName: delayed-binding
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
EOF

# 4. Check - PVC should be Pending
kubectl get pvc exercise2-pvc

# 5. Create Pod to trigger binding
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: exercise2-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /data
      name: storage
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: exercise2-pvc
EOF

# 6. Now check - PVC should be Bound
kubectl get pvc exercise2-pvc
```

### Exercise 3: Reclaim Policy Test

**Task:** Observe different reclaim policy behaviors.

```bash
# 1. Create PV with Delete policy
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: delete-policy-pv
spec:
  capacity:
    storage: 50Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  hostPath:
    path: /mnt/delete-test
EOF

# 2. Create matching PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: delete-policy-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Mi
EOF

# 3. Verify bound
kubectl get pv,pvc

# 4. Delete PVC and observe PV behavior
kubectl delete pvc delete-policy-pvc
kubectl get pv  # PV should be gone (Delete policy)
```

### Exercise 4: Troubleshooting Pending PVC

**Task:** Debug why a PVC is stuck in Pending.

```bash
# 1. Create PV with specific requirements
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: small-pv
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  hostPath:
    path: /mnt/small
EOF

# 2. Create PVC requesting MORE than available
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: large-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi  # More than PV has!
EOF

# 3. Check status
kubectl get pvc large-pvc  # Should be Pending

# 4. Debug
kubectl describe pvc large-pvc  # Look at Events

# 5. Fix: Either create larger PV or reduce PVC request
```

### Exercise 5: Timed Exam Simulation (5 minutes)

**Task:** Complete within 5 minutes.

1. Create a PV named `exam-pv` with:
   - 250Mi capacity
   - ReadWriteMany access mode
   - Retain reclaim policy
   - hostPath: `/mnt/exam`

2. Create a PVC named `exam-pvc` that:
   - Requests 200Mi
   - Uses ReadWriteMany access mode

3. Create a Pod named `exam-pod` that:
   - Uses nginx image
   - Mounts the PVC at `/usr/share/nginx/html`

---

## 14. CKA Troubleshooting Scenarios

### Scenario 1: PVC Stuck in Pending

**Symptoms:**
```bash
$ kubectl get pvc
NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc    Pending                                      standard       5m
```

**Debug Steps:**
```bash
# Step 1: Check PVC events
kubectl describe pvc my-pvc

# Step 2: Check available PVs
kubectl get pv

# Step 3: Check StorageClass
kubectl get sc
```

**Common Causes & Fixes:**

| Cause | How to Identify | Fix |
|-------|-----------------|-----|
| No matching PV | `no persistent volumes available` | Create PV with matching specs |
| StorageClass mismatch | PVC and PV have different `storageClassName` | Correct the storageClassName |
| Access mode mismatch | PVC requests RWX, PV only has RWO | Match access modes |
| Capacity insufficient | PVC requests 10Gi, PV only has 5Gi | Create larger PV or reduce PVC |
| No default StorageClass | PVC has no storageClassName, no default SC | Set default SC or specify one |

**Fix Example:**
```bash
# Check what PVC needs
kubectl get pvc my-pvc -o yaml | grep -A5 spec:

# Create matching PV
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: fix-pv
spec:
  capacity:
    storage: 10Gi  # Match or exceed PVC request
  accessModes:
    - ReadWriteOnce  # Match PVC access mode
  storageClassName: standard  # Match PVC storageClassName
  hostPath:
    path: /mnt/data
EOF
```

---

### Scenario 2: Pod Stuck in Pending Due to Volume

**Symptoms:**
```bash
$ kubectl get pods
NAME     READY   STATUS    RESTARTS   AGE
my-pod   0/1     Pending   0          5m

$ kubectl describe pod my-pod
Events:
  Warning  FailedScheduling  pod has unbound immediate PersistentVolumeClaims
```

**Debug Steps:**
```bash
# Step 1: Check which PVC the pod uses
kubectl get pod my-pod -o yaml | grep -A3 volumes:

# Step 2: Check PVC status
kubectl get pvc

# Step 3: If PVC is bound, check node affinity
kubectl describe pv <pv-name> | grep -A10 nodeAffinity
```

**Common Causes & Fixes:**

| Cause | Fix |
|-------|-----|
| PVC not bound | Fix PVC binding first |
| Volume in different zone | Use `WaitForFirstConsumer` |
| Local PV on wrong node | Check nodeAffinity in PV |
| Node doesn't have storage | Schedule pod to correct node |

---

### Scenario 3: Pod Can't Mount Volume

**Symptoms:**
```bash
$ kubectl describe pod my-pod
Events:
  Warning  FailedMount  Unable to attach or mount volumes
```

**Debug Steps:**
```bash
# Check mount errors
kubectl describe pod my-pod | grep -A5 Events

# Check PV/PVC binding
kubectl get pv,pvc

# Check if path exists on node (for hostPath)
kubectl debug node/<node-name> -it --image=busybox -- ls -la /mnt/data
```

**Common Causes & Fixes:**

| Error | Cause | Fix |
|-------|-------|-----|
| `path does not exist` | hostPath directory missing | Create directory on node |
| `permission denied` | Wrong permissions | Fix permissions or use fsGroup |
| `volume is already used` | RWO volume on multiple nodes | Use RWX or schedule to same node |

**Fix with fsGroup:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  securityContext:
    fsGroup: 1000  # Group ID for volume access
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /data
      name: my-vol
  volumes:
  - name: my-vol
    persistentVolumeClaim:
      claimName: my-pvc
```

---

### Scenario 4: PV Stuck in Released State

**Symptoms:**
```bash
$ kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM
my-pv   10Gi       RWO            Retain           Released   default/old-pvc
```

**Cause:** PVC was deleted, but PV has `Retain` policy.

**Fix Options:**

**Option 1: Reuse the PV (remove claimRef)**
```bash
# Edit PV and remove claimRef section
kubectl patch pv my-pv --type json -p '[{"op": "remove", "path": "/spec/claimRef"}]'

# PV status changes to Available
kubectl get pv my-pv
```

**Option 2: Delete and recreate PV**
```bash
# Backup data if needed, then delete
kubectl delete pv my-pv

# Recreate PV
kubectl apply -f pv.yaml
```

---

### Scenario 5: Volume Expansion Not Working

**Symptoms:**
```bash
$ kubectl edit pvc my-pvc
# Change storage from 5Gi to 10Gi
# But actual size doesn't change
```

**Debug Steps:**
```bash
# Check if StorageClass allows expansion
kubectl get sc <storageclass-name> -o yaml | grep allowVolumeExpansion

# Check PVC conditions
kubectl describe pvc my-pvc | grep -A5 Conditions
```

**Common Causes & Fixes:**

| Cause | Fix |
|-------|-----|
| `allowVolumeExpansion: false` | Create new SC with expansion enabled |
| Expansion pending | Restart pod to trigger filesystem resize |
| Volume type doesn't support | Use different storage backend |

**Trigger Filesystem Resize:**
```bash
# Delete and recreate pod to trigger resize
kubectl delete pod <pod-using-pvc>
# Pod will be recreated by controller and resize will happen
```

---

### Scenario 6: StatefulSet PVC Issues

**Symptoms:**
```bash
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
mysql-0   1/1     Running   0          10m
mysql-1   0/1     Pending   0          5m
```

**Debug Steps:**
```bash
# Check PVCs for StatefulSet
kubectl get pvc | grep mysql

# Check if PV exists for mysql-1
kubectl get pv

# Check events
kubectl describe pod mysql-1
```

**Common Causes & Fixes:**

| Cause | Fix |
|-------|-----|
| Not enough PVs | Create additional PVs |
| StorageClass can't provision | Check provisioner logs |
| Node capacity full | Add storage to nodes |

**Manual PV Creation for StatefulSet:**
```bash
# Create PV for mysql-1
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-1
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  hostPath:
    path: /mnt/mysql-1
EOF
```

---

### Scenario 7: WaitForFirstConsumer Not Binding

**Symptoms:**
```bash
$ kubectl get pvc
NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc    Pending                                      local-sc       10m

$ kubectl describe pvc my-pvc
Events:
  Normal  WaitForFirstConsumer  waiting for first consumer to be created
```

**This is NORMAL behavior!** PVC waits for Pod.

**Fix:** Create a Pod that uses the PVC:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: consumer-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /data
      name: storage
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc
EOF

# Now check - PVC should be Bound
kubectl get pvc my-pvc
```

---

### Scenario 8: Wrong Access Mode Causing Multi-Pod Failure

**Symptoms:**
```bash
# First pod works
$ kubectl get pods
NAME      READY   STATUS              RESTARTS   AGE
app-1     1/1     Running             0          5m
app-2     0/1     ContainerCreating   0          1m

$ kubectl describe pod app-2
Events:
  Warning  FailedAttachVolume  Multi-Attach error for volume
```

**Cause:** Using RWO volume with multiple pods on different nodes.

**Fix Options:**

**Option 1: Use RWX access mode (if supported)**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  accessModes:
    - ReadWriteMany  # Changed from ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

**Option 2: Use node affinity to schedule pods together**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-2
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: myapp
        topologyKey: kubernetes.io/hostname
  # ... rest of pod spec
```

---

### Quick Troubleshooting Checklist

```bash
# 1. Check all storage resources
kubectl get pv,pvc,sc

# 2. Describe problematic resource
kubectl describe pvc <name>
kubectl describe pv <name>
kubectl describe pod <name>

# 3. Check events
kubectl get events --sort-by='.lastTimestamp' | grep -i volume

# 4. Verify StorageClass
kubectl get sc -o wide

# 5. Check CSI driver (if applicable)
kubectl get csidrivers
kubectl get pods -n kube-system | grep csi
```

---

## 15. Exam Tips

### Speed Tips

```bash
# Use kubectl explain liberally
kubectl explain pv.spec --recursive | less
kubectl explain pvc.spec --recursive | less

# Generate YAML quickly
kubectl create -f - <<EOF
[paste yaml here]
EOF

# Use aliases (add to ~/.bashrc)
alias k=kubectl
alias kgpv='kubectl get pv'
alias kgpvc='kubectl get pvc'
```

### Exam Day Checklist

- [ ] Can create PV from scratch in <1 minute
- [ ] Can create PVC from scratch in <1 minute
- [ ] Know all access modes (RWO, ROX, RWX, RWOP)
- [ ] Know reclaim policies (Retain, Delete)
- [ ] Understand StorageClass and dynamic provisioning
- [ ] Can troubleshoot Pending PVCs quickly
- [ ] Know `kubectl explain` for quick reference

### Common Exam Scenarios

1. **Create PV and PVC** - Most common
2. **Fix broken PVC binding** - Troubleshooting
3. **Expand existing PVC** - If `allowVolumeExpansion: true`
4. **Change reclaim policy** - Use `kubectl patch`
5. **Mount volume in Pod** - Know volumeMounts syntax

### Time Management

- Storage questions: **2-3 minutes each**
- Don't overthink - use `kubectl explain`
- If stuck, flag and move on

---

## Quick Reference Card

```
PV States:      Available → Bound → Released → Failed
PVC States:     Pending → Bound → Lost

Access Modes:   RWO (ReadWriteOnce)
                ROX (ReadOnlyMany)
                RWX (ReadWriteMany)
                RWOP (ReadWriteOncePod)

Reclaim:        Retain (keep data)
                Delete (remove all)

Binding Modes:  Immediate
                WaitForFirstConsumer
```

---

**Good luck with your CKA exam! 🎯**
