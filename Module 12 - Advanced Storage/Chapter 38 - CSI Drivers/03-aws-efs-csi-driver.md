# AWS EFS CSI Driver: Network-Based Shared Storage Architecture

## The EBS Limitation: Why We Need EFS

EBS volumes have a fundamental architectural constraint: **one volume can only attach to one node**. This works beautifully for single-pod workloads like databases, but what about workloads that need true multi-node shared storage?

### Common Multi-Node Scenarios

**Scenario 1: Content Delivery Network (CDN) Cache**
- 50 nginx pod replicas running across 10 nodes
- All 50 pods need to read/write the same cached content
- EBS can't do this (max one node)
- Need: ReadWriteMany (RWX) access

**Scenario 2: Shared Log Aggregation**
- 100 application pods across the cluster
- All pods write logs to a shared filesystem
- A log aggregator pod reads from the shared filesystem
- EBS can't support simultaneous writes from 100 pods
- Need: ReadWriteMany access

**Scenario 3: Git Repository for CI/CD**
- Jenkins workers pull from a shared git repo
- Multiple workers need simultaneous read access
- EBS won't work (single-node attachment)
- Need: ReadWriteMany access

### The Fundamental Problem with Block Storage

Block storage (like EBS) works at the low level:
- Raw device `/dev/sda1`
- Requires exclusive access for consistency
- If two servers write simultaneously to the same blocks, corruption occurs
- No built-in coordination mechanism

To share block storage across nodes, you'd need:
1. A distributed consensus protocol (expensive, complex)
2. RAID over a network (introduces latency, still has consistency issues)
3. A network file system layer on top (this is what EFS does)

---

## EFS: Network File System Approach

Amazon Elastic File System (EFS) is AWS's managed NFS service. Instead of block-level storage, it provides **file-level storage**.

### Key Architectural Difference

**EBS (Block Storage):**
```
Pod ↔ OS ↔ Device Driver ↔ EBS Volume
                              (directly attached)
```

**EFS (Network File System):**
```
Pod ↔ OS ↔ NFS Client ↔ Network ↔ EFS Service
                                   (over network)
```

EFS operates at the filesystem level, not block level. This enables:
- **Multi-node access**: Any node in the VPC can mount the same EFS filesystem
- **Multi-AZ availability**: EFS spans all AZs in a region automatically
- **Elasticity**: No fixed capacity; grows and shrinks automatically

---

## EFS's Superior Availability Model

Unlike EBS (bound to one AZ), EFS automatically replicates data across all AZs:

```
us-east-1 region
├─ us-east-1a: EFS replica with local copies
├─ us-east-1b: EFS replica with local copies  
└─ us-east-1c: EFS replica with local copies

Result: Pod in any AZ can access the same filesystem
```

This means:
- If an AZ fails, EFS is still accessible from other AZs
- Pod migration doesn't require data migration
- True geographic resilience

---

## Access Points: The Key to Cost-Effective Multi-Tenancy

Here's an operational challenge: if EFS is so great, why not use one for everything?

**Naive approach (Expensive):**
- 100 applications each need their own data
- Create 100 separate EFS filesystems
- Cost: 100 × $6 minimum/month + per-GB charges = expensive

**Better approach (Using Access Points):**
- Create 1 EFS filesystem
- Create 100 Access Points (virtual entry points)
- Each Access Point appears as a separate filesystem to the pod
- Each pod is completely isolated

### How Access Points Work

An Access Point acts as a virtualized gateway into an EFS filesystem:

```
EFS Filesystem: fs-abc123
├─ /app1  (Access Point 1) → Pod sees isolated directory
├─ /app2  (Access Point 2) → Pod sees isolated directory
└─ /app3  (Access Point 3) → Pod sees isolated directory

When pod connected via Access Point 1:
  └─ Pod sees /app1 as the root
  └─ Cannot see /app2 or /app3
  └─ Complete isolation
```

### Access Point Isolation Mechanisms

Access Points provide isolation through:

**1. Root Directory Enforcement**
```yaml
accessPoint:
  path: /app1
  # Pod mounted here sees /app1 as / (root)
  # Cannot cd ../app2 or escape
```

**2. POSIX User/Group Enforcement**
```yaml
accessPoint:
  posixUser:
    uid: 1000
    gid: 1000
  # All operations run as UID 1000
  # Prevents escalation or access to other apps' files
```

**3. Enforcement Mode**
```yaml
accessPoint:
  rootDirectory:
    creationInfo:
      ownerUid: 1000
      ownerGid: 1000
      permissions: "750"
  # Directory ownership and permissions enforced
```

### Cost Benefit

