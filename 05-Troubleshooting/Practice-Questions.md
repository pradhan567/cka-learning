# Troubleshooting - Practice Questions

> **Domain Weight:** 30% (Highest!) | **Estimated Questions:** 5-6

---

## Question 1: Node NotReady
**Difficulty:** Medium | **Time:** 5 minutes

Node `worker-2` is showing NotReady status. Debug and fix the issue.

<details>
<summary>💡 Solution</summary>

```bash
# 1. Check node status
kubectl describe node worker-2

# 2. SSH to node
ssh worker-2

# 3. Check kubelet
systemctl status kubelet
journalctl -u kubelet | tail -50

# 4. Common fixes
systemctl start kubelet
systemctl enable kubelet

# Or if container runtime issue
systemctl start containerd
systemctl restart kubelet

# 5. Verify
kubectl get nodes
```
</details>

---

## Question 2: Pod CrashLoopBackOff
**Difficulty:** Medium | **Time:** 5 minutes

Pod `web-app` in namespace `production` is in CrashLoopBackOff. Debug and identify the issue.

<details>
<summary>💡 Solution</summary>

```bash
# 1. Check pod status
kubectl get pod web-app -n production

# 2. Check events
kubectl describe pod web-app -n production

# 3. Check logs
kubectl logs web-app -n production --previous

# 4. Common issues:
# - Application error (check logs)
# - Missing ConfigMap/Secret
# - Wrong command/args
# - Failing health checks

# 5. Check container exit code
kubectl describe pod web-app -n production | grep -A5 "Last State"
```
</details>

---

## Question 3: Service Not Routing Traffic
**Difficulty:** Medium | **Time:** 5 minutes

Service `api-svc` is not routing traffic to pods. Debug and fix.

<details>
<summary>💡 Solution</summary>

```bash
# 1. Check service
kubectl get svc api-svc
kubectl describe svc api-svc

# 2. Check endpoints (KEY!)
kubectl get endpoints api-svc
# If empty, selector doesn't match

# 3. Check selector vs pod labels
kubectl get svc api-svc -o yaml | grep -A5 selector
kubectl get pods --show-labels

# 4. Fix selector
kubectl patch svc api-svc -p '{"spec":{"selector":{"app":"api"}}}'

# Or fix pod labels
kubectl label pod <pod> app=api
```
</details>

---

## Question 4: Pod Stuck Pending
**Difficulty:** Medium | **Time:** 5 minutes

Pod `database` is stuck in Pending state. Debug and fix.

<details>
<summary>💡 Solution</summary>

```bash
# 1. Describe pod
kubectl describe pod database

# 2. Check Events section for:
# - Insufficient resources
# - Node selector not matching
# - Taints not tolerated
# - PVC not bound

# 3. Check node resources
kubectl describe nodes | grep -A5 "Allocated resources"

# 4. Common fixes:
# - Remove nodeSelector if not needed
# - Add toleration
# - Create/bind PVC
# - Scale down other pods
```
</details>

---

## Question 5: Control Plane Component Down
**Difficulty:** Hard | **Time:** 8 minutes

The kube-scheduler is not running. Pods are stuck in Pending. Debug and fix.

<details>
<summary>💡 Solution</summary>

```bash
# 1. Check scheduler pod
kubectl get pods -n kube-system | grep scheduler

# 2. If not running, check manifest
cat /etc/kubernetes/manifests/kube-scheduler.yaml

# 3. Check for syntax errors or wrong paths
# Common issues:
# - Wrong image
# - Wrong certificate paths
# - Syntax errors

# 4. Check logs
kubectl logs -n kube-system kube-scheduler-<node>
# Or if pod not running
crictl ps -a | grep scheduler
crictl logs <container-id>

# 5. Fix manifest and save
# kubelet will auto-restart the pod

# 6. Verify
kubectl get pods -n kube-system | grep scheduler
```
</details>

---

## Question 6: ImagePullBackOff
**Difficulty:** Easy | **Time:** 4 minutes

Pod `app` is in ImagePullBackOff state. Debug and fix.

<details>
<summary>💡 Solution</summary>

```bash
# 1. Describe pod
kubectl describe pod app

# 2. Check Events for error message

# 3. Common fixes:
# - Correct image name
kubectl set image pod/app app=nginx:1.21

# - Add imagePullSecrets for private registry
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass

# Then add to pod spec:
# imagePullSecrets:
# - name: regcred
```
</details>

---

## Question 7: DNS Not Working
**Difficulty:** Medium | **Time:** 5 minutes

Pods cannot resolve service names. Debug DNS.

<details>
<summary>💡 Solution</summary>

