# volumeClaimTemplates

## Introduction

`volumeClaimTemplates` is the field in a StatefulSet spec that instructs Kubernetes to automatically create a separate PVC for each Pod. It is a template — Kubernetes uses it to stamp out one PVC per Pod replica, each with its own unique name and backing storage.

Without `volumeClaimTemplates`, you would have to manually create and name a PVC for every Pod — an impossible task for dynamic workloads. With it, StatefulSets are self-managing from a storage perspective.

---

## volumeClaimTemplates Anatomy

```yaml
spec:
  # ...StatefulSet spec...
  
  volumeClaimTemplates:
  - metadata:
      name: data                    # part of the PVC name: data-<statefulset>-<ordinal>
      labels:
        app: postgres               # labels applied to each created PVC
        environment: production
      annotations:
        backup.velero.io/include: "true"    # Velero backup annotation on each PVC
    
    spec:
      storageClassName: gp3         # which StorageClass to use
      accessModes:
      - ReadWriteOnce               # EBS: one node at a time
      resources:
        requests:
          storage: 100Gi            # each Pod gets 100Gi EBS
      
      # Optional: volume mode (Filesystem is default; Block for raw block devices)
      volumeMode: Filesystem
```

---

## Multiple volumeClaimTemplates — Different Volumes Per Pod

A StatefulSet can have more than one `volumeClaimTemplate`. Each template creates a separate PVC per Pod:

```yaml
volumeClaimTemplates:
# Primary data volume
- metadata:
    name: data
  spec:
    storageClassName: gp3
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 100Gi

# Write-Ahead Log (WAL) on faster storage
- metadata:
    name: wal
  spec:
    storageClassName: fast         # io2 StorageClass
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 20Gi              # smaller but faster disk for WAL

# Log archive directory
- metadata:
    name: archive
  spec:
    storageClassName: gp3
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 50Gi
```

For a StatefulSet `postgres` with 3 replicas, this creates 9 PVCs total:

```
data-postgres-0      100Gi  gp3   (postgres-0's data directory)
data-postgres-1      100Gi  gp3   (postgres-1's data directory)
data-postgres-2      100Gi  gp3   (postgres-2's data directory)
wal-postgres-0        20Gi  fast  (postgres-0's WAL)
wal-postgres-1        20Gi  fast  (postgres-1's WAL)
wal-postgres-2        20Gi  fast  (postgres-2's WAL)
archive-postgres-0    50Gi  gp3   (postgres-0's archive)
archive-postgres-1    50Gi  gp3   (postgres-1's archive)
archive-postgres-2    50Gi  gp3   (postgres-2's archive)
```

```yaml
# The container mounts all three volumes
volumeMounts:
- name: data
  mountPath: /var/lib/postgresql/data
- name: wal
  mountPath: /var/lib/postgresql/pg_wal
- name: archive
  mountPath: /var/lib/postgresql/archive
```

---

## Changing volumeClaimTemplates — You Cannot Edit Them

This is a critical limitation: **you cannot update `volumeClaimTemplates` in a StatefulSet after creation.**

```bash
kubectl edit statefulset postgres -n production
# Changing volumeClaimTemplates.spec.resources.requests.storage
# → Error: StatefulSet.apps "postgres" is invalid: spec.volumeClaimTemplates: Forbidden: updates to statefulset spec are not allowed except for...
```

To change `volumeClaimTemplates` (for example, to increase the default size), you must:
1. Note the current PVC names and data
2. Delete the StatefulSet (with `--cascade=orphan` to keep PVCs)
3. Update the StatefulSet YAML
4. Reapply the StatefulSet

```bash
# Delete StatefulSet without deleting Pods or PVCs
kubectl delete statefulset postgres -n production --cascade=orphan

# Verify Pods are still running
kubectl get pods -n production -l app=postgres

# Apply updated StatefulSet with new volumeClaimTemplates
kubectl apply -f updated-statefulset.yaml -n production

# StatefulSet adopts the existing Pods (based on selector match)
# New PVCs created for new Pods use the new template size
# Existing PVCs are NOT resized — manually expand them separately
```

