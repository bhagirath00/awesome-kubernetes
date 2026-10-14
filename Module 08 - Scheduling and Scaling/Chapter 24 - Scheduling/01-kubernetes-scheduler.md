# The Kubernetes Scheduler

## Introduction

Imagine you have 10 servers and 100 pods that need to run. Which pod runs on which server?

That decision is made by **the Kubernetes Scheduler** — a component that watches for new pods and assigns them to nodes.

Without proper scheduling, you could have:
- All pods on one server (overloaded)
- Empty servers while others are full (wasteful)
- Incompatible workloads on the same server
- Pods failing because they need specific resources

The scheduler solves this with intelligent placement rules.

---

## How the Scheduler Works

```
1. Pod is created
   ↓
2. Scheduler watches for unscheduled pods (no .spec.nodeName)
   ↓
3. Scheduler finds eligible nodes (filtering phase)
   - Has enough CPU?
   - Has enough memory?
   - Can tolerate taints?
   - Matches nodeSelector?
   ↓
4. Scheduler scores eligible nodes (scoring phase)
   - Which node is best?
   - Consider pod affinity, existing load, etc.
   ↓
5. Scheduler binds pod to highest-scoring node
   ↓
6. kubelet picks up pod and starts it
```

**Key point:** The scheduler makes placement decisions, kubelet executes them.

---

## Scheduling Factors

### 1. Resource Requests

```yaml
# Pod needs 500m CPU and 256Mi memory minimum
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: app
    image: app:latest
    resources:
      requests:
        cpu: 500m      # Scheduler finds node with at least 500m free
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 512Mi
```

Scheduler won't place pod on node unless it has free resources.

### 2. Node Selector

```yaml
# Pod only runs on nodes with label disktype=ssd
apiVersion: v1
kind: Pod
metadata:
  name: database
spec:
  nodeSelector:
    disktype: ssd  # Scheduler finds nodes with this label
  containers:
  - name: db
    image: postgres:latest
```

Scheduler filters to only nodes matching the selector.

### 3. Affinity Rules (Advanced)

```yaml
# Pod prefers nodes in same zone (soft rule)
# or requires label app=payment (hard rule)
affinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - podAffinity:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - payment
```

More flexible than nodeSelector.

### 4. Taints and Tolerations

```yaml
# Node: "No GPU workloads" (taint)
# Pod: "I'm a GPU workload" (toleration)
# Result: Pod can run, others cannot
```

Used for specialized nodes (GPUs, high-memory, etc.).

---

## Default Scheduling Algorithm

Kubernetes uses a two-phase algorithm:

**Phase 1: Filter (yes/no)**
- Does node have enough CPU? YES/NO
- Does node have enough memory? YES/NO
- Does node match nodeSelector? YES/NO
- Can node tolerate pod taints? YES/NO

Result: List of eligible nodes

**Phase 2: Score (best?)**
- Balance pod distribution across nodes
- Consider pod affinity/antiaffinity
- Consider image locality
- Prioritize spreading

Result: Best-scoring node wins

---

## Default Scheduling Example

```
Cluster: 3 nodes
Node1: 2CPU, 4GB (1CPU 2GB already used)
Node2: 2CPU, 4GB (empty)
Node3: 2CPU, 4GB (1CPU 2GB already used)

New pod: Needs 500m CPU, 1GB memory

Filter:
  Node1: ✓ Has 1CPU, 2GB free (enough)
  Node2: ✓ Has 2CPU, 4GB free (enough)
  Node3: ✓ Has 1CPU, 2GB free (enough)

Score:
  Node1: Score based on load
  Node2: Score based on load (empty = good spread)
  Node3: Score based on load

Result: Pod goes to Node2 (best distribution)
```

---

## Scheduling Process in Detail

### Step 1: Pod Creation

```bash
kubectl apply -f pod.yaml
```

Pod is created with `status.phase: Pending` and no `spec.nodeName`.

### Step 2: Scheduler Detects It

Scheduler has a watch on all pods. It sees:
- Pod is pending
- Pod has no nodeName assignment
- Pod should be scheduled

### Step 3: Filtering

Scheduler applies all filters:

```
Can node1 run this pod?
  - Does it have resources? Check
  - Does it match selectors? Check
  - Can it tolerate taints? Check
  → Candidate

Can node2 run this pod?
  - Does it have resources? No
  → Filtered out

Can node3 run this pod?
  - Does it have resources? Check
  - Does it match selectors? Check
  - Can it tolerate taints? Check
  → Candidate

Candidates: [node1, node3]
```

### Step 4: Scoring

Scheduler scores candidates:

```
node1: Score 75 (has load, good distribution)
node3: Score 65 (has load, poor distribution)

Winner: node1
```

### Step 5: Binding

Scheduler creates a Binding object:

```bash
# Internally:
binding = {
  apiVersion: v1
  kind: Binding
  metadata:
    name: pod-name
    namespace: default
  target:
    apiVersion: v1
    kind: Node
    name: node1
}
```

### Step 6: kubelet Starts Pod

kubelet on node1 sees the binding and starts the pod.

---

## Viewing Scheduler Decisions

```bash
# Check which node pod is scheduled on
kubectl get pods -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP           NODE
web-pod     1/1     Running   0          2m    10.0.0.1     node1

# Get scheduling events
kubectl describe pod web-pod
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  2m    default-scheduler  Successfully assigned to node1
```

---

## Scheduler Configuration

Default scheduler is built-in. But you can:

1. **Use multiple schedulers** (custom logic)
2. **Adjust scheduler settings** (via ConfigMap)
3. **Enable scheduler plugins** (custom filtering/scoring)

Most teams don't need this. But advanced cases might.

---

## Common Scheduling Mistakes

### Mistake 1: No Resource Requests

```yaml
# WRONG — scheduler has no data
spec:
  containers:
  - name: app
    image: app:latest
    # No requests/limits = scheduler guesses

# Pod might be overcommitted, causing OOMKill
```

### Mistake 2: Overcommitted Node

```
Node: 4CPU, 8GB
Pod1: requests 2CPU, 4GB
Pod2: requests 2CPU, 4GB
Pod3: requests 1CPU, 2GB

All three scheduled (uses 5CPU, 10GB requested)
But node only has 4CPU, 8GB!

Result: Pod3 gets OOMKilled
```

**Fix:** Use resource quotas or limit scheduling requests.

### Mistake 3: No Taints on Special Nodes

```yaml
# WRONG — GPU node receives non-GPU pods
# Node is wasted on general workloads
```

**Fix:** Use taints to reserve nodes for special workloads.

---

## Summary

The Kubernetes Scheduler:

**What it does:**
- Watches for new pods
- Finds eligible nodes (filtering)
- Scores and picks best node (scoring)
- Assigns pod to node

**What matters:**
- Resource requests (CPU/memory)
- Labels/selectors
- Affinity rules
- Taints and tolerations

**Next:** File 02 explains nodeSelector — simple pod-to-node assignment. 