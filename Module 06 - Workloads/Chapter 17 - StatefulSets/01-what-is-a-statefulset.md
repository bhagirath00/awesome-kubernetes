# What is a StatefulSet

## Introduction

A **StatefulSet** is the Kubernetes workload designed for applications that require one or more of the following: stable, unique network identities; stable, persistent storage; ordered, graceful deployment and scaling.

Deployments treat all Pods as interchangeable — any Pod can serve any request. StatefulSets treat each Pod as an individual with a permanent identity. Each Pod has a stable name (like `postgres-0`, `postgres-1`), a stable DNS hostname, and its own persistent storage that follows it even if it is rescheduled to a different node.

---

## The Core Properties of StatefulSets

```
Property 1: Stable, unique Pod identity
  Deployment: web-app-abc123, web-app-def456  (random suffixes, change on restart)
  StatefulSet: postgres-0, postgres-1, postgres-2  (stable ordinals, permanent)

Property 2: Ordered deployment and scaling
  Deployment: all Pods start simultaneously
  StatefulSet: postgres-0 starts first → waits for Ready → postgres-1 starts → waits → postgres-2 starts

Property 3: Ordered, graceful termination (reverse order)
  Deployment: all Pods terminate simultaneously
  StatefulSet: postgres-2 terminates first → postgres-1 → postgres-0 (last, cleanest shutdown)

Property 4: Stable network identity (via Headless Service)
  Each Pod gets DNS: postgres-0.postgres-headless.production.svc.cluster.local
  Survives Pod restart — same DNS name, even if Pod IP changes

Property 5: Per-Pod PersistentVolumeClaims
  Each Pod has its own PVC — postgres-0 owns data-postgres-0
  If postgres-0 is deleted, recreated, or rescheduled, it reattaches data-postgres-0
```

---

## Why These Properties Matter for Databases

A PostgreSQL cluster needs each node to know which is the primary and which are replicas. With a Deployment:

```
Deployment: 3 Pods with random names and IPs
  Pod abc → crashes → recreated as Pod xyz (different name, different IP)
  PostgreSQL replication config: "connect to replica at IP 10.0.1.45"
  New Pod has IP 10.0.1.99 → replication breaks → manual reconfiguration required
```

With a StatefulSet:

```
StatefulSet: 3 Pods with stable names and DNS
  postgres-0 crashes → recreated as postgres-0 (same name, same DNS, same storage)
  PostgreSQL replication config: "connect to postgres-1.postgres-headless"
  DNS always resolves to the current postgres-1 Pod → replication works automatically
```

The stable identity is what makes self-healing databases possible on Kubernetes.

---

## StatefulSet vs Deployment

| | Deployment | StatefulSet |
|---|-----------|-------------|
| **Pod identity** | Random (changes on restart) | Stable ordinal (postgres-0, postgres-1) |
| **Storage** | Shared PVC or emptyDir | Per-Pod PVC via volumeClaimTemplates |
| **DNS** | Via Service ClusterIP (any Pod) | Per-Pod via Headless Service |
| **Start order** | Parallel | Sequential (0 → 1 → 2) |
| **Stop order** | Parallel | Reverse sequential (2 → 1 → 0) |
| **Updates** | Rolling (all at once up to maxSurge) | One at a time, in reverse order |
| **Scaling** | Instant (all Pods at once) | Sequential (one at a time) |
| **Use for** | Stateless: APIs, web, workers | Stateful: databases, queues, caches |

---

## Creating a StatefulSet

```yaml
# Required: Headless Service must be created BEFORE the StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
  namespace: production
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
  - port: 6379

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: production
spec:
  serviceName: redis-headless    # REQUIRED — links to Headless Service
  replicas: 3
  
  # Update strategy — one Pod at a time
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0    # update all Pods (set to 2 to update only Pod 2; set to 1 for Pods 1+2)
  
  # What to do when Pods need to be managed during scaling
  podManagementPolicy: OrderedReady   # default: wait for each Pod to be Ready before next
  # podManagementPolicy: Parallel     # start/stop all Pods simultaneously (faster, but loses ordering)

  selector:
    matchLabels:
      app: redis
  
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        command: ["redis-server", "--appendonly", "yes"]
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "500m"
            memory: "1Gi"
        readinessProbe:
          exec:
            command: ["redis-cli", "ping"]
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: data
          mountPath: /data
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      storageClassName: gp3
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

```bash
kubectl apply -f redis-statefulset.yaml

# Watch ordered Pod creation
kubectl get pods -n production -l app=redis -w

# NAME        READY   STATUS              AGE
# redis-0     0/1     ContainerCreating   2s
# redis-0     1/1     Running             8s    ← redis-0 is Running
# redis-1     0/1     Pending             1s    ← redis-1 starts only after redis-0 is Ready
# redis-1     1/1     Running             12s
# redis-2     0/1     Pending             1s    ← redis-2 starts after redis-1 is Ready
# redis-2     1/1     Running             15s
```

---

## Ordered Scaling

```bash
# Scale up — 4th Pod only starts after first 3 are Ready
kubectl scale statefulset redis --replicas=4 -n production

# Scale down — 4th Pod is deleted first, then 3rd, etc.
kubectl scale statefulset redis --replicas=2 -n production

# Watch scale-down: redis-3 → redis-2 (in reverse)
kubectl get pods -n production -l app=redis -w
```

---

## `podManagementPolicy: Parallel` — When Ordering Doesn't Matter

For some StatefulSet applications, the ordered startup is not needed — you just need stable identity:

```yaml
spec:
  podManagementPolicy: Parallel    # start/stop all Pods simultaneously
```

Use `Parallel` for:
- Read-only replicas that don't need to coordinate during startup
- Cache clusters (Redis Cluster after initial setup) where all nodes are equivalent
- Applications with their own built-in coordination protocol that doesn't rely on start order

Avoid `Parallel` for:
- Primary-replica setups (primary must be ready before replicas connect)
- Applications that do leader election using Pod ordinal (0 is always the leader)

---

## Accessing Individual StatefulSet Pods

```bash
# List all Pods and their individual DNS names
kubectl get pods -n production -l app=redis -o wide

# DNS names for each Pod (via Headless Service):
# redis-0.redis-headless.production.svc.cluster.local
# redis-1.redis-headless.production.svc.cluster.local
# redis-2.redis-headless.production.svc.cluster.local

# Connect directly to redis-0 (primary)
kubectl exec -it redis-0 -n production -- redis-cli ping
# PONG

# Connect directly to redis-1 (replica)
kubectl exec -it redis-1 -n production -- redis-cli info replication
# role:slave
# master_host:redis-0.redis-headless.production.svc.cluster.local
```

---

## Summary

- StatefulSets give each Pod a stable, permanent identity: predictable name, stable DNS, and its own PVC
- Unlike Deployments, StatefulSet Pods start sequentially (0 → 1 → 2) and terminate in reverse order (2 → 1 → 0)
- Each Pod gets its own PVC via `volumeClaimTemplates` — data is tied to the Pod ordinal, not the Pod instance
- `podManagementPolicy: OrderedReady` (default) is for databases requiring ordered startup; `Parallel` is for stateless distributed systems needing only stable identity
- Use StatefulSets for: PostgreSQL, MySQL, Redis, Kafka, ZooKeeper, Cassandra, Elasticsearch

In the next file, `02-statefulset-vs-deployment.md`, you will go deeper into the operational differences between StatefulSets and Deployments — update behavior, scaling behavior, and choosing between them for real applications.
