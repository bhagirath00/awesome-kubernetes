# Configuring HPA

## Basic CPU Scaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
  namespace: default
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
        averageUtilization: 70
```

When CPU > 70%, scale up. When CPU < 70%, scale down.

---

## CPU + Memory Scaling

Scale on BOTH CPU and memory:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  
  minReplicas: 3
  maxReplicas: 20
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

Scales if EITHER metric exceeds threshold.

---

## Scaling Behavior

Control how fast to scale:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: controlled-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  
  minReplicas: 1
  maxReplicas: 50
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100      # Double replicas
        periodSeconds: 30
      - type: Pods
        value: 5        # Or add 5 pods
        periodSeconds: 30
      selectPolicy: Max # Use whichever is larger
    
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50       # Remove half
        periodSeconds: 60
      selectPolicy: Min # Use whichever is smaller
```

**Scale-Up:** Double replicas or add 5 (whichever is larger) every 30 seconds.

**Scale-Down:** Remove 50% but wait 5 minutes between checks.

---

## Real-World Example: E-Commerce

```yaml
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecommerce-api
spec:
  replicas: 3  # HPA will override this
  selector:
    matchLabels:
      app: ecommerce
  template:
    metadata:
      labels:
        app: ecommerce
    spec:
      containers:
      - name: api
        image: ecommerce:v1
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi

---
# HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ecommerce-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ecommerce-api
  
  minReplicas: 3      # Always keep 3 (high availability)
  maxReplicas: 50     # Don't waste money on 100s
  
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
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 50       # Scale up gradually (50%)
        periodSeconds: 15
      selectPolicy: Max
    
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 2        # Remove 2 pods at a time
        periodSeconds: 60
```

---

## Advanced: Absolute Resource Values

Scale based on raw CPU amount:

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: AverageValue
      averageValue: 100m  # Scale if CPU > 100m
```

---

## Troubleshooting HPA

### HPA Not Scaling

```bash
# Check HPA status
kubectl describe hpa web-hpa

# Common reason: No metrics
"unknown" metrics

# Fix: Wait for metrics server to collect data
# Check metrics server
kubectl get deployment metrics-server -n kube-system
```

### Check Current Metrics

```bash
# View current CPU/memory
kubectl top nodes
kubectl top pods -n default

# If empty: metrics server not ready (wait 1-2 minutes)
```

### Verify Requests are Set

```bash
# Check if pod has requests
kubectl get pods -o yaml | grep resources
resources:
  requests:
    cpu: 100m

# If missing: HPA can't work
```

---

## Common Configurations

### Configuration 1: Aggressive Scaling (Auto-scale traffic)

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 100        # Double
      periodSeconds: 10 # Every 10 seconds
```

For sudden traffic spikes.

### Configuration 2: Conservative Scaling (Save resources)

```yaml
minReplicas: 1          # Scale down aggressively
maxReplicas: 5          # Cap at 5

behavior:
  scaleDown:
    stabilizationWindowSeconds: 600  # Wait 10 min
    policies:
    - type: Percent
      value: 10         # Remove 10%
      periodSeconds: 60
```

For predictable, gradual load changes.

### Configuration 3: Batch Processing (Efficient)

```yaml
minReplicas: 0          # Can scale to zero
maxReplicas: 100

behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 200        # Triple
      periodSeconds: 5
```

Aggressive scale-up for burst jobs.

---

## Summary

HPA configuration:

**Basic:**
- Target metric (CPU, memory)
- Target percentage (70%)
- Min/max replicas

**Advanced:**
- Scale-up behavior (how fast)
- Scale-down behavior (stabilization)
- Multiple metrics (scale on any)

**Next:** File 03 covers custom metrics for specialized scaling.
