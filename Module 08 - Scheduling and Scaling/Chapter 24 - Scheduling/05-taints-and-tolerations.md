# Taints and Tolerations

## Introduction

Imagine a node with a GPU. You don't want regular workloads using it. You want to reserve it for GPU jobs only.

**Taints** say: "I have special requirements. Only certain pods should run on me."

**Tolerations** say: "I can handle that requirement. I'm OK running on that node."

---

## How It Works

```
Node: "I'm tainted with gpu=true. Regular pods can't run here."

Pod (normal): "I have no toleration. I can't run on tainted node."
  → Scheduler skips this node

Pod (GPU): "I tolerate gpu=true. I can run on tainted node."
  → Scheduler allows this node
```

---

## Taint Structure

```yaml
node-name:key=value:effect
```

Three parts:

1. **key** — what does the node require?
2. **value** — specificity
3. **effect** — what happens if pod doesn't tolerate?

---

## Taint Effects

```yaml
# NoSchedule (hard)
# New pods cannot be scheduled on this node
# Already running pods are not affected
key=value:NoSchedule

# NoExecute (hard + evict)
# New pods cannot be scheduled AND running pods are evicted
key=value:NoExecute

# PreferNoSchedule (soft)
# Scheduler prefers other nodes, but will use this if needed
key=value:PreferNoSchedule
```

---

## Adding Taints

```bash
# Taint a node
kubectl taint nodes gpu-node1 gpu=true:NoSchedule

# Add another taint to same node
kubectl taint nodes gpu-node1 memory=high:NoSchedule

# Remove a taint
kubectl taint nodes gpu-node1 gpu=true:NoSchedule-

# View taints
kubectl describe node gpu-node1
Taints:
  gpu=true:NoSchedule
  memory=high:NoSchedule
```

---

## Adding Tolerations

Pod must tolerate each taint:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-job
spec:
  # Tolerate the gpu taint
  tolerations:
  - key: gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  
  # Tolerate the memory taint
  - key: memory
    operator: Equal
    value: high
    effect: NoSchedule
  
  containers:
  - name: trainer
    image: tensorflow:latest
    resources:
      limits:
        nvidia.com/gpu: 1
```

Now pod can run on gpu-node1.

---

## Real-World Examples

### Example 1: GPU Node

```bash
# Taint GPU node
kubectl taint nodes gpu-node1 accelerator=nvidia:NoSchedule
```

```yaml
# GPU workload tolerates taint
apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  tolerations:
  - key: accelerator
    operator: Equal
    value: nvidia
    effect: NoSchedule
  
  containers:
  - name: training
    image: tensorflow:latest
    resources:
      limits:
        nvidia.com/gpu: 1
```

Only GPU workloads run on GPU nodes.

### Example 2: High-Memory Node

```bash
# Reserve high-memory node
kubectl taint nodes memory-node1 memory=high:NoSchedule
```

```yaml
# Memory-intensive workload
apiVersion: v1
kind: Pod
metadata:
  name: big-data
spec:
  tolerations:
  - key: memory
    operator: Equal
    value: high
    effect: NoSchedule
  
  resources:
    requests:
      memory: 120Gi
  
  containers:
  - name: processor
    image: big-data:latest
```

### Example 3: Dedicated Node Pool

```bash
# Dedicate nodes for production only
kubectl taint nodes prod-node1 dedicated=production:NoExecute
kubectl taint nodes prod-node2 dedicated=production:NoExecute
```

```yaml
# Production pod can run here
apiVersion: v1
kind: Pod
metadata:
  name: prod-api
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: production
    effect: NoExecute
  
  containers:
  - name: api
    image: api:v1
```

Dev pods (no toleration) cannot run on prod nodes.

### Example 4: NoExecute with Tolerations Seconds

```bash
# Taint with NoExecute (evicts running pods)
kubectl taint nodes quarantine-node1 maintenance=true:NoExecute
```

```yaml
# Pod can tolerate brief maintenance
apiVersion: v1
kind: Pod
metadata:
  name: tolerant-app
