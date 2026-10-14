# AWS EBS CSI Driver: Understanding Block Storage Binding and Constraints

## EBS: The Foundation of AWS Block Storage

Amazon Elastic Block Store (EBS) is AWS's primary block storage service. For Kubernetes users, EBS is the default persistent storage backend because:

1. **Tight AWS Integration**: EBS volumes are managed by the EC2 service; no external software needed
2. **Performance Predictability**: You specify IOPS and throughput explicitly, and AWS guarantees them
3. **Mature Service**: EBS has been battle-tested since 2008; it's the foundation of most AWS infrastructure
4. **Cost Optimization**: Pay-as-you-go model with predictable pricing

However, EBS has **architectural constraints** that you must understand to use it correctly in Kubernetes.

---

## The Fundamental EBS Constraints: AZ-Binding and Single-Node Access

### Constraint 1: Availability Zone Binding

**EBS volumes are tightly bound to a single Availability Zone.**

When you create an EBS volume in us-east-1a, that volume exists only in that AZ. You cannot move it to us-east-1b or us-east-1c. This is a fundamental AWS architecture decision—each AZ has separate power, cooling, and networking infrastructure.

*Why this constraint exists:* Availability Zones are geographically separated data centers. Replicating block storage across AZs in real-time would introduce network latency that would destroy database ACID properties. EBS instead optimizes for low latency *within* an AZ by keeping storage local to compute.

### Constraint 2: ReadWriteOnce Access (RWO)

**EBS volumes can only be attached to one EC2 instance at a time.**

This is distinct from "one node" in Kubernetes terms. Multiple *pods* on the same Kubernetes node can read and write to the same EBS volume simultaneously. But pods on different nodes cannot share the same EBS volume.

*Why this constraint exists:* EBS is a block-level storage device, not a network file system. It requires exclusive access for consistency. If two servers tried to write to the same block device simultaneously, corruption would occur.

---

## The AZ-Binding Problem in Kubernetes

These two constraints create a critical scheduling problem in Kubernetes:

### The Problem Scenario

Suppose your Kubernetes cluster spans 3 Availability Zones with nodes in each:
- Zone A: 2 worker nodes
- Zone B: 2 worker nodes  
- Zone C: 2 worker nodes

A developer creates a PVC requesting 50GB of EBS storage:

```yaml
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  storageClassName: ebs-sc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

**Without proper coordination:**

1. The CSI provisioner immediately creates a 50GB EBS volume
2. But which AZ? The provisioner doesn't know yet where the pod will run
3. It might create the volume in us-east-1b
4. But later, when the pod is scheduled, the kube-scheduler assigns it to a node in us-east-1a
5. Result: **The pod can't mount the volume** (cross-AZ attachment is impossible)

This is a classic chicken-and-egg problem:
- **Storage provisioning** (controller) happens before **pod scheduling** (scheduler)
- But the storage location depends on the pod's final node assignment
- Yet the scheduler doesn't know about the volume until it's already created

### The Legacy Kubernetes Approach (Pre-CSI)

Older in-tree AWS storage provisioners solved this with a workaround: they would create volumes without specifying an AZ, letting AWS randomly place them. Then they hoped the pod would schedule to that AZ. If it didn't, the pod would get stuck pending.

This was unreliable and caused mysterious "pod won't start" incidents.

---

## VolumeBindingMode: WaitForFirstConsumer

The CSI standard introduced a sophisticated solution: **VolumeBindingMode**.

This is a StorageClass parameter that controls *when* volume provisioning happens relative to pod scheduling.

### Two Binding Modes

**1. Immediate (Bind Before Scheduling)**
```yaml
volumeBindingMode: Immediate  # Default for backward compatibility
```

With `Immediate`:
- PVC is bound to a PV as soon as the PVC is created
- Volume is provisioned immediately
- Pod scheduling happens later
- **Problem:** Volume might be in wrong AZ

**2. WaitForFirstConsumer (Bind After Scheduling)**
```yaml
volumeBindingMode: WaitForFirstConsumer  # Recommended for block storage
```

With `WaitForFirstConsumer`:
- PVC remains unbound initially
- When a pod tries to use the PVC, kube-scheduler doesn't proceed until it knows the pod's target node
- *Then* the CSI provisioner is called
- The provisioner gets the node's AZ information
- The volume is created in that exact AZ
- Pod and volume are guaranteed to be co-located
- **Benefit:** Zero risk of AZ mismatch

### The Workflow with WaitForFirstConsumer

```
1. Developer creates PVC (unbound, waiting)
   └─ Status: Pending

2. Developer creates Pod that requests PVC
   └─ Scheduler needs to place pod but sees: PVC not bound

3. Scheduler says: "I want to place this pod on node-b2 (in us-east-1b)"
   └─ But PVC must be bound first in that same AZ

4. Scheduler triggers volume provisioning:
   └─ "CSI driver, create volume in us-east-1b"

5. CSI driver calls AWS:
   └─ "Create 50GB gp3 volume in us-east-1b"
   └─ AWS returns: vol-xyz789 (in us-east-1b)

6. PV created, PVC bound to PV
   └─ Status: Bound

7. Scheduler proceeds: pod starts on node-b2
   └─ Volume attachment succeeds (same AZ)

