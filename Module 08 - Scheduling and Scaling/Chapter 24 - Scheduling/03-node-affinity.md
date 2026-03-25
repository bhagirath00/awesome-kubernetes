# Node Affinity

## Introduction

nodeSelector is simple but limited. What if you need:
- Pod prefers fast nodes, but can run on slow nodes if needed
- Pod wants zone A, but can tolerate zone B
- Pod requires any label in [ssd, nvme]

**Node Affinity** gives you this flexibility.

---

## Two Types of Affinity

### 1. Required Affinity (Hard Rule)

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  # Pod will NOT run unless this matches
```

Pod fails to schedule if no matching node.

### 2. Preferred Affinity (Soft Rule)

```yaml
preferredDuringSchedulingIgnoredDuringExecution:
  # Pod PREFERS matching node, but will run elsewhere if needed
```

Pod runs on any node if preferred node unavailable.

---

## Required Affinity (Hard Rule)

Pod MUST run on node with matching label:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
            - nvme
  containers:
  - name: postgres
    image: postgres:14
```

Translation: "Pod REQUIRES a node labeled disktype=ssd OR disktype=nvme. Will not run otherwise."

**nodeSelectorTerms:** Multiple terms (OR logic)

```yaml
nodeSelectorTerms:
- matchExpressions:          # TERM 1: OR
  - key: zone
    operator: In
    values: [us-east-1a]
- matchExpressions:          # TERM 2: OR
  - key: disktype
    operator: In
    values: [ssd]
```

Pod runs if TERM1 OR TERM2 matches.

---

## Preferred Affinity (Soft Rule)

Pod PREFERS matching node, but will run elsewhere:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: [us-east-1a]
      - weight: 20
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values: [ssd]
  containers:
  - name: web
    image: web:latest
```

Translation: "Prefer zone us-east-1a (weight 80) or SSD (weight 20). Run elsewhere if needed."

**Weight:** Higher weight = stronger preference (0-100).

---

## Operators

```yaml
# Operator: In
operator: In
values: [ssd, nvme]
# Matches: disktype=ssd OR disktype=nvme

# Operator: NotIn
operator: NotIn
values: [old, deprecated]
# Matches: any node NOT old or deprecated

# Operator: Exists
operator: Exists
values: []
# Matches: any node WITH the label (value doesn't matter)

# Operator: DoesNotExist
operator: DoesNotExist
values: []
# Matches: any node WITHOUT the label

# Operator: Gt (greater than)
operator: Gt
values: ["4"]
# Matches: labels where value > 4 (numeric)

# Operator: Lt (less than)
operator: Lt
values: ["10"]
# Matches: labels where value < 10 (numeric)
```

---

## Real-World Examples

### Example 1: Prefer Fast, Accept Slow

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-server
spec:
  affinity:
    nodeAffinity:
      # Hard: must have CPU request capacity
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: workload-type
            operator: In
            values: [compute]
      
      # Soft: prefer SSD, but CPU nodes OK too
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values: [ssd]
  
  containers:
  - name: api
    image: api:latest
```

Pod REQUIRES compute node, PREFERS SSD.

### Example 2: Multiple Zones for High Availability

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-replica
spec:
  affinity:
    nodeAffinity:
      # Spread across zones equally
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: [us-east-1a]
      - weight: 50
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: [us-east-1b]
  
  containers:
  - name: db
    image: postgres:14
```

Pod is balanced between two zones.

### Example 3: Exclude Old Nodes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: modern-app
spec:
  affinity:
    nodeAffinity:
      # REQUIRED: NOT on old nodes
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-version
            operator: NotIn
            values: [old, deprecated]
  
  containers:
  - name: app
    image: app:v2.0
```

Pod runs anywhere EXCEPT old nodes.

### Example 4: Capacity-Based Scheduling

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: big-job
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: capacity
            operator: Gt
            values: ["64"]  # Needs node with capacity > 64
  
  containers:
  - name: job
    image: big-job:latest
```

Pod only runs on large nodes (capacity > 64).

---

## Required vs. Preferred Comparison

```
Required:
  Pod: "I MUST run on SSD nodes"
  If no SSD node: Pod stays Pending forever
  Use for: Hard requirements (GPUs, compliance)

Preferred:
  Pod: "I prefer SSD, but will accept HDD"
  If no SSD node: Pod runs on HDD anyway
  Use for: Soft preferences (optimization)
```

---

## Common Mistakes

### Mistake 1: Wrong Operator

```yaml
# WRONG — trying to use = instead of In
matchExpressions:
- key: disktype
  operator: = 
  values: [ssd]

# Should be:
operator: In
```

### Mistake 2: Required When Should Be Preferred

```yaml
# WRONG — pod never runs (too strict)
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: zone
      operator: In
      values: [us-east-1a]
    - key: disktype
      operator: In
      values: [nvme]
    - key: cost
      operator: NotIn
      values: [spot]

# All conditions must match → probably no matching node

# Better: use Preferred for less critical constraints
preferredDuringSchedulingIgnoredDuringExecution:
- weight: 50
  preference:
    matchExpressions:
    - key: cost
      operator: NotIn
      values: [spot]
```

### Mistake 3: Forgetting Weight Calculation

```yaml
# Weights aren't scored additively!
preferredDuringSchedulingIgnoredDuringExecution:
- weight: 100
  preference: ...
- weight: 100
  preference: ...

# Scheduler calculates total score, picks best
# Highest weight = stronger preference
```

---

## Comparison: nodeSelector vs. Affinity

```
nodeSelector:
  disktype: ssd

Is equivalent to:

affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values: [ssd]
```

But affinity is more powerful.

---

## Debugging Affinity

```bash
# Check pod scheduling events
kubectl describe pod <pod-name>
Events:
  Type     Reason            Age   Message
  ----     ------            ---   -------
  Warning  FailedScheduling  2m    no nodes match affinity

# Check node labels
kubectl get nodes --show-labels

# Manually verify affinity rules
kubectl get nodes -l zone=us-east-1a
```

---

## Summary

Node Affinity is powerful scheduling:

**Required:** Pod MUST match (hard rule)
**Preferred:** Pod SHOULD match (soft rule)

**Operators:** In, NotIn, Exists, DoesNotExist, Gt, Lt

**When to use:**
- Multiple zones/regions
- Hardware preferences
- Cost optimization
- Capacity-based scheduling

**Next:** File 04 covers Pod Affinity/Anti-Affinity — placing pods relative to other pods.
