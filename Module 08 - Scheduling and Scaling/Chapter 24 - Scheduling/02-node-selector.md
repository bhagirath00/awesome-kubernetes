# Node Selector

## Introduction

Sometimes you need a pod to run on a specific type of node.

Maybe:
- Database needs fast SSDs
- GPU workload needs GPU servers
- High-memory service needs large nodes
- Compliance needs specific geographic region

**nodeSelector** is the simplest way to control this.

---

## How Node Selector Works

```
1. Label your nodes
   kubectl label nodes node1 disktype=ssd

2. In pod spec, add selector
   nodeSelector:
     disktype: ssd

3. Scheduler filters to matching nodes
   Result: Pod only runs on node1
```

---

## Basic Example

Label a node:

```bash
# Label node1 as having SSD
kubectl label nodes node1 disktype=ssd

# Label node2 as having HDD
kubectl label nodes node2 disktype=hdd

# View labels
kubectl get nodes --show-labels
NAME    STATUS   LABELS
node1   Ready    disktype=ssd
node2   Ready    disktype=hdd
```

Create pod that requires SSD:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database
spec:
  nodeSelector:
    disktype: ssd  # Only runs on SSD nodes
  containers:
  - name: postgres
    image: postgres:14
```

Result: Pod runs on node1 (has SSD label).

---

## Real-World Use Cases

### Use Case 1: GPU Workloads

```bash
# Label GPU nodes
kubectl label nodes gpu-node1 hardware=gpu
kubectl label nodes gpu-node2 hardware=gpu
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  nodeSelector:
    hardware: gpu
  containers:
  - name: trainer
    image: tensorflow:latest
    resources:
      limits:
        nvidia.com/gpu: 1  # Request GPU resource
```

Only GPU nodes run this pod.

### Use Case 2: Database Requires Performance

```bash
# Label high-performance nodes
kubectl label nodes perf-node1 tier=premium
kubectl label nodes perf-node1 storage=ssd
```

```yaml
apiVersion: v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  template:
    spec:
      nodeSelector:
        tier: premium
        storage: ssd
      containers:
      - name: db
        image: postgres:14
```

Database only runs on premium nodes.

### Use Case 3: Geolocation Compliance

```bash
# Label nodes by region
kubectl label nodes us-node1 region=us-east
kubectl label nodes eu-node1 region=eu-west
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: compliance-app
spec:
  nodeSelector:
    region: eu-west  # GDPR requires EU
  containers:
  - name: app
    image: compliance-app:latest
```

Pod only runs in EU (compliance).

### Use Case 4: Reserved Capacity

```bash
# Reserve nodes for production only
kubectl label nodes prod-node1 env=production
kubectl taint nodes dev-node1 env=development:NoSchedule
```

```yaml
# Production pod
apiVersion: v1
kind: Pod
metadata:
  name: prod-api
spec:
  nodeSelector:
    env: production
  containers:
  - name: api
    image: api:latest
```

Production pods only go to production nodes.

---

## Multiple Selectors (AND Logic)

All labels must match:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: picky-app
spec:
  nodeSelector:
    disktype: ssd       # AND
    memory: high        # AND
    region: us-east     # AND
  containers:
  - name: app
    image: app:latest
```

Node must have ALL three labels.

Node must be labeled:
```bash
kubectl label nodes node1 disktype=ssd memory=high region=us-east
```

---

## Viewing Node Labels

```bash
# List all nodes with labels
kubectl get nodes --show-labels

# List specific label
kubectl get nodes -L disktype
NAME    STATUS   DISKTYPE
node1   Ready    ssd
node2   Ready    hdd

# Find nodes by label
kubectl get nodes -l disktype=ssd
NAME    STATUS
node1   Ready
```

---

## Common Patterns

### Pattern 1: Tier Labels

```bash
# Label by service tier
kubectl label nodes tier-node1 tier=data
kubectl label nodes tier-node2 tier=compute
kubectl label nodes tier-node3 tier=api
```

Organize nodes by purpose.

### Pattern 2: Cost Labels

```bash
# Spot instances vs. on-demand
kubectl label nodes spot-node1 cost=low
kubectl label nodes on-demand-node1 cost=high
```

Run batch jobs on cheap nodes, critical apps on expensive nodes.

### Pattern 3: Capacity Labels

```bash
# Label by resources
kubectl label nodes big-node1 capacity=large
kubectl label nodes small-node1 capacity=small
```

Dispatch based on resource needs.

### Pattern 4: Purpose Labels

```bash
# Label by workload type
kubectl label nodes monitoring-node1 purpose=monitoring
kubectl label nodes logging-node1 purpose=logging
kubectl label nodes app-node1 purpose=app
```

Separate concerns by node purpose.

---

## Limitations of Node Selector

nodeSelector is simple but **limited**:

```
✓ Simple, easy to understand
✓ Performs exact matches
✗ Only AND logic (all labels must match)
✗ No OR logic (match any label)
✗ No NOT logic (must not have label)
✗ No complex expressions
```

For complex requirements, use **Node Affinity** (Chapter 24, File 3).

---

## Scheduling Failure with Node Selector

What happens if no node matches?

```yaml
# Pod requires label that doesn't exist
apiVersion: v1
kind: Pod
metadata:
  name: impossible
spec:
  nodeSelector:
    disktype: nvme  # No node has this label
  containers:
  - name: app
    image: app:latest
```

```bash
kubectl apply -f impossible-pod.yaml

# Pod stays pending
kubectl get pods
NAME         STATUS    AGE
impossible   Pending   5m

# Check events
kubectl describe pod impossible
Events:
  Type     Reason            Age   Message
  ----     ------            ---   -------
  Warning  FailedScheduling  5m    no nodes match selector
```

Pod never runs until a node gets the required label.

---

## Debugging Scheduling Issues

```bash
# Check node labels
kubectl get nodes --show-labels

# Check pod selector
kubectl get pod <pod-name> -o yaml | grep -A5 nodeSelector

# Describe pod for events
kubectl describe pod <pod-name>

# Manually assign if urgent
kubectl label nodes node1 missing-label=yes
```

---

## Best Practices

1. **Label consistently**
   ```bash
   # Good naming pattern
   disktype=ssd (what: value)
   tier=production
   region=us-east
   ```

2. **Document your labels**
   - Keep a label registry
   - Document what each label means

3. **Use for constraints, not preferences**
   - nodeSelector enforces hard requirements
   - For soft preferences, use affinity (next file)

4. **Test scheduling**
   ```bash
   kubectl apply -f pod.yaml
   kubectl get pod -o wide  # Check if scheduled
   ```

---

## Summary

nodeSelector is for simple node assignment:

**Usage:**
1. Label nodes: `kubectl label nodes node1 key=value`
2. Add selector to pod: `nodeSelector: key: value`
3. Scheduler matches pod to node

**When to use:**
- Different hardware types (GPU, SSD, high-memory)
- Geolocation/compliance
- Reserved node pools

**Limitations:**
- Only AND logic
- No OR or NOT logic
- For complex rules, use Node Affinity

**Next:** File 03 explains Node Affinity — powerful scheduling rules.