```bash
# 1. Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 2. Check CoreDNS service
kubectl get svc -n kube-system kube-dns

# 3. Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# 4. Test DNS
kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes

# 5. Check if NetworkPolicy blocking DNS
kubectl get networkpolicy -A

# 6. Common fixes:
# - Restart CoreDNS
kubectl rollout restart deployment coredns -n kube-system

# - Check CoreDNS ConfigMap
kubectl get configmap coredns -n kube-system -o yaml
```
</details>

---

## Question 8: etcd Backup and Restore
**Difficulty:** Hard | **Time:** 8 minutes

Create a backup of etcd to `/opt/backup/etcd.db`. Then restore from backup.

<details>
<summary>💡 Solution</summary>

```bash
# Backup
ETCDCTL_API=3 etcdctl snapshot save /opt/backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /opt/backup/etcd.db --write-out=table

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /opt/backup/etcd.db \
  --data-dir=/var/lib/etcd-restored

# Update etcd manifest
vi /etc/kubernetes/manifests/etcd.yaml
# Change --data-dir and volume hostPath to /var/lib/etcd-restored

# Wait for etcd to restart
kubectl get pods -n kube-system | grep etcd
```
</details>

---

## Question 9: kubelet Not Running
**Difficulty:** Medium | **Time:** 5 minutes

Node `worker-1` kubelet is not running. Debug and fix.

<details>
<summary>💡 Solution</summary>

```bash
# SSH to node
ssh worker-1

# Check kubelet status
systemctl status kubelet

# Check logs
journalctl -u kubelet | tail -100

# Common issues:
# - Wrong config
# - Certificate issues
# - Container runtime not running

# Fix
systemctl start containerd
systemctl start kubelet
systemctl enable kubelet

# Verify
systemctl status kubelet
```
</details>

---

## Question 10: Application Logs
**Difficulty:** Easy | **Time:** 3 minutes

Get the last 50 lines of logs from pod `web` container `nginx`. Also get logs from the previous container instance.

<details>
<summary>💡 Solution</summary>

```bash
# Last 50 lines
kubectl logs web -c nginx --tail=50

# Previous container
kubectl logs web -c nginx --previous

# Combined
kubectl logs web -c nginx --tail=50 --previous
```
</details>

---

## Question 11: Resource Monitoring
**Difficulty:** Easy | **Time:** 3 minutes

Check resource usage of all nodes and pods in namespace `production`.

<details>
<summary>💡 Solution</summary>

```bash
# Node resources
kubectl top nodes

# Pod resources in namespace
kubectl top pods -n production

# Sort by CPU
kubectl top pods -n production --sort-by=cpu

# Sort by memory
kubectl top pods -n production --sort-by=memory
```
</details>

---

## Question 12: Certificate Expiration
**Difficulty:** Medium | **Time:** 4 minutes

Check certificate expiration dates and renew if needed.

<details>
<summary>💡 Solution</summary>

```bash
# Check expiration
kubeadm certs check-expiration

# Renew all certificates
kubeadm certs renew all

# Renew specific certificate
kubeadm certs renew apiserver

# After renewal, restart control plane
# (kubelet auto-restarts static pods)
```
</details>

---

## Quick Reference

```bash
# Pod debugging
kubectl get pod <pod> -o wide
kubectl describe pod <pod>
kubectl logs <pod> [--previous] [-c container] [--tail=N]
kubectl exec -it <pod> -- /bin/sh

# Events
kubectl get events --sort-by='.lastTimestamp'

# Service debugging
kubectl get endpoints <svc>
kubectl describe svc <svc>

# Node debugging
kubectl describe node <node>
ssh <node> "systemctl status kubelet"
ssh <node> "journalctl -u kubelet"

# Control plane
kubectl get pods -n kube-system
kubectl logs -n kube-system <pod>
crictl ps
crictl logs <container-id>

# etcd
ETCDCTL_API=3 etcdctl snapshot save FILE \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Resources
kubectl top nodes
kubectl top pods

# Certificates
kubeadm certs check-expiration
kubeadm certs renew all
```

---

## Troubleshooting Checklist

### Pod Issues
- [ ] `kubectl get pod` - Check status
- [ ] `kubectl describe pod` - Check events
- [ ] `kubectl logs` - Check application logs
- [ ] `kubectl logs --previous` - Check crashed container logs

### Service Issues
- [ ] `kubectl get endpoints` - Check if endpoints exist
- [ ] Compare selector with pod labels
- [ ] Test connectivity from within cluster

### Node Issues
- [ ] `kubectl describe node` - Check conditions
- [ ] `systemctl status kubelet` - Check kubelet
- [ ] `journalctl -u kubelet` - Check kubelet logs
- [ ] `systemctl status containerd` - Check runtime

### Control Plane Issues
- [ ] `kubectl get pods -n kube-system` - Check components
- [ ] Check `/etc/kubernetes/manifests/` - Static pod manifests
- [ ] `crictl ps` - If kubectl not working
- [ ] `crictl logs` - Container logs