Result: Pod and volume are perfectly aligned
```

---

## Why gp3 Over gp2

AWS released gp3 volumes in 2020 with significant improvements over the older gp2:

**gp2 (General Purpose v2, 2013):**
- IOPS coupled to size (3 IOPS per GB)
- If you need 9000 IOPS, you must buy 3000GB minimum
- You pay for size even if you don't need all IOPS
- Limited burst
- Released over a decade ago; AWS has moved on

**gp3 (General Purpose v3, 2020):**
- IOPS independent from size (baseline 3000, up to 16000 IOPS)
- Throughput independent from size (baseline 125, up to 1000 MB/s)
- You buy exactly what you need
- Cheaper for most workloads
- Better burst behavior
- Modern, optimized for today's workloads

**Example Cost Comparison:**
Need 1TB with 10000 IOPS:
- **gp2**: Must buy 3.33TB ($0.10 × 3333 × 730 = $243/month) to get 10000 IOPS
- **gp3**: Buy 1TB + 7000 IOPS ($0.10 × 1024 + $0.015 × 7000 = $205/month)

**Recommendation:** Use gp3 for all new workloads. Only use gp2 for legacy systems.

---

## Volume Attachment Mechanics: Understanding Multi-Attach Errors

When a pod is scheduled, several systems must coordinate:

1. **kube-scheduler** places pod on a node
2. **CSI external-attacher** requests volume attachment
3. **EC2 API** processes the attachment
4. **Node plugin** formats and mounts

Sometimes this fails with an error:

```
Multi-Attach error for volume "pvc-abc123"
Volume is already exclusively attached to one node and can't be attached to another
```

### Why This Happens

This error occurs when:

**Scenario A: Node Crash Without Clean Unmount**
- Pod was running on node-1, using EBS volume
- node-1 crashes suddenly (power failure, network partition)
- Kubernetes marks the pod for termination on a new node
- kube-scheduler assigns pod to node-2
- node-2 tries to attach the volume
- But AWS still thinks the volume is attached to node-1 (cleanup didn't happen)
- AWS refuses to attach to node-2 (exclusive attachment rule)

**Scenario B: Kubernetes Bug or Race Condition**
- Rare: two nodes try to attach the same volume simultaneously
- One succeeds, other fails

### Recovery Steps

1. **Verify the volume is really gone from old node:**
   ```bash
   kubectl get volumeattachment
   # Lists all attachments; look for your volume
   ```

2. **Force detach if node is permanently down:**
   ```bash
   kubectl delete volumeattachment pvc-abc123
   # Or manually in AWS console: Volumes > Actions > Detach
   ```

3. **Restart the pod:**
   ```bash
   kubectl delete pod my-pod
   # Pod recreates, reschedules, attaches successfully
   ```

---

## Storage Class Parameters for EBS

A properly configured EBS StorageClass includes several parameters:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3                    # Volume type (gp2, gp3, io2, io1)
  iops: "3000"                 # I/O operations per second (requires gp3/io2)
  throughput: "125"            # MB/s (gp3 only; io2 doesn't have this)
  encrypted: "true"            # Always enable for security
  # kmsKeyId: arn:aws:kms:...  # Optional: custom KMS key for encryption
  tagSpecification_1: "Name=kubernetes-pvc"  # Tag the AWS volume for tracking
```

### When to Use Each Parameter

- **type**: Choose based on workload (gp3 for most, io2 for database)
- **iops**: Only needed for gp3/io2; gp2 is fixed at 3 IOPS/GB
- **throughput**: Only gp3 supports this; max 1000 MB/s
- **encrypted**: Always true for production data
- **volumeBindingMode**: Always WaitForFirstConsumer for block storage

---

## Dynamic Resizing Without Pod Downtime

One of the major benefits of CSI is **zero-downtime volume expansion**.

With older storage, resizing required:
1. Delete pod
2. Delete PVC
3. Create new larger PVC
4. Recreate pod

With CSI:
```bash
# Pod is running, serving traffic

# Edit the PVC to request more space
kubectl patch pvc my-data -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

# AWS expands the volume silently
# Node plugin expands the filesystem (resize2fs/xfs_growfs)
# Pod never stops
# Users don't notice anything
```

*Note:* You can only **increase** volume size, never decrease it. Shrinking a volume would risk data loss and is prevented by the driver.

---

## Summary: EBS Design Principles in Kubernetes

**Principle 1: AZ-Local**
- EBS volumes are tied to one AZ for low latency
- Use WaitForFirstConsumer to guarantee pod-volume AZ alignment

**Principle 2: Single-Node Exclusive**
- Only one node can attach an EBS volume
- ReadWriteOnce access mode enforces this
- Multiple pods on same node share the volume

**Principle 3: Performance Predictability**
- Explicitly specify IOPS and throughput
- gp3 gives independent control over both
- Know exactly what you're paying for

**Principle 4: Zero-Downtime Operations**
- Resize volumes while pods run
- No restart needed
- Filesystem automatically expands

**Design Pattern:** For databases, use EBS with:
- `volumeBindingMode: WaitForFirstConsumer`
- `type: io2` with high IOPS
- `reclaimPolicy: Retain`
- Encryption enabled

**Next File:** Chapter 38 File 03 covers AWS EFS, which solves the multi-node sharing problem that EBS cannot address.
