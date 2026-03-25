# Pod Affinity and Anti-Affinity

## Introduction

So far we've scheduled pods relative to nodes. But what if you care about OTHER pods?

- "This pod should run near the cache pod (same node)"
- "This pod should NOT run with the database (separate nodes)"
- "This pod should spread across zones with other API servers"

That's what **Pod Affinity** and **Pod Anti-Affinity** do.

---

## Pod Affinity (Attract)

```
Pod A wants to be near Pod B
Scheduler places Pod A where Pod B runs (or same node, or same zone)
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache-client
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: [cache]
        topologyKey: kubernetes.io/hostname  # Same node as cache
  
  containers:
  - name: app
    image: app:latest
```

Translation: "Run me on the same node (hostname) as pods labeled app=cache."

---

## Pod Anti-Affinity (Repel)

```
Pod A wants to be away from Pod B
Scheduler places Pod A on different node/zone from Pod B
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-replica
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: [database]
        topologyKey: kubernetes.io/hostname  # Different node from database
  
  containers:
  - name: db
    image: postgres:14
```

Translation: "Don't put me on the same node as database pods."

---

## Topology Key

Controls the level of separation:

```yaml
topologyKey: kubernetes.io/hostname
# Separate by node (different physical servers)

topologyKey: topology.kubernetes.io/zone
# Separate by zone (different availability zones)

topologyKey: topology.kubernetes.io/region
# Separate by region
```

---

## Real-World Examples

### Example 1: Co-locate App with Cache

```yaml
---
# Cache pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache
spec:
  template:
    metadata:
      labels:
        app: cache
    spec:
      containers:
      - name: redis
        image: redis:latest

---
# App pod wants to be on same node as cache
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: [cache]
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: web
        image: web:latest
```

App co-locates with cache for performance.

### Example 2: Spread Database Replicas

```yaml
---
# Database with anti-affinity to spread replicas
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-replicas
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: database
    spec:
      affinity:
        # Replicas should NOT run on same node
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: [database]
            topologyKey: kubernetes.io/hostname
      
      containers:
      - name: postgres
        image: postgres:14
```

All 3 replicas run on different nodes for redundancy.

### Example 3: High Availability with Zones

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        # Spread pods across zones
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: [web]
              topologyKey: topology.kubernetes.io/zone
      
      containers:
      - name: web
        image: web:latest
```

Web pods spread across zones (us-east-1a, us-east-1b, us-east-1c).

### Example 4: Avoid Database and Cache on Same Node

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache
spec:
  template:
    metadata:
      labels:
        app: cache
    spec:
      affinity:
        # Don't run cache on same node as database
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: [database]
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: redis
        image: redis:latest
```

Cache and database have separate nodes.

---

## Required vs. Preferred

### Required Pod Affinity

```yaml
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchExpressions:
      - key: app
        operator: In
        values: [cache]
    topologyKey: kubernetes.io/hostname
```

Pod WILL NOT RUN unless it can be co-located with cache pod.

Risk: If cache pod is evicted, affinity pod cannot run.

### Preferred Pod Affinity

```yaml
podAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values: [cache]
      topologyKey: kubernetes.io/hostname
```

Pod PREFERS to be near cache, but will run elsewhere if needed.

---

## Common Mistakes

### Mistake 1: Circular Affinity

```yaml
# WRONG — creates deadlock
# Pod A requires affinity with Pod B
# Pod B requires affinity with Pod A
# Neither can start!
```

### Mistake 2: Anti-Affinity Too Strict

```yaml
# WRONG — pod never starts
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchExpressions:
      - key: app
        operator: In
        values: [any-other-pod]
    topologyKey: kubernetes.io/hostname

# If ANY other pod with that label exists, new pod won't run
# Better: use Preferred for most cases
```

### Mistake 3: Wrong Label Selector

```yaml
# WRONG — selector matches nothing
labelSelector:
  matchExpressions:
  - key: app
    operator: In
    values: [nonexistent]

# Pod will stay pending (waiting for non-existent pod)
```

---

## Debugging Pod Affinity

```bash
# Check pod describe events
kubectl describe pod <pod-name>
Events:
  Type     Reason            Age   Message
  ----     ------            ---   -------
  Warning  FailedScheduling  2m    no nodes match pod affinity

# Check labels on pods
kubectl get pods --show-labels

# Check if required pod exists
kubectl get pods -l app=cache
NAME    READY   STATUS
cache   1/1     Running

# If cache pod doesn't exist, affinity pod can't start
```

---

## Performance Considerations

Pod affinity involves extra scheduler work:

```
1. Find all pods matching labelSelector
2. Get their node locations
3. Score nodes based on topology
4. Pick best node
```

With many pods, this gets expensive.

**Best practice:** Use preferred affinity when possible (more flexible).

---

## Summary

**Pod Affinity:** Attract pods to each other
- Co-locate for performance
- Use when pods need each other

**Pod Anti-Affinity:** Repel pods from each other
- Spread for redundancy
- Use with replicas

**Topology Key:**
- `kubernetes.io/hostname` = same node
- `topology.kubernetes.io/zone` = same zone
- Custom topology keys possible

**Next:** File 05 covers Taints and Tolerations — how to reserve nodes for special workloads.
