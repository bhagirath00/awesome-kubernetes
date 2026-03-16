# Access Modes

## Introduction

The **access mode** of a PersistentVolume defines how many nodes can mount the volume and with what permissions. This is one of the most important storage decisions you make — it determines which applications can share storage and which must have exclusive access.

Choosing the wrong access mode causes Pods to stay `Pending` indefinitely or results in data corruption from concurrent writes.

---

## The Four Access Modes

```
ReadWriteOnce  (RWO)  — one node mounts read-write
ReadOnlyMany   (ROX)  — many nodes mount read-only
ReadWriteMany  (RWX)  — many nodes mount read-write
ReadWriteOncePod (RWOP) — only ONE Pod cluster-wide mounts read-write
```

---

## ReadWriteOnce (RWO) — The Most Common Mode

`ReadWriteOnce` means only one **node** can mount the volume for reading and writing at a time.

```yaml
spec:
  accessModes:
  - ReadWriteOnce
```

Key clarification: **one node, not one Pod**. Multiple Pods on the **same node** can all read from and write to an RWO volume simultaneously — as long as they are all on the same node.

```
Node A (ip-10-0-1-100):
  Pod 1 → mounts EBS volume in RWO mode → can read/write ✅
  Pod 2 → mounts SAME EBS volume on same node → can read/write ✅

Node B (ip-10-0-2-200):
  Pod 3 → tries to mount the SAME EBS volume → FAILS ❌
  (EBS can only attach to one EC2 instance at a time)
```

**AWS storage that supports RWO:**
- EBS (all types: gp3, io2, st1)
- EBS can physically only attach to one EC2 instance at a time

**Implications for Deployments:**
An RWO PVC used by a Deployment with multiple replicas causes scheduling problems. Only Pods on the same node as the EBS volume can mount it:

```yaml
# PROBLEMATIC: Deployment with 3 replicas + RWO PVC
spec:
  replicas: 3          # 3 Pods need the volume
  # If Pods land on different nodes → only 1 Pod can mount the EBS → 2 Pods stay Pending
```

For stateless Deployments with multiple replicas, use **emptyDir** (no data to share) or **EFS** (RWX). Reserve RWO EBS for StatefulSets where each Pod gets its own PVC, or for single-replica Deployments (databases).

---

## ReadOnlyMany (ROX) — Shared Read-Only Data

`ReadOnlyMany` allows many nodes to mount the volume for reading, but no node can write.

```yaml
spec:
  accessModes:
  - ReadOnlyMany
```

```bash
# Typical workflow:
# 1. A build job writes data to a volume (RWO)
# 2. Volume is snapshotted → new PV created from snapshot
# 3. Multiple Pods mount the snapshot PV in ROX mode for reading
```

**Use cases:**
- Read-only reference data distributed to many Pods (ML model weights, static dataset)
- Configuration loaded once by a separate process, read by many Pods
- EFS can serve as ROX — mount a directory read-only to many Pods

```yaml
# Multiple read-only consumers of a shared dataset
spec:
  accessModes:
  - ReadOnlyMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 50Gi
```

---

## ReadWriteMany (RWX) — Shared Read-Write

`ReadWriteMany` allows many nodes to mount the volume for both reading and writing simultaneously.

```yaml
spec:
  accessModes:
  - ReadWriteMany
```

**AWS storage that supports RWX:**
- **EFS (Elastic File System)** — fully managed NFS, multi-AZ, scales to petabytes
- FSx for Lustre — for HPC workloads
- FSx for NetApp ONTAP — for enterprise file storage

**EBS does NOT support RWX.** If you need shared read-write storage across multiple Pods on different nodes, you must use EFS.

**Use cases:**
- Shared file uploads (CMS, media storage)
- Machine learning training data accessed by multiple trainer Pods
- Shared application state that multiple Pods read and write (use with caution — applications must handle concurrent writes)
- Shared configuration data that changes at runtime

