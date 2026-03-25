# VPA vs. HPA

## Quick Comparison

| Feature | HPA | VPA |
|---------|-----|-----|
| What scales | Number of replicas | Pod resources (CPU/memory) |
| Use when | Variable load, concurrent traffic | Optimize resources, single pods |
| Pod restart | No | Yes (Recreate/Auto modes) |
| Downtime | No | Yes (pod recreates) |
| Together | Can combine carefully | Can combine carefully |

---

## Example Scenario: Web App

### Current Setup

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: web:latest
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
```

App poorly sized (too high requests).

---

## Problem 1: High Load, Wasting Resources

**Traffic:** Normal (100 req/sec)
**Actual usage:** CPU 100m, Memory 200Mi
**Requests:** CPU 500m, Memory 512Mi
**Wasted:** 80% of resources

### Solution A: Use VPA

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Auto"
```

VPA sizes each pod correctly:
```
Requests: CPU 150m, Memory 300Mi
Result: Same traffic handled, 3x less resources
```

---

## Problem 2: Traffic Spikes

**Traffic:** 1000 req/sec (10x)
**Problem:** Even with right-sized pods, 3 replicas can't handle it

### Solution B: Use HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 30
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

HPA scales to 30 replicas:
```
Replicas: 3 → 30
Each pod: Still 150m CPU (thanks to VPA)
Total: Can handle spike
```

---

## Problem 3: Both Issues Together

**Pods overprovisioned:** Wasting resources
**Traffic spikes:** Can't handle demand

### Solution C: Use Both VPA + HPA

```yaml
---
# VPA: Right-size each pod
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Auto"

---
# HPA: Scale replicas for load
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 30
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**Result:**
- Each pod right-sized (VPA): 150m CPU, 300Mi memory
- More pods if needed (HPA): 3-30 replicas
- Handles spikes efficiently
- Saves money on baseline

---

## When to Use Each

### Use Only HPA

```
✓ Traffic varies (spiky load)
✓ Pods are already well-sized
✓ No pod restarts acceptable
✗ Pods overprovisioned
```

### Use Only VPA

```
✓ Single-instance pods
✓ Long-running services
✓ Pod recreation acceptable
✓ Pods poorly sized
✗ High traffic variability (need HPA too)
```

### Use Both

```
✓ Variable traffic
✓ Poorly sized pods
✓ Cost optimization important
✓ Can tolerate pod recreation for scaling
✗ Real-time apps (downtime not acceptable)
```

---

## Conflict Scenarios

### Scenario 1: VPA Conflicts with HPA

```yaml
# WRONG — they fight
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  minReplicas: 1   # HPA controls replicas
  maxReplicas: 50
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

---
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  updatePolicy:
    updateMode: "Auto"  # VPA recreates pods
    
  resourcePolicy:
    containerPolicies:
    - containerName: "app"
      controlledValues: RequestsAndLimits
```

Possible conflict: VPA recreates pod while HPA is scaling.

**Solution:** Set VPA mode to "Off" until HPA stabilizes.

---

## Real-World Recommendation

For most applications:

1. **Phase 1:** Use HPA only (handle variable load)
2. **Phase 2:** Add VPA in "Off" mode (see recommendations)
3. **Phase 3:** Apply VPA recommendations manually
4. **Phase 4:** Enable VPA with caution (careful monitoring)

---

## Summary

**HPA:** Scales replicas (handles concurrent users)
**VPA:** Scales resources (optimizes efficiency)

**Use together:** Powerful but requires careful setup

**HPA first:** Essential for variable load
**VPA second:** Optional but saves money

**Avoid conflicts:** VPA in "Off" mode initially, manual application

**Next:** Chapter 27 covers Cluster Autoscaler — adding nodes to the cluster.