spec:
  tolerations:
  - key: maintenance
    operator: Equal
    value: "true"
    effect: NoExecute
    tolerationSeconds: 3600  # Tolerate for 1 hour, then evict
  
  containers:
  - name: app
    image: app:latest
```

Pod stays for 1 hour during maintenance, then evicted.

---

## Toleration Operators

```yaml
# Operator: Equal (exact match)
operator: Equal
value: "true"

# Operator: Exists (any value)
operator: Exists
# Pod tolerates taint regardless of value

# Example:
tolerations:
- key: gpu
  operator: Exists
  effect: NoSchedule
# Tolerates: gpu=true, gpu=nvidia, gpu=anything

# Operator: Wildcard taint key
tolerations:
- key: ""
  operator: Exists
# Tolerates ANY taint (use carefully!)
```

---

## Node Lifecycle Taint

Kubernetes automatically taints nodes during maintenance:

```yaml
# Node is not ready
node.kubernetes.io/not-ready:NoExecute

# Node is unreachable
node.kubernetes.io/unreachable:NoExecute

# Pod can tolerate with tolerationSeconds
tolerations:
- key: node.kubernetes.io/not-ready
  operator: Exists
  effect: NoExecute
  tolerationSeconds: 300  # Wait 5 min before evicting
```

---

## Common Patterns

### Pattern 1: Batch Jobs on Spot Instances

```bash
# Mark spot nodes
kubectl taint nodes spot-node1 instance=spot:NoSchedule
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  template:
    spec:
      tolerations:
      - key: instance
        operator: Equal
        value: spot
        effect: NoSchedule
      
      containers:
      - name: processor
        image: batch:latest
```

### Pattern 2: Upgrade/Maintenance Window

```bash
# Mark node for upgrade
kubectl taint nodes upgrade-node1 upgrade=scheduled:NoExecute
```

Pods with toleration stay, others are evicted and rescheduled.

### Pattern 3: Cost Optimization

```bash
# High-cost on-demand
kubectl taint nodes on-demand-node1 cost=high:PreferNoSchedule

# Low-cost spot
kubectl taint nodes spot-node1 cost=low:PreferNoSchedule
```

Workloads choose based on cost tolerance.

---

## Taints + Node Affinity

Combine for powerful control:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: controlled-placement
spec:
  # MUST be on production node
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: env
            operator: In
            values: [production]
  
  # TOLERATE production taint
  tolerations:
  - key: dedicated
    operator: Equal
    value: production
    effect: NoSchedule
  
  containers:
  - name: app
    image: app:latest
```

Pod goes to production node AND can run despite taint.

---

## Mistakes and Gotchas

### Mistake 1: Missing Toleration

```yaml
# WRONG — taint exists but pod has no toleration
Node: taint gpu=true:NoSchedule
Pod: no tolerations

# Result: Pod pending forever (can't be scheduled)
```

### Mistake 2: NoExecute Without Grace Period

```yaml
# WRONG — running pod suddenly evicted
kubectl taint nodes node1 urgent=true:NoExecute

# All pods evicted immediately (no tolerationSeconds)

# Better: give grace period
tolerations:
- key: urgent
  operator: Equal
  value: "true"
  effect: NoExecute
  tolerationSeconds: 300
```

### Mistake 3: Taints Too Permissive

```yaml
# WRONG — defeats the purpose
# All pods have this toleration
tolerations:
- key: ""
  operator: Exists
# Taint does nothing
```

---

## Summary

**Taints:** Node says "I have requirements"
**Tolerations:** Pod says "I meet those requirements"

**Effects:**
- NoSchedule: Can't schedule new pods
- NoExecute: Evict existing pods
- PreferNoSchedule: Prefer other nodes

**Use cases:**
- Reserve nodes (GPU, high-memory)
- Maintenance windows
- Cost optimization
- Multi-tenant isolation

**Next:** Chapter 25 covers Horizontal Pod Autoscaler — automatically scale pods based on demand.
