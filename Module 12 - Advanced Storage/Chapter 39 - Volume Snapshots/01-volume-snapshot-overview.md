# Volume Snapshots: Point-in-Time Copies for Data Protection

## The Storage Problem That Snapshots Solve

Modern applications need protection against:

1. **Accidental Deletion** - Developer deletes wrong data
2. **Data Corruption** - Bad query updates corrupts entire database
3. **Testing Scenarios** - Need copy of production data for testing without copying 100GB manually
4. **Compliance Requirements** - Must maintain historical copies for audit trails
5. **Fast Recovery** - Restore from minutes-old snapshot vs recovering from backup tape (hours)

Traditional solutions were painful:

**Old Way (Manual Backups):**
```
1. Backup application takes consistent snapshot
2. Compresses data
3. Uploads to S3 (slow)
4. To restore: Download from S3, decompress, restore
5. Takes hours, needs coordination
6. Expensive storage (separate backup system)
```

**New Way (Kubernetes Snapshots):**
```
1. Create snapshot via Kubernetes API
2. Takes seconds (cloud native)
3. Stored in same region
4. Can restore to pod immediately
5. Pay only for differential storage
```

---

## What Is a VolumeSnapshot?

A VolumeSnapshot is a **point-in-time copy of a persistent volume** that captures the volume's data at a specific moment.

### Key Characteristics

**Crash-Consistent vs Application-Consistent:**
- **Crash-Consistent**: Captures disk state exactly as it is (like power-down snapshot)
  - Good for: Stateless services, web servers
  - Problem: Database might be mid-transaction, leaving inconsistent data
  - Solution: Database must handle crash recovery (like after power loss)

- **Application-Consistent**: Application-aware snapshot (app flushes buffers first)
  - Good for: Databases (PostgreSQL, MySQL), stateful services
  - Requires: Pre-snapshot hooks (quiesce database, flush I/O)
  - More complex but safer

Example: EBS snapshots are crash-consistent. Database inside must have journaling/WAL (Write-Ahead Logging) to recover.

### VolumeSnapshot Architecture

Kubernetes introduces three new Custom Resource Definitions (CRDs):

**1. VolumeSnapshot** - The request/template
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-snapshot
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: my-pvc  # What to snapshot
```

**2. VolumeSnapshotClass** - The provisioner configuration
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete  # Delete AWS snapshot when VolumeSnapshot deleted?
```

**3. VolumeSnapshotContent** - The actual snapshot (auto-created)
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: snapcontent-12345
spec:
  volumeSnapshotRef:
    name: my-snapshot
  snapshotHandle: snap-0987654321  # AWS snapshot ID
```

### Why Three Separate Objects?

**Separation of concerns:**
- **VolumeSnapshot**: "I want a snapshot" (user request)
- **VolumeSnapshotClass**: "Use these settings" (admin configuration)
- **VolumeSnapshotContent**: "Here's the actual snapshot" (system implementation)

Similar to how PVC → PV + StorageClass work.

---

## Snapshot Lifecycle and State Transitions

### Complete Lifecycle

```
1. CREATING
   ├─ VolumeSnapshot created
   ├─ CSI driver calls cloud API
   ├─ "Create snapshot of vol-123"
   └─ AWS begins snapshot process (reads volume data)

2. READY (Ready=True)
   ├─ Snapshot data completely captured
   ├─ Can now be used to restore
   └─ Exists in AWS (even if Kubernetes object deleted)

3. DELETING
   ├─ User deletes VolumeSnapshot
   ├─ CSI driver checks deletionPolicy
   ├─ If deletionPolicy=Delete: calls AWS to delete snapshot
   └─ AWS snapshot deleted (cannot restore anymore)

4. DELETED
   └─ VolumeSnapshot object and AWS snapshot both gone
```

### State Check Command

```bash
# Check snapshot status
kubectl get volumesnapshot

# Output:
# NAME         READYTOUSE   SOURCEPVC    SOURCESIZE   SNAPSHOTCLASS        SNAPSHOTHANDLE        CREATIONTIME
# db-backup    true         my-data      50Gi         ebs-snapshot-class   snap-0123456789abc    2m

# READYTOUSE=true means snapshot is ready to restore from

