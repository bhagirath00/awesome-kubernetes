# Storage Overview

## Introduction

Containers are ephemeral by design. When a container restarts or a Pod is replaced, the container's filesystem is wiped clean. For most stateless applications — web servers, APIs, microservices — this is fine. But for databases, message queues, and any application that needs to persist data across restarts, you need storage that outlives the container.

Kubernetes has a layered storage system that solves this problem. This file gives you the complete mental model before you go deep into each storage component.

---

## The Core Problem — Container Storage is Temporary

```
Pod starts
Container filesystem created (writable layer on top of the image)
Application writes data to /var/data/records.db
         │
Pod crashes → container is killed
         │
Kubernetes creates new container
Filesystem reset to image state → /var/data/records.db is GONE
```

Every file written inside a container's default filesystem is lost when the container restarts. This is not a bug — it is the design. Container images are immutable; the writable layer is ephemeral.

---

## The Kubernetes Storage Architecture

Kubernetes solves storage with a layered abstraction:

```
Application (Pod)
      │
      │ mounts
      ▼
Volume (in the Pod spec)
      │
      │ references
      ▼
PersistentVolumeClaim (PVC)
      │
      │ binds to
      ▼
PersistentVolume (PV)
      │
      │ backed by
      ▼
Physical Storage (AWS EBS, EFS, S3, NFS, local disk)
```

Each layer serves a specific purpose:

| Layer | Who Manages It | Purpose |
|-------|--------------|---------|
| **Volume** | Developer (Pod spec) | Connects a Pod to a storage source |
| **PersistentVolumeClaim (PVC)** | Developer | Request for storage (size, access mode, class) |
| **PersistentVolume (PV)** | Cluster admin / dynamic provisioner | Actual storage resource in the cluster |
| **StorageClass** | Cluster admin | Template defining how PVs are dynamically created |
| **Physical Storage** | AWS / cloud / IT team | The actual disk or file system |

The developer only deals with PVCs — they request "I need 20Gi of storage." The cluster provisions it automatically using a StorageClass.

---

## Volume Lifetime vs Pod Lifetime vs PersistentVolume Lifetime

This is the most important concept in Kubernetes storage:

```
emptyDir volume        → lifetime = Pod lifetime (wiped when Pod dies)
hostPath volume        → lifetime = Node lifetime (survives Pod but tied to one node)
PersistentVolume (EBS) → lifetime = independent (survives Pod and node deletion)
```

```
Pod A writes to PVC-backed EBS volume
Pod A is deleted → EBS volume remains intact, data preserved
Pod B (on a different node) binds the same PVC → reads Pod A's data
```

PersistentVolumes decouple data from the Pod lifecycle. The data lives on regardless of what happens to the Pod.

---

## Storage in AWS EKS — What You Use in Practice

On EKS, three storage options cover most use cases:

| Storage Type | AWS Service | Kubernetes Resource | Use Case |
|-------------|------------|---------------------|----------|
| **Block storage** | EBS (gp3) | PVC with `ReadWriteOnce` | Databases, stateful apps on single node |
| **File storage** | EFS | PVC with `ReadWriteMany` | Shared config, ML data, multi-pod access |
| **Object storage** | S3 | Not a volume (SDK access) | Large binary files, backups, ML datasets |
| **In-memory** | — | `emptyDir: {medium: Memory}` | Caching, scratch space, shared files between containers |

The AWS EBS CSI driver and AWS EFS CSI driver are installed on EKS clusters to bridge Kubernetes PVCs to AWS storage services.

---

## The CSI Standard — How Storage Plugins Work

Before Kubernetes 1.9, storage plugins were built into the Kubernetes codebase — adding a new storage provider required modifying and recompiling Kubernetes itself.

The **Container Storage Interface (CSI)** changed this. CSI is a standard that lets storage vendors write their own drivers (separate binaries) that run alongside Kubernetes as DaemonSets and Deployments:

```bash
# AWS CSI drivers installed on EKS
kubectl get pods -n kube-system | grep -E "ebs-csi|efs-csi"

# NAME                                       READY   STATUS
# ebs-csi-controller-7d8db9b5d-abc12        6/6     Running
# ebs-csi-controller-7d8db9b5d-def34        6/6     Running
# ebs-csi-node-4xkqp                        3/3     Running   (on every node)
# ebs-csi-node-7bmpv                        3/3     Running
```

The EBS CSI driver:
- `ebs-csi-controller` — handles provisioning and attaching EBS volumes
- `ebs-csi-node` — runs on every node, mounts volumes to Pods on that node

---

## Storage Access Modes — A Quick Reference

When requesting storage, you specify how many nodes can mount it simultaneously:

| Access Mode | Code | Meaning | AWS Storage |
|-------------|------|---------|------------|
| ReadWriteOnce | `RWO` | One node mounts read-write | EBS gp3 |
| ReadOnlyMany | `ROX` | Many nodes mount read-only | EFS, S3 via FUSE |
| ReadWriteMany | `RWX` | Many nodes mount read-write | EFS |
| ReadWriteOncePod | `RWOP` | Only one Pod mounts read-write | EBS gp3 (K8s 1.22+) |

EBS can only be mounted by one node at a time (`ReadWriteOnce`). This means if your Pod uses an EBS-backed PVC, it can only run on one node at a time — and must be on the same Availability Zone as the EBS volume.

EFS can be mounted by many nodes simultaneously (`ReadWriteMany`) — essential for shared storage across multiple replicas.

---

## Choosing the Right Storage Approach

Use this decision tree when your application needs persistent storage:

```
Does the data need to survive Pod restarts?
├── No → use emptyDir volume (in-memory or on-disk)
│
└── Yes → Does the data need to be shared across multiple Pods simultaneously?
           ├── No → EBS PVC (gp3, ReadWriteOnce)
           │         → cheap, fast, block storage
           │         → use for: databases, stateful apps
           └── Yes → EFS PVC (ReadWriteMany)
                     → shared filesystem
                     → use for: shared config, ML training data, CMS uploads
                     
Is the data very large (GBs-TBs) and accessed by the application via SDK?
└── S3 → not a volume — access via AWS SDK, boto3, s3fs
```

---

## Summary

- Container storage is ephemeral by default — all data written to a container's filesystem is lost when the container restarts
- Kubernetes solves this with a layered storage model: Volume → PVC → PV → StorageClass → Physical Storage
- Volumes are tied to Pod lifetime; PersistentVolumes are independent — data survives Pod and node deletion
- On EKS: EBS (block, `RWO`) for databases; EFS (file, `RWX`) for shared storage; S3 for object storage via SDK
- CSI drivers (`ebs-csi-driver`, `efs-csi-driver`) bridge Kubernetes PVCs to AWS storage services
- Access modes: `ReadWriteOnce` for EBS (one node), `ReadWriteMany` for EFS (many nodes simultaneously)

In the next file, `02-volumes-in-kubernetes.md`, you will learn about the Volume abstraction — the building block that connects Pods to any storage source, and how the volume lifecycle relates to the Pod lifecycle.
