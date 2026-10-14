# StatefulSet Storage

## Introduction

StatefulSets are the Kubernetes workload designed for stateful applications — databases, message queues, and distributed systems. Unlike Deployments, StatefulSets give each Pod a stable identity (a persistent name, DNS record, and its own PersistentVolumeClaim) that survives Pod restarts and rescheduling.

The storage architecture for StatefulSets is fundamentally different from Deployments. Each Pod in a StatefulSet gets its own dedicated PVC — they never share a volume. This is what makes databases like PostgreSQL, MySQL, Kafka, and Cassandra work reliably on Kubernetes.

---

## Why StatefulSets Need Different Storage

```
Deployment (stateless):
  3 Pods share a single EFS volume → any Pod can serve any request
  Pod dies → new Pod picks up work → no state to track

StatefulSet (stateful):
  Pod 0 (postgres-0) → its OWN PVC (data-postgres-0) → its OWN data
  Pod 1 (postgres-1) → its OWN PVC (data-postgres-1) → different data
  Pod 2 (postgres-2) → its OWN PVC (data-postgres-2) → different data

postgres-0 is the PRIMARY — it cannot share its data directory with postgres-1
postgres-1 is a REPLICA — it has its own copy, synced from the primary
```

If all PostgreSQL Pods shared one volume, they would all try to write to the same data directory simultaneously — immediate data corruption.

---

## StatefulSet Storage Architecture

```
StatefulSet: postgres (3 replicas)
│
├─ Pod: postgres-0
│    ├─ Stable DNS: postgres-0.postgres-headless.production.svc.cluster.local
│    ├─ PVC: data-postgres-0  →  EBS vol-0aaa111 (100Gi)
│    └─ Data: primary database (read/write)
│
├─ Pod: postgres-1
│    ├─ Stable DNS: postgres-1.postgres-headless.production.svc.cluster.local
│    ├─ PVC: data-postgres-1  →  EBS vol-0bbb222 (100Gi)
│    └─ Data: replica database (synced from primary)
│
└─ Pod: postgres-2
     ├─ Stable DNS: postgres-2.postgres-headless.production.svc.cluster.local
     ├─ PVC: data-postgres-2  →  EBS vol-0ccc333 (100Gi)
     └─ Data: replica database (synced from primary)
```

Each Pod gets its own EBS volume. When `postgres-0` is deleted and recreated, it reattaches to `data-postgres-0` — the same EBS volume with all its data. The Pod has a persistent identity.

---

## Complete StatefulSet with Storage

```yaml
# Headless Service — required for StatefulSet Pod DNS
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - name: postgres
    port: 5432

---
# StatefulSet with per-Pod PVCs
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless    # required — links to the Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  
  # Pod update strategy — one Pod at a time, in reverse order (2, 1, 0)
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0              # update all Pods (set to 1 to canary test on Pod 2 first)

  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name   # "postgres-0", "postgres-1", etc.
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          initialDelaySeconds: 30
          periodSeconds: 10
        volumeMounts:
        - name: data                          # matches volumeClaimTemplates name below
          mountPath: /var/lib/postgresql/data

  # THIS is what creates per-Pod PVCs
  volumeClaimTemplates:
  - metadata:
      name: data                             # results in PVCs named "data-postgres-0", "data-postgres-1"
      labels:
        app: postgres
    spec:
      storageClassName: gp3
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
```

---

## PVC Naming Convention

When a StatefulSet with `volumeClaimTemplates` is created, PVCs are named automatically:

```
<volumeClaimTemplate.metadata.name>-<statefulset.name>-<ordinal>

StatefulSet: postgres
Template name: data
Ordinal: 0, 1, 2

Results in PVCs:
data-postgres-0   → bound to EBS vol-0aaa111 (used by postgres-0 Pod)
data-postgres-1   → bound to EBS vol-0bbb222 (used by postgres-1 Pod)
data-postgres-2   → bound to EBS vol-0ccc333 (used by postgres-2 Pod)
```

```bash
kubectl get pvc -n production -l app=postgres

# NAME               STATUS   VOLUME                  CAPACITY   ACCESS MODES   STORAGECLASS
# data-postgres-0    Bound    pvc-abc123...           100Gi      RWO            gp3
# data-postgres-1    Bound    pvc-def456...           100Gi      RWO            gp3
# data-postgres-2    Bound    pvc-ghi789...           100Gi      RWO            gp3
```

---

## PVC Persistence — Data Survives Pod Deletion

The most important property of StatefulSet PVCs: **they are NOT deleted when the Pod is deleted**.

```bash
# Delete postgres-0 Pod
kubectl delete pod postgres-0 -n production

# PVC still exists
kubectl get pvc data-postgres-0 -n production
# NAME              STATUS   VOLUME        CAPACITY
# data-postgres-0   Bound    pvc-abc123    100Gi    ← still bound, data preserved

# StatefulSet recreates postgres-0
kubectl get pod postgres-0 -n production
# NAME         READY   STATUS    RESTARTS
# postgres-0   1/1     Running   0    ← restarted, reattached same PVC, same data
```

Even if you delete the entire StatefulSet (`kubectl delete statefulset postgres`), the PVCs are NOT deleted. This is intentional — you might want to restore the StatefulSet from the same data. You must manually delete the PVCs when you are sure you want to destroy the data.

---

## Scaling StatefulSets — PVCs Created and Deleted Accordingly

```bash
# Scale up from 3 to 5 replicas — creates data-postgres-3 and data-postgres-4
kubectl scale statefulset postgres -n production --replicas=5

kubectl get pvc -n production -l app=postgres
# data-postgres-0  Bound
# data-postgres-1  Bound
# data-postgres-2  Bound
# data-postgres-3  Bound    ← new PVC for postgres-3
# data-postgres-4  Bound    ← new PVC for postgres-4

# Scale down from 5 to 3 — deletes postgres-4 and postgres-3 Pods
# BUT the PVCs (data-postgres-3, data-postgres-4) are KEPT
kubectl scale statefulset postgres -n production --replicas=3

kubectl get pvc -n production -l app=postgres
# data-postgres-0  Bound
# data-postgres-1  Bound
# data-postgres-2  Bound
# data-postgres-3  Bound    ← PVC kept even though Pod is deleted
# data-postgres-4  Bound    ← PVC kept even though Pod is deleted
```

PVCs from scale-down are kept to prevent accidental data loss. Manually delete orphaned PVCs when you are sure they are no longer needed.

---

## Summary

- StatefulSets give each Pod a stable name, DNS record, and its own dedicated PVC — essential for databases that cannot share data directories
- `volumeClaimTemplates` in the StatefulSet spec automatically creates one PVC per Pod, named `<template>-<statefulset>-<ordinal>`
- PVCs created by StatefulSets are NOT deleted when the Pod or StatefulSet is deleted — data survives independently
- When `postgres-0` is deleted and recreated, it reattaches to `data-postgres-0` — the same EBS volume with all the original data
- Scaling down keeps PVCs from the removed Pods — manually clean up orphaned PVCs after verifying data is no longer needed
- Always use `gp3` StorageClass with `ReadWriteOnce` for StatefulSet database storage; use `Retain` reclaim policy for production databases

In the next file, `02-volumeclaimtemplates.md`, you will go deep into `volumeClaimTemplates` — the field that makes per-Pod storage automatic, with advanced patterns for capacity planning and snapshot-based recovery.
