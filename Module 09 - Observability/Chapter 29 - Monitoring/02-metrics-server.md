# Metrics Server

## What is Metrics Server?

Metrics Server collects resource metrics from Kubernetes:

```
Every 15 seconds:
  Kubelet → Metrics Server
  "Pod CPU: 150m, Memory: 256Mi"
  
Metrics Server:
  Aggregates across all nodes
  Stores in memory (short-term)
  Provides via API
```

---

## Check If Installed

```bash
# Check deployment
kubectl get deployment -n kube-system metrics-server

# If not found: Install it
kubectl apply -f \
  https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Wait for rollout
kubectl rollout status deployment/metrics-server -n kube-system
```

---

## Metrics Server Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: metrics-server
        image: registry.k8s.io/metrics-server/metrics-server:v0.6.1
        args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-insecure-tls  # For self-signed certs
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
          limits:
            cpu: 100m
            memory: 200Mi
```

---

## Using Metrics

### kubectl top pods

```bash
# View pod resource usage
kubectl top pods -n default

# Output:
NAME                CPU(cores)   MEMORY(bytes)
web-app-6bf4d5c94   150m         256Mi
db-pod-7f8c9d4e2    500m         1Gi

# In production, 150m = 150 millicores = 0.15 CPU
# 256Mi = 256 megabytes
```

### kubectl top nodes

```bash
# View node resource usage
kubectl top nodes

# Output:
NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-1        800m         40%    2Gi             50%
node-2        1200m        60%    3Gi             75%
node-3        400m         20%    1Gi             25%
```

---

## Metrics for HPA

HPA needs Metrics Server to make decisions:

```yaml
# Without Metrics Server
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
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

# Status: "unknown" (waiting for metrics)
```

---

## Real-World Scenario: Debugging HPA

```bash
# HPA not scaling?

# Step 1: Check metrics availability
kubectl top pods -n default

# If "metrics not available":
#   → Metrics Server not running
#   → Deploy Metrics Server

# Step 2: Check HPA status
kubectl describe hpa web-hpa

# If "unknown" metrics:
#   → Pods don't have resource requests
#   → Add requests to deployment

# Step 3: Verify pod requests
kubectl describe pod web-app-xxx

# Resources:
#   Requests: cpu 100m, memory 128Mi
```

---

## Troubleshooting Metrics Server

### Metrics Server Won't Start

```bash
# Check logs
kubectl logs -n kube-system deployment/metrics-server

# Common errors:
# 1. "unable to fully scrape metrics"
#    → kubelet TLS issues
#    → Fix: --kubelet-insecure-tls (dev only)

# 2. "readiness probe failed"
#    → Metrics Server not ready
#    → Wait: kubectl rollout status deployment/metrics-server -n kube-system
```

### kubectl top Returns "metrics not available"

```bash
# Pod doesn't have resource requests
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
      - name: app
        image: app:v1
        # MISSING: resources section

# Fix: Add resource requests
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
```

### High Latency in Metrics Collection

```yaml
# Reduce collection interval (faster but more load)
args:
- --metric-resolution=10s  # Default 15s, reduce to 10s

# Increase caching
- --metric-resolution=30s  # Increase if overloaded
```

---

## Metrics API Query

Direct API query (for testing):

```bash
# Get pod metrics
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/default/pods | jq

# Get node metrics
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes | jq

# Output (example):
{
  "kind": "PodMetricsList",
  "items": [
    {
      "metadata": {"name": "web-app-abc123"},
      "containers": [
        {
          "name": "web",
          "usage": {
            "cpu": "150m",
            "memory": "256Mi"
          }
        }
      ]
    }
  ]
}
```

---

## Integration with HPA

Step-by-step:

```yaml
---
# Step 1: Deployment with resource requests
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: web
        image: web:v1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi

---
# Step 2: HPA references deployment
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## Best Practices

1. **Always Set Resource Requests**
   - Required for metrics calculations
   - Required for HPA/VPA/CA

2. **Monitor Metrics Server**
   - Alert if metrics unavailable
   - Check CPU/memory usage

3. **Don't Over-Reduce Metric Resolution**
   - Too frequent: Overloads cluster
   - Too sparse: Misses spikes

4. **Use for Capacity Planning**
   - View kubectl top periodically
   - Track resource trends

---

## Summary

Metrics Server:

**Purpose:**
- Collect CPU/memory metrics
- Provide data to HPA/VPA/CA
- Enable kubectl top command

**Setup:**
- Apply manifest
- Verify deployment running

**Usage:**
- kubectl top pods/nodes
- HPA/VPA automatic scaling
- Direct API queries

**Next:** File 03 covers Prometheus setup.