# Check details
kubectl describe volumesnapshot db-backup

# Output includes:
# Status:
#   Creation Time:  2024-01-15T10:30:00Z
#   Ready To Use:   true
#   Restore Size:   50Gi
```

---

## Snapshot Use Cases: When and Why

### Use Case 1: Backup Before Risky Operations

```yaml
# Before major database migration
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: pre-migration-backup
  labels:
    purpose: migration-safety
    expires: "2024-02-01"
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: postgres-data
```

**Workflow:**
```
1. DBA takes snapshot (backup created)
2. Migration proceeds
3. If migration fails: restore from snapshot
4. If migration succeeds: delete snapshot (saves money)
```

### Use Case 2: Development/Test Data Cloning

```
Production Environment:
  Database size: 500GB
  Daily snapshot: 1GB differential

Development Environment:
  Need same production data for debugging
  Can't afford 500GB copy
  Solution: Clone from snapshot (only changed data copied)
```

**Problem without snapshots:** Copying 500GB takes 30 minutes + storage costs.
**Problem with snapshots:** Clone a snapshot in 5 minutes, pay only for differential.

### Use Case 3: Compliance and Audit Trails

```
# Financial services requirement: Keep data snapshots for 7 years
# Law: Must prove data existed on specific date

# Daily automated snapshots
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-compliance-snapshot
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: snapshot-creator
            image: snapshot-tool:latest
            command: ["./create-snapshot.sh", "ledger-data", "compliance-backup"]
```

Snapshots create immutable, timestamped records for audit.

### Use Case 4: Ransomware Protection

```
Ransomware Attack Timeline:
10:00 AM - Attacker gains access
10:15 AM - Ransomware begins encryption
10:30 AM - Most recent backup corrupted too (ransomware infected backup system)
Problem: No clean copy to restore from

Solution: Snapshots with immutable retention
- Snapshots created hourly
- Set AWS to prevent deletion for 30 days (immutable)
- Ransomware can't delete snapshots (cloud prevents it)
- Can restore from 1-hour-old snapshot after incident
```

### Use Case 5: Cost Optimization for Multi-Environment

```
Scenario: Need 5 environments (prod, staging, qa, dev, feature-test)
Without snapshots: 5 × 100GB = 500GB storage cost

With snapshots:
  Production: 100GB (primary)
  Staging: Clone from snapshot (differential only ~5GB)
  QA: Clone from snapshot (differential ~3GB)
  Dev: Clone from snapshot (differential ~2GB)
  Feature-test: Clone from snapshot (differential ~1GB)
  
Total: ~111GB instead of 500GB (78% cost savings!)
```

---

## Snapshot Architecture: How CSI Drivers Implement Snapshots

### CSI Snapshot Controller Components

Kubernetes uses several controller pods for snapshot management:

**1. Snapshot Controller** (cluster-level)
```
Watches: VolumeSnapshot and VolumeSnapshotContent objects
Responsibility: Reconciliation (matching snapshots to content)
Location: kube-system namespace
```

**2. CSI Snapshot Sidecar** (per storage driver)
```
Watches: VolumeSnapshotContent objects
Responsibility: Call CSI driver APIs
Location: Same pod as CSI controller driver
Operation: "Create snapshot via driver", "Delete snapshot via driver"
```

**3. CSI Driver Pods** (controller + node DaemonSet)
```
Controller pod: Handles CreateSnapshot, DeleteSnapshot, ListSnapshots (API calls)
Node plugin: NOT involved (snapshots are cloud-level, not OS-level)
```

### Flow: What Happens When You Create a VolumeSnapshot

```
Step 1: User Creates VolumeSnapshot Object
  └─ kubectl apply -f snapshot.yaml

Step 2: API Server Validates
  └─ Checks VolumeSnapshotClass exists

Step 3: Snapshot Controller Sees New VolumeSnapshot
  ├─ Creates VolumeSnapshotContent object (auto-created)
  ├─ Sets spec.source.persistentVolumeClaimName → looks up PVC
  ├─ Finds PVC → gets PV → gets volume info
  └─ Marks VolumeSnapshotContent for creation

