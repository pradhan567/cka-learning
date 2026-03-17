# ConfigMaps and Secrets

> **CKA Domain:** Workloads & Scheduling (15%)
> **Official Docs:** 
> - https://kubernetes.io/docs/concepts/configuration/configmap/
> - https://kubernetes.io/docs/concepts/configuration/secret/

---

## Table of Contents

1. [ConfigMaps](#1-configmaps)
2. [Creating ConfigMaps](#2-creating-configmaps)
3. [Using ConfigMaps](#3-using-configmaps)
4. [Secrets](#4-secrets)
5. [Creating Secrets](#5-creating-secrets)
6. [Using Secrets](#6-using-secrets)
7. [Essential Commands](#7-essential-commands)
8. [Practice Exercises](#8-practice-exercises)

---

## 1. ConfigMaps

**ConfigMaps** store non-confidential configuration data as key-value pairs.

### Use Cases

- Environment variables
- Configuration files
- Command-line arguments

---

## 2. Creating ConfigMaps

### From Literals

```bash
kubectl create configmap my-config \
  --from-literal=DB_HOST=mysql \
  --from-literal=DB_PORT=3306
```

### From File

```bash
# From single file
kubectl create configmap my-config --from-file=config.properties

# From directory
kubectl create configmap my-config --from-file=./configs/

# With custom key
kubectl create configmap my-config --from-file=app-config=config.properties
```

### From YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  DB_HOST: mysql
  DB_PORT: "3306"
  config.properties: |
    server.port=8080
    server.host=0.0.0.0
```

---

## 3. Using ConfigMaps

### As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: nginx
    env:
    # Single key
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: my-config
          key: DB_HOST
    # All keys as env vars
    envFrom:
    - configMapRef:
        name: my-config
```

### As Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: my-config
```

### Specific Keys as Files

```yaml
volumes:
- name: config-volume
  configMap:
    name: my-config
    items:
    - key: config.properties
      path: app.conf
```

---

## 4. Secrets

**Secrets** store sensitive data like passwords, tokens, and keys.

### Secret Types

| Type | Description |
|------|-------------|
| `Opaque` | Generic secret (default) |
| `kubernetes.io/tls` | TLS certificate |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/basic-auth` | Basic authentication |
| `kubernetes.io/ssh-auth` | SSH credentials |

---

## 5. Creating Secrets

### Generic Secret

```bash
# From literals
kubectl create secret generic my-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123

# From file
kubectl create secret generic my-secret --from-file=./credentials
```

### TLS Secret

```bash
kubectl create secret tls my-tls-secret \
  --cert=path/to/cert.crt \
  --key=path/to/key.key
```

### Docker Registry Secret

```bash
kubectl create secret docker-registry my-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com
```

### From YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: YWRtaW4=      # base64 encoded
  password: c2VjcmV0MTIz  # base64 encoded
```

### Encode/Decode Base64

```bash
# Encode
echo -n 'admin' | base64
# Output: YWRtaW4=

# Decode
echo 'YWRtaW4=' | base64 -d
# Output: admin
```

---

## 6. Using Secrets

### As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: password
    envFrom:
    - secretRef:
        name: my-secret
```

### As Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: my-secret
```

### Image Pull Secret

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: registry.example.com/my-app:v1
  imagePullSecrets:
  - name: my-registry-secret
```

---

## 7. Essential Commands

```bash
# ConfigMaps
kubectl create configmap NAME --from-literal=key=value
kubectl create configmap NAME --from-file=path
kubectl get configmaps
kubectl describe configmap NAME
kubectl delete configmap NAME

# Secrets
kubectl create secret generic NAME --from-literal=key=value
kubectl create secret tls NAME --cert=CERT --key=KEY
kubectl get secrets
kubectl describe secret NAME
kubectl delete secret NAME

# View secret data (decoded)
kubectl get secret NAME -o jsonpath='{.data.key}' | base64 -d
```

---

## 8. Practice Exercises

### Exercise 1: ConfigMap with Environment Variables

```bash
# Create ConfigMap
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info

# Create Pod using ConfigMap
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: nginx
    envFrom:
    - configMapRef:
        name: app-config
EOF

# Verify
kubectl exec app-pod -- env | grep -E 'APP_ENV|LOG_LEVEL'
```

### Exercise 2: Secret as Volume

```bash
# Create Secret
kubectl create secret generic db-creds \
  --from-literal=username=dbuser \
  --from-literal=password=dbpass123

# Create Pod with secret volume
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/db-creds
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: db-creds
EOF

# Verify
kubectl exec db-pod -- cat /etc/db-creds/username
kubectl exec db-pod -- cat /etc/db-creds/password
```

---

**Next:** [03-Pod-Scheduling.md](./03-Pod-Scheduling.md)
