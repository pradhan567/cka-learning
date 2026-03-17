# Application Troubleshooting

> **CKA Domain:** Troubleshooting (30%)
> **Official Docs:** https://kubernetes.io/docs/tasks/debug/debug-application/

---

## Table of Contents

1. [Pod Troubleshooting](#1-pod-troubleshooting)
2. [Container Logs](#2-container-logs)
3. [Common Pod States](#3-common-pod-states)
4. [Debugging Running Pods](#4-debugging-running-pods)
5. [Service Troubleshooting](#5-service-troubleshooting)
6. [Resource Issues](#6-resource-issues)
7. [Essential Commands](#7-essential-commands)
8. [Practice Scenarios](#8-practice-scenarios)

---

## 1. Pod Troubleshooting

### Debug Flow

```
Pod Issue
    │
    ├── Check pod status: kubectl get pod
    │
    ├── Check pod details: kubectl describe pod
    │
    ├── Check logs: kubectl logs
    │
    ├── Check events: kubectl get events
    │
    └── Exec into pod: kubectl exec
```

### Quick Pod Debug

```bash
# Get pod status
kubectl get pod <pod-name> -o wide

# Describe pod (shows events, conditions)
kubectl describe pod <pod-name>

# Get pod YAML
kubectl get pod <pod-name> -o yaml

# Watch pods
kubectl get pods -w
```

---

## 2. Container Logs

### View Logs

```bash
# Current logs
kubectl logs <pod-name>

# Previous container logs (after restart)
kubectl logs <pod-name> --previous

# Follow logs
kubectl logs <pod-name> -f

# Last N lines
kubectl logs <pod-name> --tail=100

# Since time
kubectl logs <pod-name> --since=1h

# Multi-container pod
kubectl logs <pod-name> -c <container-name>

# All containers
kubectl logs <pod-name> --all-containers=true
```

### Log Locations on Node

```bash
# Container logs on node
/var/log/containers/
/var/log/pods/

# kubelet logs
journalctl -u kubelet
```

---

## 3. Common Pod States

### Pod Status Reference

| Status | Meaning | Debug |
|--------|---------|-------|
| `Pending` | Not scheduled yet | Check events, resources |
| `ContainerCreating` | Pulling image, mounting volumes | Check events |
| `Running` | Container running | Check logs if issues |
| `Completed` | Container exited successfully | Normal for Jobs |
| `CrashLoopBackOff` | Container crashing repeatedly | Check logs |
| `ImagePullBackOff` | Can't pull image | Check image name, registry |
| `ErrImagePull` | Image pull failed | Check image, credentials |
| `Error` | Container error | Check logs |
| `Terminating` | Being deleted | Check finalizers |

### Pending Pod

```bash
# Check why pending
kubectl describe pod <pod>

# Common causes:
# - Insufficient resources
# - Node selector/affinity not matching
# - Taints not tolerated
# - PVC not bound

# Check node resources
kubectl describe nodes | grep -A5 "Allocated resources"
```

### CrashLoopBackOff

```bash
# Check logs
kubectl logs <pod> --previous

# Common causes:
# - Application error
# - Missing config/secrets
# - Wrong command/args
# - Health check failing

# Check container exit code
kubectl describe pod <pod> | grep -A5 "Last State"
```

### ImagePullBackOff

```bash
# Check events
kubectl describe pod <pod> | grep -A10 Events

# Common causes:
# - Wrong image name
# - Image doesn't exist
# - Private registry without credentials
# - Network issues

# Fix: Check image name or add imagePullSecrets
```

---

## 4. Debugging Running Pods

### Exec into Pod

```bash
# Interactive shell
kubectl exec -it <pod> -- /bin/sh
kubectl exec -it <pod> -- /bin/bash

# Run command
kubectl exec <pod> -- ls /app
kubectl exec <pod> -- cat /etc/config

# Multi-container pod
kubectl exec -it <pod> -c <container> -- /bin/sh
```

### Debug with Ephemeral Container

```bash
# Add debug container to running pod
kubectl debug <pod> -it --image=busybox --target=<container>

# Debug with copy
kubectl debug <pod> -it --image=busybox --copy-to=debug-pod
```

### Port Forward

```bash
# Forward local port to pod
kubectl port-forward <pod> 8080:80

# Forward to service
kubectl port-forward svc/<service> 8080:80
```

---

## 5. Service Troubleshooting

### Service Not Working

```bash
# Check service
kubectl get svc <service>
kubectl describe svc <service>

# Check endpoints (most important!)
kubectl get endpoints <service>

# If endpoints empty:
# - Selector doesn't match pod labels
# - Pods not ready
# - Pods in wrong namespace
```

### Debug Service

```bash
# 1. Check service exists
kubectl get svc <service>

# 2. Check endpoints
kubectl get ep <service>

# 3. Check pod labels match selector
kubectl get svc <service> -o yaml | grep -A5 selector
kubectl get pods --show-labels

# 4. Test from within cluster
kubectl run test --image=busybox --rm -it --restart=Never -- wget -qO- http://<service>:<port>

# 5. Check if pods are ready
kubectl get pods -l <selector>
```

### Common Service Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| No endpoints | Selector mismatch | Fix selector or pod labels |
| Connection refused | Wrong port | Check targetPort |
| Timeout | Network policy | Check NetworkPolicy |
| DNS not resolving | CoreDNS issue | Check CoreDNS pods |

---

## 6. Resource Issues

### Check Resource Usage

```bash
# Node resources
kubectl top nodes

# Pod resources
kubectl top pods

# Describe node for allocations
kubectl describe node <node> | grep -A10 "Allocated resources"
```

### Resource Limits Issues

```bash
# Pod OOMKilled
kubectl describe pod <pod> | grep -i oom
# Fix: Increase memory limit

# Pod evicted
kubectl describe pod <pod> | grep -i evict
# Fix: Check node resources, increase limits

# CPU throttling
# Check if limits too low
kubectl describe pod <pod> | grep -A5 Limits
```

### LimitRange and ResourceQuota

```bash
# Check LimitRange
kubectl get limitrange -n <namespace>
kubectl describe limitrange -n <namespace>

# Check ResourceQuota
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota -n <namespace>
```

---

## 7. Essential Commands

```bash
# Pod debugging
kubectl get pod <pod> -o wide
kubectl describe pod <pod>
kubectl logs <pod> [--previous] [-c container]
kubectl exec -it <pod> -- /bin/sh

# Events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events --field-selector involvedObject.name=<pod>

# Service debugging
kubectl get svc <svc>
kubectl get endpoints <svc>
kubectl describe svc <svc>

# Resource usage
kubectl top nodes
kubectl top pods

# Quick connectivity test
kubectl run test --image=busybox --rm -it --restart=Never -- wget -qO- http://<svc>
```

---

## 8. Practice Scenarios

### Scenario 1: Pod CrashLoopBackOff

```bash
# Debug
kubectl get pod <pod>
kubectl describe pod <pod>
kubectl logs <pod> --previous

# Common fixes:
# - Fix application code
# - Add missing ConfigMap/Secret
# - Fix command/args
# - Adjust health checks
```

### Scenario 2: Pod Stuck Pending

```bash
# Debug
kubectl describe pod <pod>
# Look at Events section

# Common fixes:
# - Add more nodes/resources
# - Fix node selector
# - Add toleration for taints
# - Bind PVC
```

### Scenario 3: Service No Endpoints

```bash
# Debug
kubectl get endpoints <svc>
kubectl get svc <svc> -o yaml | grep -A5 selector
kubectl get pods --show-labels

# Fix: Match selector to pod labels
kubectl patch svc <svc> -p '{"spec":{"selector":{"app":"correct-label"}}}'
```

### Scenario 4: ImagePullBackOff

```bash
# Debug
kubectl describe pod <pod> | grep -A10 Events

# Fixes:
# - Correct image name
# - Add imagePullSecrets for private registry
kubectl create secret docker-registry regcred \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<pass>
```

---

**Next:** [Practice-Questions.md](./Practice-Questions.md)