Step 4: CSI Sidecar Detects New VolumeSnapshotContent
  ├─ Reads spec (volumeSnapshotClassName, source PVC, etc)
  ├─ Calls CSI driver: CreateSnapshot(volumeId, snapshotName)
  └─ Waits for response

Step 5: CSI Driver Calls Cloud API
  ├─ Driver calls AWS EC2 API
  ├─ "CreateSnapshot(VolumeId=vol-abc123)"
  ├─ AWS begins snapshot process
  └─ Returns SnapshotId=snap-xyz789

Step 6: Status Updated
  ├─ VolumeSnapshotContent.status.snapshotHandle = snap-xyz789
  ├─ VolumeSnapshot.status.ready = true
  └─ VolumeSnapshot.status.restoreSize = 50Gi

Step 7: User Can Now Restore
  ├─ Create new PVC with dataSource.name=my-snapshot
  └─ New volume created from snapshot data
```

---

## Snapshot Limitations and Constraints

### Limitation 1: Crash-Consistent Only (for EBS)

```
Problem: Database in middle of transaction

Timeline:
10:00:00 - Snapshot starts
10:00:15 - Database writes record A (partial)
10:00:30 - Snapshot completes

Result: Snapshot has partial record A (corrupted state)

Solution: 
1. Database must handle crash recovery (journaling/WAL)
2. Or use pre-snapshot hooks to quiesce database
```

### Limitation 2: Same Storage Class

```yaml
# Cannot do this:
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-data
spec:
  storageClassName: gp3-cheaper  # Different class!
  dataSource:
    name: my-snapshot
    kind: VolumeSnapshot
  # ERROR: Snapshot from io2 cannot restore to gp3
```

Snapshots preserve storage type information.

### Limitation 3: AZ Binding (for EBS)

```
Scenario: Snapshot in us-east-1a, need volume in us-east-1b

Result: Cannot restore directly to different AZ

Solution: 
- Copy snapshot to target AZ first (cross-AZ copy)
- Then restore in target AZ
```

### Limitation 4: Size Cannot Decrease

```yaml
# Original snapshot: 50Gi
# Try to restore as: 10Gi

Result: ERROR - Cannot shrink snapshot

# Why: Would lose data

# Allowed: Expand snapshot to larger size (e.g., 100Gi)
```

---

## Snapshot Storage and Cost Implications

### How Snapshot Storage is Charged

**EBS Snapshots use Incremental Snapshots:**

```
Day 1: Take snapshot of 50GB volume
  └─ Snapshot storage: 50GB charged

Day 2: 1GB changed on volume, take snapshot 2
  └─ Snapshot 2 stores: Only 1GB delta (not 50GB again)
  └─ Total storage: 50GB + 1GB = 51GB charged

Day 3: 500MB changed, take snapshot 3
  └─ Snapshot 3 stores: Only 500MB delta
  └─ Total storage: 50GB + 1GB + 500MB ≈ 51.5GB charged

Cost: ~$0.025 per GB per month in us-east-1
  └─ 51.5GB × $0.025 = ~$1.29/month for snapshots
```

**Problem: Chain of Snapshots**

```
If you delete snapshot 2 (middle of chain):
  ├─ Snapshot 1 has days 1 data
  ├─ Snapshot 2 DELETED
  ├─ Snapshot 3 still needs snapshot 2 to restore
  └─ AWS merges data: keeps all 3 snapshots' data alive!

Solution: Only delete oldest snapshots, not middle ones
```

### Cost Optimization

```yaml
# Bad: Keep all snapshots forever
apiVersion: batch/v1
kind: CronJob
metadata:
  name: snapshot-daily
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: snapshot-creator
            command: ["kubectl", "apply", "-f", "snapshot.yaml"]
  # Result: 365 snapshots per year, storage grows unbounded

# Good: Snapshot with retention policy
apiVersion: batch/v1
kind: CronJob
metadata:
  name: snapshot-daily-with-cleanup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: snapshot-creator
            command: ["/scripts/create-and-cleanup.sh"]
            # Script: Create new snapshot, delete snapshots >30 days old
```

---

## Snapshot Provisioners and Classes

### AWS EBS Snapshots

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  tags: "Environment=production,Team=database"  # AWS tags
```

