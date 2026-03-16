# Storage Classes

## Introduction

A **StorageClass** is a template that defines how PersistentVolumes are dynamically created. When a PVC references a StorageClass, the StorageClass's provisioner automatically creates the EBS volume (or EFS share, or other storage) and represents it as a PV — without any manual admin work.

StorageClasses abstract the underlying storage system. Application developers write PVCs that request a StorageClass name. The cluster administrator defines the StorageClasses. The two sides are decoupled — changing from `gp2` to `gp3` only requires updating the StorageClass, not every PVC.

---

## StorageClass on EKS — What You Get by Default

EKS clusters come with a default `gp2` StorageClass. For new clusters, AWS now recommends creating a `gp3` StorageClass and making it the default:

```bash
# View existing StorageClasses
kubectl get storageclass

# NAME      PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION
# gp2       kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false   ← default on older EKS
# gp3       ebs.csi.aws.com         Delete          WaitForFirstConsumer   true    ← modern recommendation
# efs-sc    efs.csi.aws.com         Delete          Immediate              false   ← for EFS

# Which is the default?
kubectl get storageclass -o yaml | grep "storageclass.kubernetes.io/is-default-class"
# storageclass.kubernetes.io/is-default-class: "true"   ← on the default one
```

---

## Creating a gp3 StorageClass

The `gp3` EBS volume type is better than `gp2` in every way — higher baseline performance, lower cost. Define it as your default:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # make this the default
provisioner: ebs.csi.aws.com      # AWS EBS CSI driver

parameters:
  type: gp3                       # EBS volume type
  iopsPerGB: "50"                 # IOPS (gp3: 3000 min, up to 16000)
  throughput: "125"               # MB/s (gp3: 125 min, up to 1000)
  encrypted: "true"               # encrypt the EBS volume with KMS
  kmsKeyId: "arn:aws:kms:us-east-1:123456789:key/abc-123"  # specific KMS key

reclaimPolicy: Delete             # delete EBS when PVC is deleted
allowVolumeExpansion: true        # allow PVC resize
volumeBindingMode: WaitForFirstConsumer  # don't provision until Pod is scheduled

mountOptions:
- noatime                         # don't update access time on reads (performance)
- nodiratime
```

```bash
kubectl apply -f gp3-storageclass.yaml

# Remove default annotation from old gp2 StorageClass
kubectl patch storageclass gp2 \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

---

## volumeBindingMode — A Critical Performance Setting

The `volumeBindingMode` field controls when EBS volumes are provisioned relative to Pod scheduling:

### `Immediate`
The EBS volume is provisioned the moment the PVC is created — before any Pod is scheduled.

**Problem:** EBS volumes exist in a specific Availability Zone. If the volume is provisioned in `us-east-1a` but the Pod gets scheduled to a node in `us-east-1b`, the Pod cannot start — it is in the wrong AZ.

### `WaitForFirstConsumer` (recommended)
The EBS volume is provisioned only after a Pod consuming the PVC is scheduled to a node. The EBS volume is created in the same AZ as the node.

```
Pod with PVC created
    │
    ▼
Scheduler determines which node the Pod should run on
    │
    ▼ (node is in us-east-1a)
EBS CSI driver provisions a gp3 volume IN us-east-1a
    │
    ▼
PVC is bound → Pod starts → volume mounted
```

**Always use `WaitForFirstConsumer` for EBS StorageClasses.**

---

## EBS Volume Types — Which StorageClass Parameter to Use

| Parameter Value | Type | Baseline IOPS | Max IOPS | Best For |
|----------------|------|--------------|----------|---------|
| `gp2` | SSD | 100 | 16,000 | Legacy — avoid for new deployments |
| `gp3` | SSD | 3,000 | 16,000 | General purpose, databases (best value) |
| `io1` | SSD | Configurable | 64,000 | High-IOPS databases (MySQL, PostgreSQL at scale) |
| `io2` | SSD | Configurable | 256,000 | Mission-critical, ultra-high IOPS (SAP, large Oracle) |
| `st1` | HDD | 40 per TB | 500 | Throughput-intensive sequential reads (log processing) |
| `sc1` | HDD | 12 per TB | 250 | Coldest data, infrequent access, lowest cost |

For most Kubernetes workloads, `gp3` is the right choice. Only switch to `io1`/`io2` if you have measured IOPS requirements above what `gp3` provides.

---

## EFS StorageClass

For workloads requiring `ReadWriteMany` (shared storage across multiple Pods), use EFS:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap          # use EFS Access Points
  fileSystemId: fs-abc123de         # your EFS file system ID
  directoryPerClaim: "true"         # create a separate directory for each PVC
  uid: "1000"
  gid: "1000"
  basePath: "/dynamic_provisioning" # base path for PVC directories in EFS
reclaimPolicy: Delete
volumeBindingMode: Immediate        # EFS is multi-AZ — no AZ binding needed
allowVolumeExpansion: false         # EFS storage is elastic (no fixed size)
```

```bash
# Install the EFS CSI driver (if not installed)
helm upgrade -i aws-efs-csi-driver \
  aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::123456789:role/EFSCSIDriverRole
```

A PVC using the EFS StorageClass:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-storage-pvc
  namespace: production
spec:
  storageClassName: efs-sc
  accessModes:
  - ReadWriteMany          # multiple Pods on multiple nodes can mount simultaneously
  resources:
    requests:
      storage: 5Gi         # EFS is elastic — this is a soft request, not a hard limit
```

---

## Multiple StorageClasses for Different Workloads

Define tiered StorageClasses to give developers appropriate storage options:

```yaml
# Tier 1: Fast NVMe for latency-sensitive databases
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "10000"
  encrypted: "true"
reclaimPolicy: Retain       # production databases keep data
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

# Tier 2: Standard gp3 for most workloads (default)
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

# Tier 3: Shared storage for multi-pod access
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: shared
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-abc123de
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

---

## Summary

- StorageClasses define how PVs are dynamically created — the provisioner, disk type, encryption, reclaim policy, and expansion settings
- Always use `gp3` instead of `gp2` for new EKS deployments — better performance at lower cost
- `WaitForFirstConsumer` is essential for EBS StorageClasses — it ensures the EBS volume is created in the same AZ as the Pod's node
- EFS StorageClasses use `ReadWriteMany` access mode and `Immediate` binding — EFS is multi-AZ by default
- `allowVolumeExpansion: true` must be set to allow PVC resizing; `reclaimPolicy: Retain` should be used for production databases
- Define tiered StorageClasses (`fast`, `gp3`, `shared`) so developers choose the right storage for their workload

In the next file, `04-dynamic-provisioning.md`, you will learn how dynamic provisioning works end-to-end — from PVC creation through EBS volume provisioning, Pod mounting, and cleanup.
