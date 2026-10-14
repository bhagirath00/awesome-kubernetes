# Storage Class Parameters: Templates for Dynamic Provisioning

## The Storage Provisioning Problem Kubernetes Solved

Before Kubernetes, deploying stateful applications was tedious:

**Old Process (Manual):**
1. DevOps admin logs into AWS console
2. Creates an EBS volume (say, 100GB)
3. Creates Kubernetes PersistentVolume yaml pointing to vol-xyz123
4. Developer creates PersistentVolumeClaim requesting 100GB
5. Waits for manual matching
6. If volume too small: repeat process, new volume, new PV yaml
7. If volume too large: wasted money, can't shrink

This was **static provisioning**—each volume required manual AWS work plus Kubernetes YAML.

### The DevOps Dream: Dynamic Provisioning

**New Process (Kubernetes):**
1. Developer writes: "I need 100GB storage with gp3 performance"
2. Creates PVC
3. Kubernetes automatically creates the volume
4. Volume mounts
5. Done

This is **dynamic provisioning**—Kubernetes automates the entire workflow.

---

## What is a StorageClass?

A StorageClass is a **template definition** that tells Kubernetes:

> "When someone creates a PVC requesting storage of type X, here's how to automatically provision it"

It answers several critical questions:

1. **Who provisions?** (Which CSI driver?)
2. **Where?** (Which cloud region, subnet, AZ?)
3. **What kind?** (Performance tier, encryption, type?)
4. **How much control?** (Can it be deleted? Resized?)
5. **When to bind?** (Before or after pod scheduling?)

### The StorageClass as a Contract

Think of a StorageClass like a **service level agreement (SLA)**:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: database-tier  # Service name
provisioner: ebs.csi.aws.com  # Who fulfills it?
allowVolumeExpansion: true  # Can it grow?
volumeBindingMode: WaitForFirstConsumer  # Smart binding?
parameters:
  type: io2  # High performance
  iops: "20000"  # Guaranteed 20,000 I/O ops/sec
  throughput: "1000"  # 1000 MB/s throughput
  encrypted: "true"  # Encrypted at rest
reclaimPolicy: Retain  # Keep data if PVC deleted
```

When a developer creates a PVC with `storageClassName: database-tier`, they're saying: *"Give me what this contract defines."*

---

## The Provisioner: Who Creates the Volume?

The `provisioner` field specifies **which CSI driver** handles volume creation.

### Understanding Provisioner Selection

Different provisioners exist because storage systems are fundamentally different:

**Block Storage Provisioners:**
- `ebs.csi.aws.com` — AWS Elastic Block Store (single AZ, single-node attach)
- `disk.csi.azure.com` — Azure Managed Disk
- `pd.csi.storage.gke.io` — Google Persistent Disk

**File System Provisioners:**
- `efs.csi.aws.com` — AWS Elastic File System (multi-AZ, multi-node, NFS)
- `smb.csi.k8s.io` — SMB volumes (Windows fileshare style)

**Distributed Storage Provisioners:**
- `ceph.rook.io` — Ceph (open-source distributed storage)
- `netapp.trident.io` — NetApp Trident (enterprise SAN)

**Local/Test Provisioners:**
- `kubernetes.io/no-provisioner` — Local volumes (no dynamic provisioning)

### How Provisioner Discovery Works

When a PVC is created, Kubernetes:
1. Reads the `storageClassName`
2. Finds the StorageClass definition
3. Looks up the `provisioner` field
4. Finds the CSI driver pod(s)
5. Sends: "Create a volume with these parameters"
6. CSI driver calls the cloud API
7. Volume is created and returned

---

## Parameters: Cloud-Specific Configuration

The `parameters` section is **highly specific to each provisioner**. Different clouds have different options.

### AWS EBS Parameters

```yaml
parameters:
  type: gp3              # Volume type: gp2, gp3, io1, io2, st1, sc1
  iops: "3000"           # I/O ops per second (for gp3, io1, io2)
  throughput: "125"      # MB/s throughput (gp3 only)
  encrypted: "true"      # Enable EBS encryption at rest
  kmsKeyId: "..."        # Optional: custom KMS key ARN
  tagSpecification_1: "Environment=production"  # AWS tags