### AWS EFS Snapshots (NFS)

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: efs-snapshot-class
driver: efs.csi.aws.com
deletionPolicy: Delete
# Note: EFS snapshots different (backup service, not native snapshots)
```

### Custom Snapshot Policies

```yaml
# Development environment: Auto-delete after 7 days
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: dev-snapshot-temp
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  retention-days: "7"

# Production environment: Keep for 90 days
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: prod-snapshot-long-term
driver: ebs.csi.aws.com
deletionPolicy: Retain  # Don't auto-delete
parameters:
  retention-days: "90"
```

---

## Pre-Snapshot Hooks for Application Consistency

### Why Pre-Snapshot Hooks Matter

```
Database State Before Hook:
  - Write buffer has pending transactions
  - In-memory cache not flushed to disk
  - Transaction log not synchronized

Taking snapshot WITHOUT hook:
  - Snapshot captures inconsistent state
  - Restore leads to data corruption

With Pre-Snapshot Hook:
  1. Hook runs: FLUSH TABLES, COMMIT, fsync()
  2. All in-memory data written to disk
  3. Database in known-good state
  4. Then snapshot taken
  5. Restore gives consistent database
```

### Example: PostgreSQL Pre-Snapshot Hook

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-with-snapshot-hook
spec:
  containers:
  - name: postgres
    image: postgres:15
    env:
    - name: PRE_SNAPSHOT_HOOK
      value: |
        psql -U postgres -c "SELECT pg_start_backup('backup');"
        sleep 10
        psql -U postgres -c "SELECT pg_stop_backup();"
  lifecycle:
    preStop:  # Before pod shutdown
      exec:
        command: ["/bin/sh", "-c", "$PRE_SNAPSHOT_HOOK"]
```

---

## Troubleshooting: Common Snapshot Issues

### Issue 1: VolumeSnapshot Stuck in Pending

```bash
kubectl get volumesnapshot
# NAME    READYTOUSE   SOURCEPVC    STATUS
# snap-1  false        my-pvc       Unknown (pending forever!)

# Debug:
kubectl describe volumesnapshot snap-1

# Check if CSI driver running
kubectl get pods -n kube-system | grep snapshot

# Check CSI driver logs
kubectl logs -n kube-system deployment/csi-snapshotter-aws
# Error: "provisioner not installed"

# Fix: Install snapshot controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
```

### Issue 2: Cannot Restore from Snapshot

```bash
# Create PVC from snapshot
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-data
spec:
  storageClassName: ebs-gp3
  dataSource:
    name: my-snapshot
    kind: VolumeSnapshot
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 50Gi
EOF

# Error: "VolumeSnapshot not found"

# Debug: Check if VolumeSnapshot exists and is ready
kubectl get volumesnapshot my-snapshot
# If not ready: wait a few minutes

# If wrong namespace:
kubectl get volumesnapshot -n production  # Correct namespace
```

### Issue 3: Snapshots Consuming Too Much Storage

```bash
# Check all snapshots
kubectl get volumesnapshot --all-namespaces

# Find old snapshots
kubectl get volumesnapshot --all-namespaces -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.creationTime}{"\n"}{end}' | sort -k2

# Delete old ones
kubectl delete volumesnapshot old-snapshot-from-2024-01-01

# Verify AWS snapshot deleted
aws ec2 describe-snapshots --filters Name=status,Values=completed | grep -c snap
```

---

## Summary: Snapshots at a Glance

**What:** Point-in-time copies of persistent volumes

**Why:** 
- Recovery from accidents
- Testing with production data
- Compliance and audit trails
- Ransomware protection

**How:**
- Create via VolumeSnapshot CRD
- CSI driver handles cloud API calls
- Stored as cloud snapshots (S3 for EBS)

**Cost:**
- Incremental storage (~$0.025/GB/month)
- Only pay for changed data

**Limitations:**
- Crash-consistent (not app-consistent by default)
- Same storage class required
- Cannot shrink
- AZ-bound (for EBS)

**Best Practice:**
- Use snapshots for backups before risky operations
- Set retention policies (don't keep forever)
- Pre-snapshot hooks for databases
- Monitor snapshot storage costs

**Next File:** Chapter 39 File 02 covers creating snapshots (manual and automated).
