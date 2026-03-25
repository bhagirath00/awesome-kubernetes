# Autoscaler Best Practices

## 1. Proper Resource Requests

HPA/VPA/CA all depend on resource requests:

```yaml
# WRONG — no requests
spec:
  containers:
  - name: app
    image: app:latest
    # Will break all autoscaling

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

Without requests:
- HPA can't calculate percentage
- VPA has no baseline
- CA can't plan capacity

---

## 2. Combine HPA + CA Properly

```yaml
---
# Deployment with right-sized pods
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3  # Ignored by HPA
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: web:latest
        resources:
          requests:
            cpu: 100m
            memory: 128Mi

---
# HPA scales replicas
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
  maxReplicas: 100
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

---
# CA scales nodes
# (configured in kube-system namespace)
```

**Result:**
- Pods scale 3-100 (HPA)
- Nodes scale to fit (CA)

---

## 3. Testing Autoscaling

```bash
# Create load test pod
kubectl run load-test --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://web-service; done"

# Watch scaling
watch -n 5 'kubectl get pods; echo "---"; kubectl get nodes'

# Monitor:
# 1. Pods pending? (HPA scaling)
# 2. New nodes? (CA scaling)
# 3. Latency acceptable? (scaling response time)

# Clean up
kubectl delete pod load-test
```

---

## 4. Set Realistic Min/Max

```yaml
# Bad: Min too low
minReplicas: 1  # Risky, no redundancy

# Good: Min for HA
minReplicas: 3  # Always 3 for high availability

# Bad: Max too high
maxReplicas: 1000  # Huge cost spike possible

# Good: Max with ceiling
maxReplicas: 100   # Cost cap at ~$10k/month
```

Calculate max based on:
- **Peak expected load:** 2-3x
- **Budget:** Max monthly spend
- **SLA:** Need for redundancy

---

## 5. Separate Scaling Concerns

```yaml
---
# Critical workload
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-hpa
spec:
  minReplicas: 5        # High baseline (HA)
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80  # Conservative

---
# Non-critical workload
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: batch-hpa
spec:
  minReplicas: 0        # Can scale to zero
  maxReplicas: 100
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

## 6. Monitor and Alert

```yaml
# Prometheus rules
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: autoscaling-alerts
spec:
  groups:
  - name: autoscaling.rules
    rules:
    - alert: PodsPending
      expr: count(kube_pod_status_phase{phase="Pending"}) > 10
      for: 5m
      annotations:
        summary: "{{ $value }} pods pending (check CA)"
    
    - alert: HighNodeUtilization
      expr: (sum(kube_node_status_capacity_cpu_cores) - sum(kube_node_status_allocatable_cpu_cores)) / sum(kube_node_status_capacity_cpu_cores) > 0.9
      annotations:
        summary: "Cluster 90% CPU utilized, consider increasing max nodes"
```

---

## 7. Graceful Drain on Scale-Down

```yaml
# Pod Disruption Budget prevents forced eviction
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web

# When CA drains node:
# Waits for minAvailable pods to move
# Graceful shutdown period respected
```

---

## 8. Handle Stateful Workloads

```yaml
# StatefulSet with graceful shutdown
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  template:
    metadata:
      labels:
        app: database
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: postgres
        image: postgres:14
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15; pg_ctl stop"]
```

CA waits for termination grace period before killing pod.

---

## 9. Cost Optimization Strategy

```
Phase 1: Baseline
  Minimum nodes: Calculate peak normal load
  Always keep running

Phase 2: HPA Scaling
  Scale replicas 2-5x for traffic spikes
  Handles 99% of use cases

Phase 3: CA Scaling
  Scale nodes only if HPA maxes out
  Last resort (expensive)

Phase 4: Spot Instances
  Use 80% spot, 20% on-demand
  Save 70% on compute costs
```

---

## 10. Prevent Autoscaling Thrashing

```yaml
# DON'T: Scale too aggressively
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    periodSeconds: 5

# DO: Stabilize before scaling
behavior:
  scaleUp:
    stabilizationWindowSeconds: 30
    policies:
    - type: Percent
      value: 50
      periodSeconds: 30
  
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
    - type: Percent
      value: 10
      periodSeconds: 60
```

---

## Troubleshooting Checklist

- [ ] Resource requests set on all containers
- [ ] HPA min < max
- [ ] CA max nodes configured
- [ ] Metrics server running
- [ ] Pods can be evicted (check PDB)
- [ ] No pods with local storage blocking eviction
- [ ] IAM/cloud provider permissions correct
- [ ] Test scaling in staging first
- [ ] Alerts configured for issues
- [ ] Cost monitoring enabled

---

## Common Scenario: Black Friday

```yaml
# Expected: 10x traffic spike
# Budget: $50k for 48 hours

# Configuration:
# Normal cluster: 10 nodes
# Peak cluster: 100 nodes
# Cost cap: nodes.max: "100"

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: bf-hpa
spec:
  minReplicas: 30    # High baseline for spike
  maxReplicas: 500   # Allow massive scale
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50  # Aggressive

# Result:
# 1. HPA scales to 500 replicas
# 2. CA adds 90 nodes
# 3. All traffic handled
# 4. Auto-removes nodes after spike
```

---

## Summary

Autoscaling best practices:

**Setup:**
- Resource requests on all containers
- HPA + CA together
- Realistic min/max

**Operation:**
- Monitor and alert
- Test before production
- Handle stateful workloads

**Optimization:**
- Spot instances for cost
- Separate critical/batch workloads
- Prevent thrashing

**Module 8 Complete!**

You have learned scheduling and autoscaling:
- Chapter 24: Pod Scheduling (5 files)
- Chapter 25: Horizontal Pod Autoscaler (4 files)
- Chapter 26: Vertical Pod Autoscaler (3 files)
- Chapter 27: Cluster Autoscaler (4 files)

**Total: 16 files covering full autoscaling strategy**