```yaml
# EFS PVC for shared read-write storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-uploads-pvc
  namespace: production
spec:
  storageClassName: efs-sc
  accessModes:
  - ReadWriteMany              # many Pods on many nodes can mount simultaneously
  resources:
    requests:
      storage: 100Gi
---
# Deployment with 3 replicas — all can mount the EFS PVC
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cms-app
spec:
  replicas: 3
  template:
    spec:
      volumes:
      - name: uploads
        persistentVolumeClaim:
          claimName: shared-uploads-pvc
      containers:
      - name: cms
        volumeMounts:
        - name: uploads
          mountPath: /var/www/uploads
```

All 3 replicas mount the same EFS share. A file uploaded via Pod 1 is immediately visible to Pods 2 and 3.

---

## ReadWriteOncePod (RWOP) — Exclusive Single Pod

`ReadWriteOncePod` (added in Kubernetes 1.22) provides the strictest guarantee — only **one Pod** in the entire cluster can mount the volume for reading and writing.

```yaml
spec:
  accessModes:
  - ReadWriteOncePod
```

Unlike `ReadWriteOnce` (one node, potentially many Pods), `RWOP` ensures exclusive access at the Pod level.

**Supported by:** EBS CSI driver (`ebs.csi.aws.com`) on Kubernetes 1.22+

**Use case:** Databases and stateful applications where even accidental multi-pod mounting would cause corruption (MySQL, PostgreSQL, Kafka). With `RWOP`, Kubernetes prevents a second Pod from mounting the volume — even if a human accidentally sets `replicas: 2` on a database Deployment.

```bash
# If you try to start a second Pod with the same RWOP PVC:
kubectl get events -n production | grep "RWOP"
# Warning  FailedMount  volume "pvc-xyz" is already mounted by Pod "postgres-old"
# New Pod stays Pending until the old Pod terminates
```

---

## Access Mode Support by Storage Type

| Access Mode | EBS | EFS | NFS | hostPath | emptyDir |
|------------|-----|-----|-----|---------|---------|
| RWO | ✅ | ✅ | ✅ | ✅ | ✅ |
| ROX | ❌ | ✅ | ✅ | ✅ | ❌ |
| RWX | ❌ | ✅ | ✅ | ✅ | ❌ |
| RWOP | ✅ | ❌ | ❌ | ❌ | ❌ |

A PV lists all the access modes it supports. A PVC requests one mode. The PVC's requested mode must be in the PV's supported list:

```yaml
# PV supports multiple modes
spec:
  accessModes:
  - ReadWriteOnce
  - ReadOnlyMany

# PVC requests one of the supported modes
spec:
  accessModes:
  - ReadOnlyMany    # valid — PV supports this
```

---

## Choosing the Right Access Mode

```
One Pod needs the storage (database, stateful app)?
├── Use RWOP (ReadWriteOncePod) → prevents accidental multi-mount
│   OR use RWO (ReadWriteOnce) → simpler, widely supported
└── → EBS gp3

Multiple Pods need the same data, but only need to read it?
├── Use ROX (ReadOnlyMany)
└── → EFS or pre-loaded volume snapshot

Multiple Pods need to read AND write to the same storage?
├── Use RWX (ReadWriteMany)
│   ← WARNING: application must handle concurrent writes (locking, etc.)
└── → EFS

Multiple containers in the SAME Pod need to share files?
├── Use emptyDir — no PVC needed, same Pod = same node
└── → No access mode concern (shared via volume, not PVC)
```

---

## Summary

- `ReadWriteOnce (RWO)` — one node can mount read-write at a time — supported by EBS and EFS — most common mode
- `ReadOnlyMany (ROX)` — many nodes can mount read-only — supported by EFS — for shared reference data
- `ReadWriteMany (RWX)` — many nodes mount read-write simultaneously — EFS only on AWS — for shared file storage and ML training data
- `ReadWriteOncePod (RWOP)` — only one Pod in the cluster can mount read-write — EBS CSI only — strictest database protection
- EBS does NOT support RWX — a Deployment with multiple replicas cannot share a single EBS PVC; use EFS instead
- Multiple Pods on the SAME node can all mount an RWO EBS volume — the restriction is per node, not per Pod

This completes Chapter 13 — Persistent Storage. In Chapter 14, `01-statefulset-storage.md`, you will learn how StatefulSets combine PVCs with ordered deployment — the pattern that makes databases and message queues reliable on Kubernetes.
