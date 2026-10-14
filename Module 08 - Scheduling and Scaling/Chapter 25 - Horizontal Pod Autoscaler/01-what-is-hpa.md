# What is HPA?

## Introduction

Your app is running smoothly on 3 replicas. Then suddenly, traffic spikes 10x.

Your options:
1. Manually add more replicas (slow, error-prone)
2. Have the cluster do it automatically (smart!)

**Horizontal Pod Autoscaler (HPA)** automatically scales your pods based on demand.

---

## How HPA Works

```
1. Metrics are collected (CPU, memory, or custom metrics)
   ↓
2. HPA checks target (e.g., "keep CPU at 70%")
   ↓
3. If actual > target: Scale up (add replicas)
   If actual < target: Scale down (remove replicas)
   ↓
4. Controller adjusts replicas
   ↓
5. Scheduler places new pods on nodes
```

---

## Horizontal vs. Vertical Scaling

**Horizontal Scaling (HPA):**
```
Add more copies of the app
3 replicas → 10 replicas
Scales across multiple nodes
Handles many concurrent requests
```

**Vertical Scaling (VPA):**
```
Give more resources to existing pod
1 pod with 2CPU → 1 pod with 8CPU
Scales up a single instance
Better for single-threaded apps
```

---

## Metrics Used

HPA can scale based on:

### 1. CPU Usage

```yaml
# Scale if CPU exceeds 70%
targetCPUUtilizationPercentage: 70
```

### 2. Memory Usage

```yaml
# Scale if memory exceeds 80%
targetMemoryUtilizationPercentage: 80
```

### 3. Custom Metrics

```yaml
# Scale based on HTTP requests per second
# Scale based on queue length
# Scale based on anything you define
```

---

## Basic Example

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
  
  minReplicas: 2        # Never scale below 2
  maxReplicas: 10       # Never scale above 10
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale when CPU > 70%
```

---

## HPA Workflow

```
Time: 0:00 — Traffic is normal
  HPA: CPU is 30%, target is 70%
  Action: Nothing (keep 3 replicas)

Time: 0:05 — Traffic spikes
  HPA: CPU is 85%, target is 70%
  Action: Scale up to 5 replicas

Time: 0:10 — New pods started
  HPA: CPU is now 60%, target is 70%
  Action: Wait (stabilize before more scaling)

Time: 1:00 — Traffic returns to normal
  HPA: CPU is 20%, target is 70%
  Action: Evaluate scaling down

Time: 3:00 — Confirmed low load
  HPA: Scale down to 2 replicas (minimum)
```

---

## Why Use HPA?

### Problem Without HPA

```
Traffic spike at 2 AM
- Apps are overloaded
- Requests timeout
- Users see errors
- Manual intervention required
- Takes 10 minutes to add replicas
```

### Solution With HPA

```
Traffic spike at 2 AM
- HPA detects high CPU
- Automatically scales up
- Load distributed
- Requests complete
- No manual intervention
- 30 seconds to scale
```

---

## HPA Requirements

HPA needs:

1. **Metrics Server** (collects metrics)
   ```bash
   kubectl get deployment metrics-server -n kube-system
   ```

2. **Resource Requests** (on containers)
   ```yaml
   resources:
     requests:
       cpu: 100m
       memory: 128Mi
   ```

   Without requests, HPA can't calculate percentage.

3. **Deployment/StatefulSet/etc.** (thing to scale)
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   # HPA scales this
   ```

---

## Scaling Behavior

### Scale-Up (Add Replicas)

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 100
      periodSeconds: 15
    - type: Pods
      value: 4
      periodSeconds: 15
```

Scales up by 100% (double) or 4 pods, every 15 seconds. Whichever is larger.

### Scale-Down (Remove Replicas)

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
    - type: Percent
      value: 50
      periodSeconds: 15
```

Scales down by 50%, but waits 5 minutes between checks (stabilization).

---

## Common Mistakes

### Mistake 1: No Resource Requests

```yaml
# WRONG
spec:
  containers:
  - name: app
    image: app:latest
    # No requests!

# HPA sees 0% (can't calculate)
# Result: Never scales
```

### Mistake 2: Min = Max

```yaml
# WRONG — can't scale
minReplicas: 3
maxReplicas: 3

# Nothing scales
```

### Mistake 3: Too Aggressive Scaling

```yaml
# WRONG — scales up too fast
behavior:
  scaleUp:
    policies:
    - type: Percent
      value: 500  # +500% every 5 seconds
      periodSeconds: 5
# Creates thrashing and instability
```

---

## Viewing HPA Status

```bash
# List HPAs
kubectl get hpa

# Watch HPA
kubectl get hpa -w
NAME           REFERENCE        TARGETS   MINPODS  MAXPODS  REPLICAS  AGE
web-app-hpa    Deployment/web   60%/70%   2        10       6         5m

# Describe HPA details
kubectl describe hpa web-app-hpa
```

---

## Summary

HPA automatically scales pods:

**How it works:**
1. Measure metrics (CPU, memory, custom)
2. Compare to target
3. Scale up if too high
4. Scale down if too low

**Requirements:**
- Metrics Server
- Resource requests on pods
- Deployment/StatefulSet/etc.

**Use cases:**
- Traffic spikes
- Batch job scaling
- Cost optimization

**Next:** File 02 covers configuring HPA — practical examples.
