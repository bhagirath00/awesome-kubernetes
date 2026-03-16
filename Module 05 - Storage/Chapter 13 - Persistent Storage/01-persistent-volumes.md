# Persistent Volumes

## Introduction

A **PersistentVolume (PV)** is a piece of storage in the cluster that has been provisioned — either by an administrator or automatically by a StorageClass. It is a cluster-level resource (like a Node) — it does not belong to any namespace. Once provisioned, a PV can be claimed by a Pod in any namespace via a PersistentVolumeClaim.

The PV abstraction decouples storage from Pods. The actual storage (an EBS volume, an EFS share, an NFS mount) is represented as a PV object in Kubernetes. Pods never reference physical storage directly — they reference PVs through PVCs.

---

## PV Lifecycle

```
1. Provisioning
   Admin creates PV manually (static)
   OR StorageClass creates PV automatically (dynamic) ← preferred on EKS

2. Binding
   A PVC is created requesting storage
   Kubernetes matches PVC to a compatible PV (size, access mode, storage class)
   PV status changes to: Bound

3. Using
   Pod references the PVC in its volume spec
   Pod mounts the PV's backing storage

4. Releasing
   Pod is deleted
   PVC is deleted
   PV status changes to: Released

5. Reclaiming
   Based on reclaimPolicy:
   - Delete  → PV object deleted, backing storage (EBS) also deleted
   - Retain  → PV object stays (Released), backing storage preserved, manual cleanup needed
   - Recycle → (deprecated) basic scrub and make available again
```

---

## Static Provisioning — Manually Created PV

You create the physical storage first, then represent it as a PV:

```yaml
# Manually created PV backed by an existing EBS volume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-postgres-prod
  labels:
    type: postgres
    environment: production
spec:
  capacity:
    storage: 100Gi          # size of the volume
  
  accessModes:
  - ReadWriteOnce           # only one node can mount read-write
  
  persistentVolumeReclaimPolicy: Retain  # keep the EBS volume when PV is released
  
  storageClassName: gp3     # must match the PVC's storageClassName
  
  awsElasticBlockStore:     # legacy — use csi instead
    volumeID: vol-0abc123def456789
    fsType: ext4

  # OR modern CSI approach:
  csi:
    driver: ebs.csi.aws.com          # AWS EBS CSI driver
    volumeHandle: vol-0abc123def456789  # existing EBS volume ID
    fsType: ext4
    volumeAttributes:
      storage.kubernetes.io/csiProvisionerIdentity: ...
```

```bash
kubectl apply -f pv-postgres.yaml

kubectl get pv pv-postgres-prod

# NAME                  CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      STORAGECLASS
# pv-postgres-prod      100Gi      RWO            Retain           Available   gp3
```

---

## PV Status Values

```bash
kubectl get pv

# NAME              CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS
# pv-postgres       100Gi      RWO            Retain           Available  ← no PVC yet
# pv-mysql          50Gi       RWO            Delete           Bound      ← claimed by a PVC
# pv-redis          20Gi       RWO            Retain           Released   ← PVC deleted, data preserved
# pv-old-app        10Gi       RWO            Retain           Failed     ← binding error
```

| Status | Meaning |
|--------|---------|
| `Available` | PV is ready to be claimed — not bound to any PVC |
| `Bound` | PV is claimed by a PVC — in use |
| `Released` | PVC was deleted — PV data preserved (if Retain policy) — needs manual intervention |
| `Failed` | Automatic reclamation failed |

---

## Reclaim Policies — What Happens to Data After Release

The `persistentVolumeReclaimPolicy` field is one of the most important decisions when creating a PV:

### `Delete` (Default for Dynamically Provisioned PVs)

```yaml
persistentVolumeReclaimPolicy: Delete
```

When the PVC is deleted:
- The PV object is deleted
- The backing storage (EBS volume, EFS directory) is also deleted
- All data is permanently lost

Use when: the storage is disposable — test data, ephemeral batch jobs, development environments.

### `Retain`

```yaml
persistentVolumeReclaimPolicy: Retain
```