```

**Understanding these parameters:**

**type**: Selects the EBS volume class
- `gp2`: General Purpose v2 (2013, older, coupled IOPS-to-size)
- `gp3`: General Purpose v3 (2020, modern, independent IOPS/throughput)
- `io1/io2`: High IOPS provisioned volumes (expensive)
- `st1`: Throughput-optimized (for big data sequential access)
- `sc1`: Cold/archive storage (cheapest, slowest)

**Why gp3 is Now Default:**
- gp2 couples IOPS to capacity (3 IOPS per GB)
- If you need 9000 IOPS, you must buy 3000GB minimum
- gp3 decouples them (buy 100GB + 9000 IOPS independently)
- gp3 is cheaper, more flexible, better performance

**iops**: Guarantees specific I/O throughput
- `iops: "3000"` guarantees 3000 operations per second minimum
- AWS will deliver this consistently (with tail latency bounds)
- Important for databases, transaction processing

**throughput**: Bandwidth guarantee (gp3 only)
- `throughput: "125"` guarantees 125 MB/s
- Separate from IOPS (you can have low IOPS, high throughput or vice versa)
- Important for sequential workloads (big data, backups)

**encrypted**: Always enable for sensitive data
- Encryption at rest (data encrypted when written to disk)
- AWS handles key management (unless you use custom kmsKeyId)
- Compliance requirement for HIPAA, PCI-DSS, SOC2

### Google Cloud Parameters

```yaml
parameters:
  type: pd-ssd           # pd-standard, pd-ssd, pd-balanced
  replication-type: regional-pd  # For high availability
```

Different clouds have different concepts:
- Google uses "persistent disk" instead of "EBS"
- Google supports regional replication natively
- Parameter names and meanings vary by provider

---

## Dynamic Provisioning Workflow: From PVC to Running Pod

Understanding this workflow is critical for troubleshooting.

### Complete Step-by-Step Flow

**Step 1: Developer Creates PVC**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-database
  namespace: production
spec:
  storageClassName: database-tier  # References StorageClass
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi  # Requesting 100GB
```

Developer thinks in terms of: "I need 100GB of fast storage."

**Step 2: Kubernetes Validates Storage Class**

The PVC controller validates:
- Does StorageClass "database-tier" exist?
- Is the provisioner installed and running?
- Does the StorageClass allow this PVC's access mode?

If any check fails, PVC stays Pending forever.

**Step 3: CSI Provisioner Watches and Acts**

The provisioner sidecar container (running in the CSI driver pod) watches for unprovisioned PVCs:

```
"I see PVC app-database is unprovisioned"
"StorageClass says: provisioner=ebs.csi.aws.com, type=io2, iops=20000"
"I need to call AWS EC2 API"
```

**Step 4: CSI Driver Calls Cloud API**

The driver calls AWS:
```
POST https://ec2.amazonaws.com/
Action=CreateVolume&
VolumeType=io2&
Size=100&
Iops=20000&
AvailabilityZone=us-east-1a
```

AWS processes the request asynchronously:
- Allocates storage in specified AZ
- Configures IOPS and throughput
- Returns: `Volume ID: vol-0123456789abcdef0`

**Step 5: Kubernetes Creates PersistentVolume**

The provisioner creates a PV object:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pvc-abc123def456  # Auto-generated
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0123456789abcdef0  # The AWS volume ID
  persistentVolumeReclaimPolicy: Delete  # Per StorageClass
