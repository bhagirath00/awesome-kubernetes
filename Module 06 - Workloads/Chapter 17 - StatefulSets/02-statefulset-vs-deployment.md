# StatefulSet vs Deployment

## Introduction

Choosing between a StatefulSet and a Deployment is one of the most fundamental architectural decisions in Kubernetes. Making the wrong choice leads to data loss (using Deployment for a database) or unnecessary complexity (using StatefulSet for a stateless API). This file gives you the exact criteria to make the right call.

---

## The Decision Framework

Ask these questions in order:

```
1. Does each instance of this application need its own persistent data?
   Yes → StatefulSet
   No  → next question

2. Does each instance need a stable, permanent network identity (hostname)?
   Yes → StatefulSet
   No  → next question

3. Does the application require ordered startup or shutdown?
   Yes → StatefulSet
   No  → Deployment
```

---

## Real-World Classification

| Application | Right Choice | Reason |
|------------|-------------|--------|
| REST API server | Deployment | Stateless — any Pod serves any request |
| PostgreSQL primary-replica | StatefulSet | Per-Pod data + stable DNS for replication |
| Redis single instance | Deployment | Stateless if no persistence needed |
| Redis Cluster | StatefulSet | Per-node data + stable identity for cluster membership |
| Kafka broker | StatefulSet | Per-broker data + stable hostname (advertised listeners) |
| NGINX web server | Deployment | Stateless |
| Elasticsearch node | StatefulSet | Per-node data (index shards) + stable identity |
| Celery worker | Deployment | Stateless — processes tasks from queue |
| ZooKeeper | StatefulSet | Per-node data + quorum requires stable identity |

---

## Key Behavioral Differences — Update

**Deployment update:**
```
Old Pods: web-abc(v1)  web-def(v1)  web-ghi(v1)
                ↓ maxSurge: 1, maxUnavailable: 0
New Pods: web-jkl(v2)  web-mno(v2)  web-pqr(v2)
→ Several Pods update simultaneously (parallel)
→ Any order
```

**StatefulSet update (RollingUpdate):**
```
StatefulSet: redis-0(v1)  redis-1(v1)  redis-2(v1)
→ Update starts from the HIGHEST ordinal first
→ redis-2 updated first: terminate redis-2(v1) → create redis-2(v2) → wait for Ready
→ redis-1 updated next: terminate redis-1(v1) → create redis-1(v2) → wait for Ready
→ redis-0 updated last (primary is always last)
```

This reverse order is intentional — in a primary-replica setup, replicas are updated first. The primary (index 0 by convention) is updated last, minimizing disruption to write capacity.

---

## Partition-Based Updates — Safe Canary for StatefulSets

StatefulSets support a `partition` parameter for staged rollouts:

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2    # only update Pods with ordinal >= 2
```

With `partition: 2` and 3 replicas:
- `redis-2` is updated (ordinal 2 >= partition 2) → test it
- `redis-0` and `redis-1` stay on old version
- After validation: set partition to 0 → all Pods updated

```bash
# Start: update only redis-2 (highest risk, least impact)
kubectl patch statefulset redis -n production \
  -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'

kubectl set image statefulset/redis redis=redis:7.2-alpine -n production

# redis-2 updates → test it
kubectl exec redis-2 -n production -- redis-cli info server | grep redis_version
# redis_version:7.2.1   ← redis-2 is on new version

# redis-0, redis-1 still on old version:
kubectl exec redis-0 -n production -- redis-cli info server | grep redis_version
# redis_version:7.0.12  ← still old version

# After validation → update all
kubectl patch statefulset redis -n production \
  -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
```

---

## Key Behavioral Differences — Scaling

**Deployment scale up:**
```
kubectl scale deployment web-app --replicas=5
→ Creates Pods 4 and 5 simultaneously
→ Order does not matter — both start in parallel
```

**StatefulSet scale up:**
```
kubectl scale statefulset redis --replicas=5
→ redis-3 is created → waits for Ready
→ redis-4 is created → waits for Ready
→ Sequential — next Pod only starts after previous is Ready
```

**Deployment scale down:**
```
Deployment: terminates Pods in any order, in parallel
```

**StatefulSet scale down:**
```
StatefulSet: terminates redis-4 first (highest ordinal) → waits for termination
→ terminates redis-3 → waits
→ One at a time, reverse order
```

---

## Key Behavioral Differences — Pod Failure

**Deployment Pod failure:**
```
Pod web-abc (v1) crashes
→ ReplicaSet creates web-xyz (new random name, new IP)
→ Application must handle new instance joining mid-operation
```

**StatefulSet Pod failure:**
```
Pod redis-1 crashes
→ StatefulSet creates a new redis-1 (SAME name, SAME PVC, new IP)
→ Same DNS hostname → redis-0 and redis-2 can reconnect automatically
→ Data is preserved (same PVC)
```

This is the most critical difference for databases. When a database replica crashes, it needs to come back with the same hostname and the same data directory — so the primary can recognize it and resume replication.

---

## The Headless Service Requirement

Every StatefulSet requires a **Headless Service** (`clusterIP: None`) to provide stable DNS for each Pod. This is not optional:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
  namespace: production
spec:
  clusterIP: None          # Headless — no virtual IP
  publishNotReadyAddresses: true  # include unready Pods in DNS (needed during startup)
  selector:
    app: kafka
  ports:
  - port: 9092
    name: broker
```

The `publishNotReadyAddresses: true` field is important for StatefulSets where Pods need to discover each other during initialization — before they pass their readiness probes.

---

## When NOT to Use a StatefulSet

StatefulSets add operational complexity. Do not use them just because an application writes to disk:

**Use Deployment + PVC instead** when:
- Only 1 replica (single database instance — no replication)
- The application doesn't need stable DNS
- There is no coordination between replicas

```yaml
# Single-instance PostgreSQL → Deployment is fine
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-dev
spec:
  replicas: 1          # single instance — no stable identity needed
  template:
    spec:
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: postgres-dev-pvc   # manually created PVC
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
```

This is simpler than a StatefulSet and works perfectly for dev environments or single-instance databases.

---

## Summary

- Use StatefulSet when: each Pod needs its own persistent data, stable DNS hostname, or ordered startup/shutdown
- Use Deployment when: Pods are interchangeable, stateless, or only one replica is needed (even with a PVC)
- StatefulSet updates go in reverse order (highest ordinal first) — replicas update before the primary
- `partition` enables staged rollouts for StatefulSets — update the highest Pod first, validate, then roll out to all
- A Headless Service is mandatory for StatefulSets — it provides per-Pod DNS names (`pod-0.headless-svc.namespace`)
- `publishNotReadyAddresses: true` on the Headless Service enables inter-Pod discovery during initialization

In the next file, `03-ordered-deployment.md`, you will learn the mechanics of ordered deployment and termination in depth — how `OrderedReady` policy works, readiness gates, and initialization patterns for clustered applications.
