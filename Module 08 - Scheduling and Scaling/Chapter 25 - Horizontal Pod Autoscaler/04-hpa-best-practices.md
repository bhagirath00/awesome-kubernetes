# HPA Best Practices

## 1. Always Set Resource Requests

```yaml
# WRONG
spec:
  containers:
  - name: app
    image: app:latest

# RIGHT
spec:
  containers:
  - name: app
    image: app:latest
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

HPA calculates percentage based on requests.

---

## 2. Set Reasonable Min/Max

```yaml
# WRONG — wasteful
minReplicas: 1
maxReplicas: 1000

# RIGHT — balanced
minReplicas: 2    # High availability
maxReplicas: 50   # Cost control
```

---

## 3. Avoid Thrashing

```yaml
# WRONG — scales constantly
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 50  # Too aggressive

behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 100
      periodSeconds: 5  # Every 5 seconds

# RIGHT — stable
averageUtilization: 70  # Higher threshold

behavior:
  scaleUp:
    stabilizationWindowSeconds: 30
    policies:
    - type: Percent
      value: 50
      periodSeconds: 30
```

---

## 4. Multiple Metrics

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 70

- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 80
```

Scales on EITHER metric exceeding threshold.

---

## 5. Gradual Scale-Down

```yaml
# WRONG — scales down too fast
behavior:
  scaleDown:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 100
      periodSeconds: 10

# RIGHT — conservative
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300  # Wait 5 minutes
    policies:
    - type: Percent
      value: 10                        # Remove 10%
      periodSeconds: 60                # Every minute
```

Removes load gradually to avoid cascading failures.

---

## 6. Handle Scale-to-Zero Carefully

```yaml
# Can scale to zero (for batch jobs)
minReplicas: 0
maxReplicas: 100

# But ensures traffic can wait
metrics:
- type: Pods
  pods:
    metric:
      name: queue_length
    target:
      type: AverageValue
      averageValue: "10"
```

For batch processing, not live traffic.

---

## 7. Monitor HPA Activity

```bash
# Watch HPA in real-time
kubectl get hpa -w

# Check events
kubectl describe hpa app-hpa

# View metrics
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/cpu" | jq
```

---

## 8. Test Before Production

```yaml
# Test in staging with same config
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: staging-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: staging-app
  
  minReplicas: 1
  maxReplicas: 5  # Limit for testing
  
  # Use same thresholds as production
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## 9. Different HPAs for Different Purposes

```yaml
---
# API service — responsive scaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  
  minReplicas: 3
  maxReplicas: 100
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60

---
# Batch job — cost-optimized scaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: batch-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: batch
  
  minReplicas: 0
  maxReplicas: 50
  
  metrics:
  - type: Pods
    pods:
      metric:
        name: queue_length
      target:
        type: AverageValue
        averageValue: "50"
```

---

## 10. Account for Startup Time

```yaml
# Pod takes 30 seconds to warm up
# Give HPA stabilization window
behavior:
  scaleUp:
    stabilizationWindowSeconds: 30
    policies:
    - type: Percent
      value: 50
      periodSeconds: 30
```

Otherwise HPA scales before pod is ready.

---

## Common Scenarios

### Scenario 1: Daily Traffic Pattern

```yaml
# Traffic peaks at 9 AM, low at 3 AM
minReplicas: 5
maxReplicas: 50

# Generous stabilization prevents thrashing
behavior:
  scaleDown:
    stabilizationWindowSeconds: 600  # 10 minutes
```

---

### Scenario 2: Unpredictable Spikes

```yaml
# E-commerce flash sale
minReplicas: 10  # Higher baseline
maxReplicas: 200

behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 100  # Double aggressively
      periodSeconds: 10
```

---

### Scenario 3: Long-Running Jobs

```yaml
# Background processing
minReplicas: 1
maxReplicas: 20

metrics:
- type: Pods
  pods:
    metric:
      name: job_queue_length
    target:
      type: AverageValue
      averageValue: "100"  # Allow batch accumulation
```

---

## Troubleshooting Checklist

- [ ] Resource requests set on all containers
- [ ] Metrics server running (`kubectl get deployment metrics-server -n kube-system`)
- [ ] HPA can see metrics (`kubectl describe hpa`)
- [ ] Min < Max replicas
- [ ] Reasonable scale targets (not 99% CPU)
- [ ] Stabilization windows prevent thrashing
- [ ] Custom metrics adapter working (if using custom)
- [ ] Pod startup time accounted for
- [ ] Tested in staging first

---

## Summary

HPA best practices ensure stable, efficient auto-scaling:

**Do:**
- Set resource requests
- Use reasonable min/max
- Scale gradually
- Monitor HPA activity
- Test in staging

**Don't:**
- Scale too aggressively
- Use zero CPU target
- Forget stabilization windows
- Use HPA without resource requests

**Next:** Chapter 26 covers Vertical Pod Autoscaler (VPA) — scaling resources instead of replicas.