```

**Step 6: PVC Binds to PV**

The bind controller matches:
- PVC requesting 100Gi ↔ PV offering 100Gi
- Access modes match
- StorageClass match

PVC status changes: `Pending` → `Bound`

**Step 7: Pod References the PVC**

A pod spec can now use the PVC:
```yaml
spec:
  containers:
  - name: postgres
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-database  # References the PVC
```

**Step 8: kubelet Attaches Volume to Node**

When the pod is scheduled to a node:
1. kubelet sees the volume is needed
2. Calls the node plugin (CSI DaemonSet)
3. Node plugin tells AWS: "Attach vol-0123456789abcdef0 to instance i-abc123"
4. AWS attaches the volume
5. Device appears in the OS (e.g., `/dev/nvme1n1`)

**Step 9: Node Plugin Formats and Mounts**

The node plugin:
1. Formats the raw device: `mkfs.ext4 /dev/nvme1n1`
2. Mounts it: `mount /dev/nvme1n1 /var/lib/kubelet/pods/.../mount`
3. Pod's container now sees `/data` (mounted volume)

**Step 10: Container Starts**

The container starts and can now read/write to `/data` (the persistent volume).

---

## Why StorageClass Parameters Matter for Performance

Different workloads need different parameters:

### Database Workload
```yaml
parameters:
  type: io2           # Highest performance
  iops: "20000"       # High IOPS for transactional workload
  throughput: "1000"  # High throughput for replication
```
*Why:* Databases need consistent, low-latency I/O. Transactions require guaranteed performance.

### Cache/Session Store Workload
```yaml
parameters:
  type: gp3
  iops: "3000"        # Moderate IOPS
  throughput: "125"   # Default throughput
```
*Why:* Cache misses are acceptable. Doesn't need database-level guarantees.

### Backup/Archive Workload
```yaml
parameters:
  type: sc1           # Cold storage (cheapest)
  # IOPS/throughput not specified (sequential access doesn't need them)
```
*Why:* Archive rarely accesses data. Cheapest option makes sense. Latency irrelevant.

### Shared File System (logs, uploads)
```yaml
provisioner: efs.csi.aws.com  # Use EFS, not EBS
parameters:
  accessPoints: "true"  # Use access points for isolation
```
*Why:* Need multi-node access (ReadWriteMany). EBS won't work.

---

## Failure Modes and Troubleshooting

### Failure 1: Provisioner Not Found

**Symptom:**
```bash
kubectl describe pvc my-pvc
# Warning: ProvisioningFailed: provisioner efs.csi.aws.com not found
```

**Cause:**
- CSI driver not installed
- Provisioner name misspelled
- Driver pod crashed or not running

**Fix:**
```bash
# Verify driver is installed
kubectl get pods -n kube-system | grep csi

# Verify provisioner name in StorageClass
kubectl get sc my-sc -o yaml | grep provisioner
```

### Failure 2: Parameters Invalid

**Symptom:**
```bash
kubectl describe pvc my-pvc
# Error: invalid parameters: type=io3 is not supported
```

**Cause:**
- Parameter value not valid (io3 doesn't exist)
- Parameter not supported by provisioner version

**Fix:**
- Check AWS documentation for valid volume types
- Update StorageClass with correct parameter

---

## Summary: StorageClass as Infrastructure Code

A StorageClass is **infrastructure as code**:

```yaml
# This YAML defines: "In this cluster, 'high-performance' means:"
# - Use AWS EBS
# - gp3 volumes
# - 10,000 IOPS
# - 500 MB/s throughput
# - Encrypted by default
# - Can be resized
# - Smart AZ-aware binding
```

**Design Pattern: Multiple Classes**

Production clusters typically have multiple StorageClasses:

```
high-performance → io2, 20000 IOPS (databases)
general-purpose  → gp3, 3000 IOPS (apps)
archive          → sc1, cheap (backups)
shared           → EFS (multi-node workloads)
```

Developers choose the class that matches their workload's requirements.

**Next File:** Chapter 37 File 02 covers reclaim policies (what happens when a PVC is deleted).
