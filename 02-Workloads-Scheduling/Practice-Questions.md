# Workloads & Scheduling - Practice Questions

> **Domain Weight:** 15% | **Estimated Questions:** 2-3

---

## Question 1: Create Deployment with Rolling Update
**Difficulty:** Easy | **Time:** 4 minutes

Create a deployment named `web-app` with image `nginx:1.20` and 3 replicas. Then update the image to `nginx:1.21` and verify the rolling update.

<details>
<summary>💡 Solution</summary>

```bash
# Create deployment
kubectl create deployment web-app --image=nginx:1.20 --replicas=3

# Update image
kubectl set image deployment/web-app nginx=nginx:1.21

# Watch rollout
kubectl rollout status deployment/web-app

# Verify
kubectl get deployment web-app
kubectl describe deployment web-app | grep Image
```
</details>

---

## Question 2: Rollback Deployment
**Difficulty:** Easy | **Time:** 3 minutes

The deployment `web-app` was updated to a broken image. Rollback to the previous working version.

<details>
<summary>💡 Solution</summary>

```bash
# Check history
kubectl rollout history deployment/web-app

# Rollback
kubectl rollout undo deployment/web-app

# Or rollback to specific revision
kubectl rollout undo deployment/web-app --to-revision=1

# Verify
kubectl rollout status deployment/web-app
```
</details>

---

## Question 3: ConfigMap as Environment Variables
**Difficulty:** Medium | **Time:** 5 minutes

Create a ConfigMap named `app-config` with:
- `APP_ENV=production`
- `LOG_LEVEL=info`

Create a pod named `config-pod` using image `nginx` that uses all keys from this ConfigMap as environment variables.

<details>
<summary>💡 Solution</summary>

```bash
# Create ConfigMap
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info

# Create Pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
  - name: nginx
    image: nginx
    envFrom:
    - configMapRef:
        name: app-config
EOF

# Verify
kubectl exec config-pod -- env | grep -E 'APP_ENV|LOG_LEVEL'
```
</details>

---

## Question 4: Secret as Volume
**Difficulty:** Medium | **Time:** 5 minutes

Create a Secret named `db-secret` with:
- `username=admin`
- `password=secret123`

Create a pod named `secret-pod` that mounts this secret at `/etc/db-creds`.

<details>
<summary>💡 Solution</summary>

```bash
# Create Secret
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123

# Create Pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/db-creds
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: db-secret
EOF

# Verify
kubectl exec secret-pod -- ls /etc/db-creds
kubectl exec secret-pod -- cat /etc/db-creds/username
```
</details>

---

## Question 5: Node Selector
**Difficulty:** Easy | **Time:** 3 minutes

Create a pod named `gpu-pod` with image `nginx` that only runs on nodes labeled `gpu=true`.

<details>
<summary>💡 Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  nodeSelector:
    gpu: "true"
  containers:
  - name: nginx
    image: nginx
```

```bash
# Label a node first (if needed)
kubectl label nodes <node-name> gpu=true

# Apply pod
kubectl apply -f gpu-pod.yaml
```
</details>

---

## Question 6: Taints and Tolerations
**Difficulty:** Medium | **Time:** 5 minutes

Node `worker-1` is tainted with `dedicated=database:NoSchedule`. Create a pod named `db-pod` with image `mysql:8` that can tolerate this taint.

<details>
<summary>💡 Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
  containers:
  - name: mysql
    image: mysql:8
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
```
</details>

---

## Question 7: Node Affinity
**Difficulty:** Medium | **Time:** 5 minutes

Create a pod named `zone-pod` with image `nginx` that:
- Must run on nodes in zone `us-east-1a` OR `us-east-1b`
- Prefers nodes with `disktype=ssd`

<details>
<summary>💡 Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: zone-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - us-east-1a
            - us-east-1b
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: nginx
    image: nginx
```
</details>

---

## Question 8: Resource Limits
**Difficulty:** Easy | **Time:** 3 minutes

Create a pod named `limited-pod` with image `nginx` that:
- Requests 100m CPU and 128Mi memory
- Limits to 200m CPU and 256Mi memory

<details>
<summary>💡 Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limited-pod
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "256Mi"
```
</details>

---

## Question 9: HPA
**Difficulty:** Medium | **Time:** 4 minutes

Create a Horizontal Pod Autoscaler for deployment `web-app` that:
- Minimum replicas: 2
- Maximum replicas: 10
- Target CPU utilization: 50%

<details>
<summary>💡 Solution</summary>

```bash
kubectl autoscale deployment web-app \
  --min=2 \
  --max=10 \
  --cpu-percent=50

# Verify
kubectl get hpa
```

Or YAML:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```
</details>

---

## Question 10: Pod Anti-Affinity
**Difficulty:** Hard | **Time:** 6 minutes

Create a deployment named `web` with 3 replicas of `nginx` image. Ensure pods are spread across different nodes using pod anti-affinity.

<details>
<summary>💡 Solution</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: web
            topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx
```
</details>

---

## Quick Reference

```bash
# Deployments
kubectl create deployment NAME --image=IMAGE --replicas=N
kubectl set image deployment/NAME CONTAINER=IMAGE
kubectl rollout status deployment/NAME
kubectl rollout undo deployment/NAME
kubectl scale deployment/NAME --replicas=N

# ConfigMaps
kubectl create configmap NAME --from-literal=key=value

# Secrets
kubectl create secret generic NAME --from-literal=key=value

# Labels/Taints
kubectl label nodes NODE key=value
kubectl taint nodes NODE key=value:effect

# HPA
kubectl autoscale deployment NAME --min=N --max=N --cpu-percent=N
```