**Single EFS filesystem with 100 Access Points:**
- Base cost: $6/month
- Storage: $0.30/GB/month
- Access Points: No additional cost (they're virtual)
- Total: ~$6 + storage

vs.

**100 Separate EFS filesystems:**
- Total: 100 × $6 = $600/month
- Plus storage per filesystem
- 100x more expensive!

---

## The Dynamic Provisioning Workflow

The EFS CSI driver handles provisioning differently than EBS:

### The Setup (Admin Task)

```bash
# Step 1: Create EFS filesystem in AWS (once, manually)
# This is done outside Kubernetes
aws efs create-file-system --performance-mode generalPurpose

# Step 2: Create mount targets (one per AZ)
# This enables nodes to reach EFS
```

### The Kubernetes StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-dynamic
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-abc123def456  # The parent filesystem
  directoryPerms: "700"           # Directory permissions
  basePath: /dynamic_provisioning # Where APs are created
  gidRangeStart: "1000"           # UID/GID for isolation
  gidRangeEnd: "2000"
```

### The Workflow

1. **Developer creates PVC:**
   ```yaml
   kind: PersistentVolumeClaim
   metadata:
     name: shared-data
   spec:
     storageClassName: efs-dynamic
     accessModes:
       - ReadWriteMany    # RWX allowed!
     resources:
       requests:
         storage: 5Gi     # Size requested (largely ignored by driver)
   ```

2. **CSI Driver Action:**
   - Creates a subdirectory: `/dynamic_provisioning/pvc-abc123/`
   - Creates an Access Point pointing to this directory
   - Applies POSIX user/group isolation
   - Returns the Access Point ID

3. **Pod Mounts:**
   - The node's NFS client mounts the EFS via the Access Point
   - Pod sees isolated directory
   - Can scale to 100 replicas; all share the same data

---

## When NOT to Use EFS

Understanding limitations is as important as understanding benefits.

### Limitation 1: Latency for Databases

**EFS is NFS over the network.** Network latency is much higher than block storage latency.

- **EBS latency**: ~1-3ms (direct block device)
- **EFS latency**: ~10-20ms (over NFS, plus network)

For databases with tight ACID transactional requirements:
- MySQL FLUSH TABLES WITH READ LOCK (expects fast fsync)
- PostgreSQL checkpoint (expects fast disk I/O)
- MongoDB writes (expects <10ms fsync time)

Network file systems introduce latency that breaks these guarantees. **Never use EFS/NFS for databases.**

### Limitation 2: POSIX Compliance

NFS is not 100% POSIX-compliant. Specific behaviors differ:

- File locking semantics differ (advisory vs mandatory)
- Directory entry caching causes stale reads
- Performance of small random I/O is terrible

Applications expecting strict POSIX file semantics may fail.

### Limitation 3: Performance for High-Concurrency Metadata Operations

If 100 pods simultaneously:
- Create files
- List directories
- Check file attributes

NFS has limits on metadata operation throughput. Metadata operations become bottlenecks.

---

## Networking Requirements: The Critical Security Group Issue

EFS requires network access on **NFS Port 2049**. This is frequently misconfigured.

### The Setup

1. **EFS Mount Targets** (created in each AZ):
   - These are ENIs (Elastic Network Interfaces) in your VPC
   - They have a security group
   - Port 2049 must be open

2. **Node Security Group**:
   - Worker nodes must reach port 2049 on EFS
   - The EFS security group must allow inbound 2049 from node SG

### Common Failure Mode

```yaml
# Pod stuck in ContainerCreating
# Events show: mount.nfs4: Connection refused

# Root cause: EFS security group doesn't allow port 2049
```

**Fix:**
```bash
# Edit EFS mount target security group
aws ec2 authorize-security-group-ingress \
  --group-id sg-efssg123 \
  --protocol tcp \
  --port 2049 \
  --source-security-group-id sg-nodessg456
```

---

## Performance Characteristics

EFS has different performance profiles than EBS:

**Good at:**
- Sequential reads/writes
- Shared access patterns
- Variable workloads (scales automatically)

**Poor at:**
- Random I/O heavy workloads
- High-concurrency metadata ops
- Latency-critical applications

**Throughput Modes:**
- **Bursting (default)**: Good for unpredictable workloads, pay less
- **Provisioned**: Guarantee exact throughput, pay more

---

## Use Case Decision Tree

```
Does your workload need multi-pod write access?
  ├─ No → Use EBS (better performance)
  └─ Yes (need RWX)
      ├─ Is it a database? → Use EBS on single pod (RWO)
      ├─ Is it NFS-sensitive (strict POSIX)? → Use on-prem storage/EBS RAID
      └─ Is it file-sharing (web app uploads, logs, cache)? → Use EFS
```

---

## Summary: EFS in Kubernetes Architecture

**Architectural Role:** Network-shared filesystem for multi-node workloads

**Key Benefits:**
- Multi-AZ automatic replication
- ReadWriteMany access
- Cost-effective multi-tenancy via Access Points
- Elastic (no pre-provisioning)

**Key Constraints:**
- Higher latency (not for databases)
- POSIX compliance gaps
- Network dependency

**Best Practices:**
- Use Access Points for isolation
- Configure security groups for port 2049
- Monitor throughput for concurrent workloads
- Never use for databases

**Next File:** Chapter 38 File 04 covers installing and securing the EBS and EFS CSI drivers with IAM.