---

## Resizing Existing StatefulSet PVCs

To expand existing PVCs after a StatefulSet was created (without recreating):

```bash
# Resize each PVC individually (StorageClass must have allowVolumeExpansion: true)
for i in 0 1 2; do
  kubectl patch pvc data-postgres-$i -n production \
    --type='merge' \
    -p='{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
done

# Watch resizing
kubectl get pvc -n production -l app=postgres -w

# NAME              STATUS   CAPACITY
# data-postgres-0   Bound    200Gi    ← resized
# data-postgres-1   Bound    200Gi    ← resized
# data-postgres-2   Bound    200Gi    ← resized

# Verify inside each Pod
for i in 0 1 2; do
  echo "postgres-$i:"
  kubectl exec postgres-$i -n production -- df -h /var/lib/postgresql/data
done
```

---

## Snapshot-Based PVC Initialization

For new StatefulSet replicas that need to start from an existing data snapshot (e.g., adding a new replica to an existing PostgreSQL cluster):

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    storageClassName: gp3
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 100Gi
    # Initialize from a snapshot
    dataSource:
      name: postgres-daily-snapshot    # VolumeSnapshot name
      kind: VolumeSnapshot
      apiGroup: snapshot.storage.k8s.io
```

This instructs the EBS CSI driver to create the EBS volume from a snapshot — the new Pod starts with all the data from the snapshot, then applies WAL to catch up to current state.

---

## Capacity Planning for volumeClaimTemplates

Before setting the storage size in `volumeClaimTemplates`, estimate the requirements:

```bash
# For PostgreSQL: check current database sizes
kubectl exec postgres-0 -n production -- \
  psql -U postgres -c "\l+" | awk 'NR>3 {print $1, $4}'
# myapp_db     45 GB
# metrics_db   12 GB
# Total: ~57 GB

# Add headroom: ~30% growth buffer + 20% for PostgreSQL overhead
# 57 GB × 1.5 = ~86 GB → round up to 100 GB

# For Kafka: brokers need: (message rate × retention period × replication factor) / replicas
# Example: 100 MB/s × 86400 s × 3 replicas / 3 = 8.64 TB per broker
# Round up with buffer → 10 TB per broker
```

---

## Backup Strategy for StatefulSet PVCs

Use AWS EBS snapshots via the Kubernetes snapshot API:

```yaml
# Create a VolumeSnapshotClass (one-time setup)
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-vsc
driver: ebs.csi.aws.com
deletionPolicy: Delete

---
# Create a snapshot of postgres-0's PVC
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-0-snapshot
  namespace: production
spec:
  volumeSnapshotClassName: ebs-vsc
  source:
    persistentVolumeClaimName: data-postgres-0
```

```bash
kubectl apply -f snapshot.yaml

kubectl get volumesnapshot -n production

# NAME                     READYTOUSE   SOURCEPVC        RESTORESIZE   SNAPSHOTCONTENT
# postgres-0-snapshot      true         data-postgres-0  100Gi         snapcontent-abc123
```

Automate daily snapshots with a CronJob or Velero.

---

## Summary

- `volumeClaimTemplates` is a list of PVC templates — Kubernetes creates one PVC per Pod per template, named `<template>-<statefulset>-<ordinal>`
- Multiple templates create multiple volumes per Pod — useful for separating data, WAL, and archive directories onto different StorageClasses
- `volumeClaimTemplates` cannot be edited after StatefulSet creation — use `kubectl delete statefulset --cascade=orphan` and reapply to change the template
- Resize existing StatefulSet PVCs individually with `kubectl patch pvc` — does not require StatefulSet modification
- Initialize new PVCs from snapshots using `dataSource` in the template — enables snapshot-based replica provisioning
- Implement EBS snapshot-based backup using the `VolumeSnapshot` API and `VolumeSnapshotClass`

In the next file, `03-stateful-app-patterns.md`, you will learn production patterns for running real stateful applications — PostgreSQL, Redis, and Kafka — on Kubernetes with proper storage configuration.