When the PVC is deleted:
- The PV stays (status: `Released`)
- The backing EBS volume is NOT deleted
- A human must manually decide: reclaim the data or delete it

After the data is archived, manually delete the PV to free it up:
```bash
# Manually delete the Released PV (after archiving data)
kubectl delete pv pv-postgres-prod
```

Use when: production databases, anything with valuable data. **Always use Retain for production databases.**

### `Recycle` (Deprecated)

Do not use. Use dynamic provisioning with `Delete` policy + snapshots for data protection instead.

---

## PV Access Modes

```yaml
accessModes:
- ReadWriteOnce    # RWO — one node, read-write
- ReadOnlyMany     # ROX — many nodes, read-only
- ReadWriteMany    # RWX — many nodes, read-write (EFS only on AWS)
- ReadWriteOncePod # RWOP — only ONE Pod cluster-wide (K8s 1.22+, EBS CSI)
```

A PV lists which access modes it **supports**. A PVC requests which access mode it **needs**. The PVC's requested mode must be one of the PV's supported modes.

EBS volumes support only `ReadWriteOnce` — a single node mounts them at a time. This means EBS-backed PVCs can only be used by a Deployment with `replicas: 1` or by StatefulSets (where each Pod gets its own PVC).

---

## PV Capacity and Storage Class Binding

When a PVC is created, Kubernetes finds a matching PV using these criteria in order:

1. **StorageClass** — must match (or both must be empty/default)
2. **Access mode** — PV must support the requested access mode
3. **Capacity** — PV must have at least the requested size

If multiple PVs match, Kubernetes picks the smallest one that satisfies the request:

```yaml
# PVC requests 20Gi
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  resources:
    requests:
      storage: 20Gi

# Available PVs: 10Gi (too small), 20Gi (exact), 50Gi (would work but wasteful)
# → Kubernetes binds to the 20Gi PV
```

---

## Viewing PV Details

```bash
# List all PVs in the cluster
kubectl get pv

# Detailed view of a specific PV
kubectl describe pv pv-postgres-prod

# Name:            pv-postgres-prod
# Capacity:        100Gi
# Access Modes:    RWO
# Reclaim Policy:  Retain
# Status:          Bound
# Claim:           production/postgres-pvc   ← which PVC owns this PV
# Storage Class:   gp3
# Reason:
# Message:
# Source:
#   Driver:          ebs.csi.aws.com
#   VolumeHandle:    vol-0abc123def456789
#   ReadOnly:        false
# Node Affinity:
#   Required Terms:
#     Term 0:        topology.kubernetes.io/zone in [us-east-1a]
#     ← EBS is AZ-specific — this PV can only be used by Pods in us-east-1a
```

The `Node Affinity` section shows AZ constraints. EBS volumes exist in a specific Availability Zone — the Pod using this PV must be scheduled in the same AZ.

---

## Dynamic vs Static Provisioning

| | Static | Dynamic |
|---|--------|---------|
| **Who creates PV** | Admin manually | StorageClass provisioner |
| **When** | Before PVC is created | When PVC is created |
| **Configuration** | Specific EBS volume ID | StorageClass template |
| **Scalability** | Low (manual per volume) | High (automated) |
| **Best for** | Migrating existing volumes | New applications |

In production on EKS, always use **dynamic provisioning** — you never manually create PVs. You create PVCs that reference a StorageClass, and the EBS CSI driver creates the EBS volume automatically.

---

## Summary

- PersistentVolumes are cluster-level resources representing provisioned storage — they are independent of any Pod or namespace
- PV lifecycle: Provisioned → Bound (to PVC) → Released (PVC deleted) → Reclaimed (based on policy)
- `Retain` reclaim policy preserves data after PVC deletion — always use for production databases
- `Delete` reclaim policy destroys the EBS volume when PVC is deleted — appropriate for disposable storage
- EBS PVs are AZ-specific — Pods using them must run in the same Availability Zone
- In modern EKS, use dynamic provisioning via StorageClass — never manually create PVs for new applications

In the next file, `02-persistent-volume-claims.md`, you will learn about PersistentVolumeClaims — the developer-facing storage request that triggers PV binding or dynamic provisioning.
