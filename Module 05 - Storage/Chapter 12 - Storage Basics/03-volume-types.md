# Volume Types

## Introduction

Kubernetes supports many volume types — each designed for a different storage backend and use case. This file is a comprehensive reference of the volume types you will use most in real EKS environments, with full YAML examples and production guidance for each.

---

## 1. `emptyDir` — Temporary Pod-Scoped Storage

`emptyDir` is the simplest volume type. It creates an empty directory when the Pod starts and deletes it when the Pod stops. It is backed by either the node's disk or RAM.

```yaml
volumes:
# On-disk emptyDir (default)
- name: scratch
  emptyDir: {}

# In-memory emptyDir (faster, uses RAM)
- name: cache
  emptyDir:
    medium: Memory
    sizeLimit: 512Mi    # optional — limits memory usage

# On-disk with size limit
- name: work-dir
  emptyDir:
    sizeLimit: 5Gi      # evict Pod if it uses more than 5Gi of disk
```

**Use cases:**
- Scratch space for intermediate processing
- Sharing files between containers in the same Pod (sidecar patterns)
- In-memory caching (fast reads, wiped on Pod restart)

Covered in detail in `04-emptydir-volume.md`.

---

## 2. `hostPath` — Node Filesystem

`hostPath` mounts a directory or file from the node's filesystem into the Pod. The data persists beyond the Pod's lifetime — but only on that specific node.

```yaml
volumes:
- name: node-logs
  hostPath:
    path: /var/log/pods     # path on the node filesystem
    type: Directory         # must be an existing directory

- name: socket
  hostPath:
    path: /var/run/docker.sock
    type: Socket            # Unix socket

- name: device
  hostPath:
    path: /dev/sda
    type: BlockDevice
```

`type` options:

| Type | Behavior |
|------|---------|
| `` (empty) | No checks — mount whatever exists or is created |
| `Directory` | Must exist as a directory |
| `DirectoryOrCreate` | Create if not exists |
| `File` | Must exist as a file |
| `FileOrCreate` | Create empty file if not exists |
| `Socket` | Must exist as a Unix socket |
| `BlockDevice` | Must exist as a block device |

**When to use:** DaemonSets that need access to node-level data (log collection from `/var/log`, monitoring agents accessing `/proc`, CNI plugins accessing host networking files). **Never use `hostPath` for application data** — data is tied to one node and lost if the node is replaced.

Covered in detail in `05-hostpath-volume.md`.

---

## 3. `persistentVolumeClaim` — The Standard for Persistent Data

This is the volume type you use for all real persistent application data. A PVC volume references a PersistentVolumeClaim, which is bound to a PersistentVolume backed by real storage (EBS, EFS, NFS):

```yaml
volumes:
- name: database-storage
  persistentVolumeClaim:
    claimName: postgres-pvc     # references a PVC object in the same namespace
    readOnly: false
```

PVCs are covered in depth in Chapter 13. This is the most important volume type for production workloads.

---

## 4. `configMap` — Configuration Files

ConfigMap volumes mount ConfigMap keys as files inside the container:

```yaml
volumes:
- name: nginx-config
  configMap:
    name: nginx-config-cm        # ConfigMap name
    defaultMode: 0644            # file permissions
    items:                       # optional: mount specific keys only
    - key: nginx.conf
      path: nginx.conf
```

Files update automatically within ~60 seconds when the ConfigMap changes. Covered in Chapter 9.

---

## 5. `secret` — Sensitive Files

Secret volumes mount Secret keys as files, decoded from base64 automatically:

```yaml
volumes:
- name: tls-certs
  secret:
    secretName: api-tls-secret
    defaultMode: 0400            # read-only for owner only
    items:
    - key: tls.crt
      path: certificate.pem
    - key: tls.key
      path: private.pem
      mode: 0400
```

Covered in Chapter 10.

---

## 6. `nfs` — Network File System

NFS volumes mount a remote NFS share. The NFS server must be reachable from all nodes:

```yaml
volumes:
- name: shared-storage
  nfs:
    server: nfs-server.mycompany.internal   # NFS server hostname or IP
    path: /exports/shared-data              # exported path on the NFS server
    readOnly: false
```

**On AWS:** Use EFS instead of bare NFS — it is managed, highly available, and scales automatically. Access EFS through the EFS CSI driver as a PVC with `ReadWriteMany`, not through the raw `nfs` volume type.

---

## 7. `csi` — Generic CSI Driver Volumes

The `csi` volume type allows inline use of any CSI driver without creating a PersistentVolume:

```yaml
volumes:
- name: secrets-store
  csi:
    driver: secrets-store.csi.k8s.io    # AWS Secrets Store CSI driver
    readOnly: true
    volumeAttributes:
      secretProviderClass: aws-secrets    # which SecretProviderClass to use
```

The AWS Secrets Store CSI driver uses this to mount AWS Secrets Manager secrets as files directly into Pods, without going through a Kubernetes Secret object. This is an alternative to the External Secrets Operator.

---

## 8. `awsElasticBlockStore` — Direct EBS (Legacy)

```yaml
volumes:
- name: ebs-volume
  awsElasticBlockStore:
    volumeID: vol-0abc123def456789   # EBS volume ID
    fsType: ext4
    readOnly: false
    partition: 0
```

> ⚠️ **Warning:** The `awsElasticBlockStore` volume type is **deprecated** and removed in newer Kubernetes versions. Do not use it in new code. Use PersistentVolumeClaims with the EBS CSI driver (`ebs.csi.aws.com`) instead. The PVC approach supports dynamic provisioning and is the correct modern pattern.

---

## 9. `ephemeral` — Inline PersistentVolumeClaim

Kubernetes 1.23+ supports inline ephemeral volumes — a PVC that is automatically created and deleted with the Pod:

```yaml
volumes:
- name: scratch-storage
  ephemeral:
    volumeClaimTemplate:
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 5Gi
```

This creates a PVC (and thus an EBS volume) when the Pod starts, and deletes both when the Pod is deleted. Useful for large scratch space that needs persistent-storage performance but should not outlive the Pod.

---

## Volume Types — Comparison Table

| Type | Data Persists After Pod? | Shared Across Nodes? | AWS Equivalent | Use In Production? |
|------|------------------------|---------------------|----------------|-------------------|
| `emptyDir` | ❌ No | ❌ No | — | ✅ For temp/sharing |
| `hostPath` | ✅ On same node | ❌ No | — | ⚠️ Only for DaemonSets |
| `persistentVolumeClaim` (EBS) | ✅ Yes | ❌ No (RWO) | EBS gp3 | ✅ Databases, stateful |
| `persistentVolumeClaim` (EFS) | ✅ Yes | ✅ Yes (RWX) | EFS | ✅ Shared storage |
| `configMap` | Auto-updated | ✅ (K8s object) | — | ✅ Configuration files |
| `secret` | Auto-updated | ✅ (K8s object) | — | ✅ Credentials, certs |
| `nfs` | ✅ Yes | ✅ Yes | Use EFS instead | ⚠️ Legacy |
| `csi` (Secrets Store) | No (in-memory) | — | Secrets Manager | ✅ Direct secret files |
| `awsElasticBlockStore` | ✅ Yes | ❌ No | EBS | ❌ Deprecated |
| `ephemeral` | ❌ Pod-scoped | ❌ No | EBS (temp) | ✅ Large scratch space |

---

## Choosing the Right Volume Type

```
Need to share files between containers in same Pod?
→ emptyDir

Need application configuration or secrets?
→ configMap or secret volume

Need data that persists beyond Pod lifetime?
→ persistentVolumeClaim
   ├── Only one node needs access → EBS (gp3 StorageClass, RWO)
   └── Multiple nodes need access → EFS (efs StorageClass, RWX)

Running a DaemonSet that needs node-level access?
→ hostPath

Need to mount AWS Secrets Manager secrets as files?
→ csi (Secrets Store CSI driver)
```

---

## Summary

- `emptyDir` — temporary, Pod-scoped; use for scratch space and inter-container sharing; `medium: Memory` for fast RAM-backed storage
- `hostPath` — node filesystem; use only in DaemonSets for accessing node-level data; data is tied to one specific node
- `persistentVolumeClaim` — the standard for all persistent application data; backed by EBS (single node) or EFS (multi-node)
- `configMap` and `secret` volumes mount Kubernetes objects as files; ConfigMaps auto-update in ~60s
- `csi` with the Secrets Store CSI driver mounts AWS Secrets Manager values directly as files
- `awsElasticBlockStore` is deprecated — always use PVCs with the EBS CSI driver instead
- `ephemeral` inline PVCs create and destroy EBS volumes tied to Pod lifetime — for large scratch needs

In the next file, `04-emptydir-volume.md`, you will go deep into `emptyDir` — the most commonly used temporary volume, covering its memory mode, size limits, container sharing patterns, and when to use RAM vs disk.
