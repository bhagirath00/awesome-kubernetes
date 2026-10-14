# What is CSI? Understanding the Container Storage Interface

## The Storage Problem That Kubernetes Faced

In the early days of Kubernetes (2014-2016), storage provisioning was a fundamental architectural challenge. Kubernetes needed to work across multiple cloud providers—AWS, Google Cloud, Azure, and others—each with their own block storage APIs and file system services.

The solution at the time was to **embed storage provider code directly into the Kubernetes core**. This meant that AWS EBS logic, Google Persistent Disk logic, and Azure storage logic all lived in the main Kubernetes source code repository. When you compiled Kubernetes, all of these drivers compiled together.

### Why This Became Problematic

As Kubernetes matured and more storage vendors wanted to integrate with it, several serious architectural problems emerged:

**1. Release Cycle Coupling**
When Amazon released a new EBS feature (like `gp3` volume types), users couldn't use it until the next Kubernetes release—which could be 3 months away. This created artificial delays between storage innovation and adoption.

**2. Tight Coupling Risk**
If a bug existed in the Google storage driver, it could crash the entire Kubernetes control plane's storage controller manager. A single vendor's bad code could destabilize the entire cluster. This violated the principle of separation of concerns.

**3. Maintenance Burden**
The Kubernetes core team was responsible for maintaining dozens of storage drivers for vendors they didn't work for. This placed an enormous burden on the Kubernetes project and diluted focus on core features.

**4. Bloat**
Not all clusters needed all drivers. Users running on AWS didn't need Azure's code compiled into their Kubernetes binaries, yet it was there, increasing binary size and complexity.

**5. Innovation Speed**
Storage vendors couldn't deploy features at their own pace. They were locked into Kubernetes release cycles, slowing down the entire ecosystem's evolution.

---

## The Container Storage Interface (CSI) Solution

In 2018-2019, the Kubernetes community, working with storage vendors, created the **Container Storage Interface (CSI)** standard.

The core insight: **Extract storage code from Kubernetes core and run it as external plugins that communicate with Kubernetes via a well-defined interface.**

This was a paradigm shift. Instead of embedding storage logic, Kubernetes would only define the interface that storage systems must implement. Each vendor would maintain their own out-of-tree driver.

### The Architectural Shift

**Before CSI (In-Tree Model):**
```
Kubernetes Control Plane (Core)
├─ EBS Storage Controller
├─ GCP PD Storage Controller  
├─ Azure Storage Controller
└─ Lots of vendor code...
```
*All coupled together, released as one unit.*

**After CSI (Out-of-Tree Model):**
```
Kubernetes Control Plane (Slim Core)
└─ CSI Interface Standard
     │
     ├─ AWS EBS CSI Driver (AWS maintains)
     ├─ Google PD CSI Driver (Google maintains)
     ├─ Azure Disk CSI Driver (Microsoft maintains)
     └─ Community CSI Drivers (Ceph, NetApp, etc.)
```
*Each vendor maintains their own driver independently.*

---

## How CSI Works: The Architecture

### The CSI Standard

CSI is a specification (not Kubernetes-specific) that defines how containerized storage systems should expose their capabilities to container orchestrators. The spec defines:

- **Create/Delete volumes**
- **Attach/Detach volumes to nodes**
- **Mount/Unmount volumes into pods**
- **Take snapshots**
- **Clone volumes**
- **Expand volumes**

Kubernetes implements the orchestrator side, storage vendors implement the driver side.

### The Two Roles in a CSI Driver

Every CSI driver deployment consists of two distinct components with different responsibilities:

**1. The Controller Plugin (Controller Service)**
Typically runs as a StatefulSet or Deployment with 1-3 replicas in the control plane region.

*Responsibilities:* High-level cloud operations
- Talks to the cloud provider's API (AWS EC2 API, Google GCE API, etc.)
- Creates new volumes when PVCs are created
- Deletes volumes when PVCs are deleted
- Attaches volumes to specific EC2 instances/compute nodes
- Detaches volumes
- Takes snapshots
- Handles volume expansion requests at the cloud layer

The controller never runs on individual worker nodes. It's centralized and uses your cloud provider credentials (injected via IAM roles or secrets).

**2. The Node Plugin (Node Service)**
Runs as a DaemonSet on every worker node in the cluster.

*Responsibilities:* Local OS-level operations
- Formats the raw disk (e.g., creates ext4 or XFS filesystems)
- Mounts the formatted disk to a path in the pod's namespace
- Unmounts the disk when the pod terminates
- Reports node-level storage metrics and health
- Handles symlink management for device discovery

The node plugin has direct access to the OS and can execute commands like `mkfs.ext4`, `mount`, `umount`.

---

## The Provisioning Workflow: From PVC to Running Pod

Understanding the full workflow helps explain why CSI has both controller and node plugins:

### Step 1: Developer Creates PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  storageClassName: ebs-gp3
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

The developer isn't thinking about AWS. They're just declaring: "I need 50GB of storage with the characteristics of the `ebs-gp3` class."

### Step 2: Kubernetes Controller Observes PVC
The CSI provisioner sidecar (running in the controller plugin) watches for new PVCs. It sees the `storageClassName: ebs-gp3` reference.

The provisioner looks up the `StorageClass` to understand:
- Which CSI driver to use (provisioner field)
- What parameters to pass (type: gp3, iops: 3000, etc.)
- Reclaim policy and other behaviors

### Step 3: CSI Driver Calls Cloud API
The controller plugin's driver calls the cloud provider's API:

**For EBS:** "AWS, please create a 50GB gp3 volume in the same AZ as the node where this will be used."

The AWS API returns: `vol-0123456789abcdef0` (the volume ID).

### Step 4: Kubernetes Creates PersistentVolume (PV)
The provisioner automatically creates a PV object that tracks this actual AWS volume:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pvc-abc123
spec:
  capacity:
    storage: 50Gi
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0123456789abcdef0
  persistentVolumeReclaimPolicy: Delete
```

### Step 5: PVC Binds to PV
Kubernetes matches the PVC to the PV. The developer's PVC now has `status: Bound`.

### Step 6: Pod Scheduled to a Node
The deployment/pod is scheduled by the kube-scheduler to a specific node (e.g., worker-1 in us-east-1a).

### Step 7: CSI Controller Attaches Volume to Node
Before the pod can run, the volume must be attached to the EC2 instance.

The CSI external-attacher sidecar (running in the controller plugin) calls AWS:
"Attach vol-0123456789abcdef0 to instance i-0987654321fedcba0 as device /dev/sdf"

AWS processes this and device becomes visible on the node's OS.

### Step 8: Node Plugin Formats and Mounts
The CSI node plugin DaemonSet running on worker-1 detects the new device.

It executes OS commands:
```bash
# Format the raw device with ext4
mkfs.ext4 /dev/nvme1n1

# Mount it to the pod's mount point
mount /dev/nvme1n1 /var/lib/kubelet/pods/pod-abc123/volumes/kubernetes.io~csi/app-data/mount
```

### Step 9: kubelet Mounts into Container
Now kubelet mounts the pod's filesystem with the persistent volume at the requested path (e.g., `/data`).

The container sees `/data` as the 50GB formatted volume.

---

## Why This Architecture Matters: Separation of Concerns

The split between controller and node plugins embodies a critical architectural principle:

**Controllers need cloud credentials and make API calls → Run centralized with proper RBAC**
**Node plugins need OS access and run privileged → Run locally on each node**

This separation provides:
- **Security**: Cloud credentials stay on controller, not on every worker node
- **Scalability**: Node plugins are lightweight and distributed
- **Reliability**: A node plugin crash doesn't affect the controller
- **Flexibility**: Different storage systems have different attachment mechanisms

---

## In-Tree vs CSI: Why Migration Matters

Many Kubernetes users still have legacy StorageClasses using in-tree provisioners:

```yaml
# DEPRECATED IN-TREE (Don't use)
provisioner: kubernetes.io/aws-ebs
```

Kubernetes is **removing all in-tree drivers**. In Kubernetes 1.27+, they're gone or deprecated.

**Why the migration?**
- In-tree code lives in Kubernetes source
- Bug fixes require Kubernetes release
- Features wait for release cycles
- Vendor innovation is blocked

**CSI approach:**
- Driver code lives in vendor repo
- Bug fixes deployed immediately (no K8s wait)
- Features released at vendor pace
- True independence

---

## The CSI Ecosystem Today

CSI is now industry-standard across Kubernetes distributions:
- **AWS EBS CSI Driver** (official AWS support)
- **AWS EFS CSI Driver** (for NFS needs)
- **Google Persistent Disk CSI**
- **Azure Disk CSI**
- **NetApp Trident CSI** (on-prem enterprise storage)
- **Ceph RBD CSI** (open-source distributed storage)
- **Etc.** (100+ drivers exist)

This ecosystem exists because CSI decoupled storage from core Kubernetes, allowing vendors to innovate freely.

---

## Summary: Why CSI Changed Everything

**The Problem:** Storage code embedded in Kubernetes core created coupling, slowed innovation, and created maintenance burden.

**The Solution:** CSI standard defines the interface; vendors implement drivers independently.

**The Benefit:** 
- Faster feature releases
- Better separation of concerns
- Vendor independence
- Stable core Kubernetes
- True plugin architecture

**Next File:** Chapter 38 File 02 dives deep into AWS EBS CSI specifically, explaining AZ-binding, volume attachment mechanics, and the critical `WaitForFirstConsumer` binding mode.
