# Restoring from Snapshots: Cloning and Recovery Workflows

## The Restore Problem Snapshots Solve

Restoring data traditionally was painful:

**Old Approach (Manual Restore):**
```
1. Backup administrator notified
2. Backup location identified (tape drive, S3, NAS)
3. Data extracted from backup format
4. Transferred over network
5. Restored to new storage
6. Pod configured to use new storage
Total time: 30 minutes to 2 hours
```

**With Snapshots:**
```
1. Identify snapshot
2. Create PVC from snapshot
3. Pod immediately uses restored data
4. Within 1-2 minutes
```

---

## Restore Architecture: How It Works Under the Hood

### Three-Step Restore Process

```
Step 1: Create PVC with dataSource pointing to VolumeSnapshot
   ├─ User specifies: "I want a volume from snapshot X"
   └─ Kubernetes reads: volumeSnapshotClassName and snapshotHandle

Step 2: CSI Driver Detects New PVC Needing Restoration
   ├─ Provisioner sees: PVC with dataSource.name = my-snapshot
   ├─ Looks up VolumeSnapshot my-snapshot
   ├─ Extracts AWS snapshotHandle (snap-0123456789)
   └─ Calls AWS API: CreateVolume(SnapshotId=snap-0123456789)

Step 3: AWS Creates New Volume from Snapshot
   ├─ AWS reads snapshot data (snap-0123456789)
   ├─ Creates new EBS volume
   ├─ Populates volume with snapshot data
   ├─ Returns volumeId (vol-9876543210)
   └─ Kubernetes binds this volume to PVC

Step 4: Pod Mounts and Accesses Data
   ├─ Pod spec references restored PVC
   ├─ kubelet attaches volume to node
   ├─ Filesystem already present (copied from snapshot)
   ├─ Pod starts immediately with data
   └─ No format, no recovery needed
```

### Why This Works So Quickly

**Traditional Restore:**
```
Backup → Compress → Transfer → Decompress → Format → Copy → Done
(Each step adds time)
```

**Snapshot Restore:**
```
Snapshot → Create Volume → Done
(AWS does the copy at cloud speed)
```

---

## Basic Restore: From Snapshot to New PVC

### The Simplest Restore Pattern

**Scenario:** Database crashed, need to restore from yesterday's snapshot.

**Step 1: Identify the snapshot to restore from**

```bash
# List all snapshots
kubectl get volumesnapshot -n production

# Output:
# NAME                          READYTOUSE   SOURCEPVC        CREATIONTIME
# postgres-backup-2024-01-15    true         postgres-data    2024-01-15T02:00:00Z
# postgres-backup-2024-01-14    true         postgres-data    2024-01-14T02:00:00Z
# postgres-backup-2024-01-13    true         postgres-data    2024-01-13T02:00:00Z

# Choose: postgres-backup-2024-01-15 (most recent)
```

**Step 2: Create new PVC from snapshot**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-restored
  namespace: production
spec:
  storageClassName: ebs-gp3
  dataSource:
    name: postgres-backup-2024-01-15      # Snapshot name
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi  # Must match snapshot size (or larger)
```

**Step 3: Wait for PVC to be bound**

```bash
# Watch PVC creation
kubectl get pvc postgres-data-restored -n production -w

# Initially:
# NAME                        STATUS    VOLUME   CAPACITY
# postgres-data-restored      Pending   -        -

# After 30-60 seconds:
# NAME                        STATUS    VOLUME                CAPACITY
# postgres-data-restored      Bound     pvc-abc123def456      50Gi

# PVC now ready to use
```

**Step 4: Update pod to use restored PVC**

```yaml
# Old pod (using crashed data):
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  containers:
  - name: postgres
    image: postgres:15
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: postgres-data  # OLD PVC

# New pod (using restored data):
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  containers:
  - name: postgres
    image: postgres:15
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: postgres-data-restored  # NEW restored PVC
```

**Step 5: Update service to route to new pod**

```bash
# If using Deployment:
kubectl set env deployment/postgres DATABASE_PVC=postgres-data-restored -n production

# Pod automatically restarts with new volume
# Database starts with yesterday's data

# Verify data restored
kubectl exec -it deployment/postgres -n production -- psql -U postgres -c "SELECT COUNT(*) FROM orders;"
```

---

## Advanced Restore: Sizing and Performance Considerations

### The Sizing Constraint: Cannot Shrink, Can Expand

**Constraint 1: Snapshot-to-PVC Size Matching**

```yaml
# Original snapshot: 50Gi
# Option 1: Restore to exactly 50Gi
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-exact
spec:
  dataSource:
    name: my-snapshot
    kind: VolumeSnapshot
  resources:
    requests:
      storage: 50Gi  # Matches snapshot
  storageClassName: ebs-gp3

---
# Option 2: Restore to larger size (expansion)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-expanded
spec:
  dataSource:
    name: my-snapshot
    kind: VolumeSnapshot
  resources:
    requests:
      storage: 100Gi  # Larger than snapshot (creates 100Gi volume, fills first 50Gi with snapshot data)
  storageClassName: ebs-gp3

---
# Option 3: Restore to smaller size - ERROR!
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-shrink
spec:
  dataSource:
    name: my-snapshot
    kind: VolumeSnapshot
  resources:
    requests:
      storage: 20Gi  # ERROR: Cannot shrink!
  storageClassName: ebs-gp3
```

**Why No Shrinking:**
- Data in snapshot takes 50GB
- Cannot fit 50GB into 20GB volume
- Would lose data
- Kubernetes prevents this

**When to Expand:**

```
Scenario: Snapshot from production (50GB), restoring for testing
  ├─ Database has grown since snapshot
  ├─ Testing needs more space (projecting growth)
  ├─ Restore as 100GB (data starts at 50GB, room for growth)
  └─ Filesystem can grow without new allocation
```

### Performance Considerations During Restore

**Restore Time Depends On:**

1. **Snapshot size** (larger = longer)
2. **Network speed** (usually instant in same region)
3. **IOPS/throughput** of destination volume (gp3 vs io2)

**Typical Timings:**

```
10GB snapshot → 15 seconds restore time
50GB snapshot → 60 seconds restore time
100GB snapshot → 120 seconds restore time

After restore, volume accessible immediately
(AWS doesn't wait for full data transfer to complete)
```

**Performance After Restore:**

```
First reads (warm-up):
  ├─ Data not yet in AWS cache
  ├─ Slight performance hit (10-20% slower)
  └─ Lasts 1-5 minutes

After warm-up:
  ├─ Data cached in AWS infrastructure
  └─ Performance = new volume (no snapshot overhead)
```

---

## Cloning Pattern: Create Development Copies from Production

### The Problem: Multi-Environment Testing

```
Scenario: Need production data in development for debugging
  Without snapshots:
    ├─ Manual backup of 500GB production data
    ├─ Transfer to dev environment (2 hours)
    ├─ Storage cost: 500GB × 2 environments = 1TB
    └─ Total cost: High, time-consuming

  With snapshots:
    ├─ Production: 500GB (primary)
    ├─ Dev clone: From snapshot (only differential cost)
    ├─ Staging clone: From snapshot (only differential cost)
    └─ Total storage: 500GB + deltas (much cheaper)
```

### Clone Pattern: Same Region

```yaml
---
# Step 1: Snapshot production database
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: prod-db-clone-source
  namespace: production
spec:
  volumeSnapshotClassName: prod-snapshot-compliance
  source:
    persistentVolumeClaimName: postgres-production

---
# Step 2: Wait for snapshot to be ready (check READYTOUSE=true)

---
# Step 3: Create dev PVC from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-development
  namespace: development
spec:
  storageClassName: dev-snapshot-temp
  dataSource:
    name: prod-db-clone-source
    kind: VolumeSnapshot
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Gi  # Matches production size

---
# Step 4: Dev pod uses cloned data
apiVersion: v1
kind: Pod
metadata:
  name: postgres-dev
  namespace: development
spec:
  containers:
  - name: postgres
    image: postgres:15
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: postgres-development
```

**Cost Comparison:**

```
Without snapshots:
  Production DB: 500GB × $0.10/GB/month = $50
  Dev DB copy: 500GB × $0.10/GB/month = $50
  Staging copy: 500GB × $0.10/GB/month = $50
  Total: $150/month

With snapshots:
  Production DB: 500GB × $0.10/GB/month = $50
  Production snapshot: 500GB × $0.025/GB/month = $12.50
  Dev clone (incremental): ~5GB × $0.10/GB/month = $0.50
  Staging clone (incremental): ~5GB × $0.10/GB/month = $0.50
  Total: ~$63.50/month (58% savings!)
```

---

## Cross-AZ Restore (EBS Constraint Workaround)

### The Problem: EBS Snapshots Are Region-Bound

```
Scenario: Snapshot in us-east-1a, need to restore in us-east-1b

Direct restore fails:
  ├─ Snapshot tied to us-east-1a
  ├─ Cannot directly create volume in us-east-1b
  └─ Error: "Cannot restore snapshot across AZs"

Solution: Copy snapshot to target AZ first
```

### Cross-AZ Restore Workflow

```
Step 1: Create snapshot in us-east-1a
  └─ Standard snapshot (in AZ-a)

Step 2: Copy snapshot to us-east-1b
  ├─ AWS command: "Copy snapshot to AZ-b"
  ├─ Creates new snapshot (in AZ-b)
  └─ Costs: Additional storage for copied snapshot

Step 3: Restore from copied snapshot in AZ-b
  └─ Create PVC with AZ-b snapshot
```

### Implementation

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: snapshot-copier
  namespace: kube-system
spec:
  serviceAccountName: snapshot-copier
  containers:
  - name: copier
    image: amazon/aws-cli:latest
    command:
    - /bin/sh
    - -c
    - |
      # Get snapshot from us-east-1a
      SOURCE_SNAPSHOT=$(aws ec2 describe-snapshots \
        --filters "Name=tag:kubernetes.io/created-for/pvc/name,Values=postgres-data" \
        --region us-east-1 \
        --query 'Snapshots[0].SnapshotId' \
        --output text)
      
      echo "Copying snapshot $SOURCE_SNAPSHOT to us-east-1b"
      
      # Copy snapshot (cross-AZ)
      DEST_SNAPSHOT=$(aws ec2 copy-snapshot \
        --source-region us-east-1 \
        --source-snapshot-id $SOURCE_SNAPSHOT \
        --destination-region us-east-1 \
        --query 'SnapshotId' \
        --output text)
      
      echo "Copied to: $DEST_SNAPSHOT"
      
      # Tag for identification
      aws ec2 create-tags \
        --region us-east-1 \
        --resources $DEST_SNAPSHOT \
        --tags Key=CrossAZCopy,Value=true Key=SourceSnapshot,Value=$SOURCE_SNAPSHOT
      
      echo "Copy complete - snapshot available in both AZs"
  restartPolicy: Never
```

---

## Restore Scenarios: Real-World Examples

### Scenario 1: Database Corruption Recovery

```
Timeline:
10:00 AM - Bad query corrupts customer data
10:05 AM - Corruption discovered
10:10 AM - Snapshot from 9:00 AM (1 hour old data) identified
10:15 AM - Restore started
10:17 AM - New PVC bound
10:20 AM - Application using restored data
10:25 AM - Full recovery complete
Total downtime: 25 minutes (could be 5 with pre-planning)
```

**Recovery Steps:**

```bash
# 1. Stop current application
kubectl scale deployment/app --replicas=0 -n production

# 2. Identify good snapshot (from 1 hour ago)
kubectl get volumesnapshot -n production --sort-by=.metadata.creationTimestamp | head -5

# 3. Restore to new PVC
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-recovered
  namespace: production
spec:
  storageClassName: ebs-io2
  dataSource:
    name: app-backup-2024-01-15-09  # 1 hour old
    kind: VolumeSnapshot
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
EOF

# 4. Wait for PVC to be ready
kubectl wait --for=condition=bound pvc/app-data-recovered -n production

# 5. Point deployment to recovered PVC
kubectl patch deployment/app -n production --type merge -p \
  '{"spec":{"template":{"spec":{"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"app-data-recovered"}}]}}}}'

# 6. Start application
kubectl scale deployment/app --replicas=3 -n production

# 7. Verify recovery
kubectl logs deployment/app -n production | head -20
```

### Scenario 2: Ransomware Attack Recovery

```
Attack Timeline:
9:00 AM - Hourly snapshot created (clean state)
10:00 AM - Ransomware encrypts data
10:05 AM - Attack discovered
10:30 AM - Snapshot from 9:00 AM identified (unencrypted)
10:35 AM - Restore started
10:40 AM - New volume ready
10:45 AM - Application running on clean data
Recovery time: 45 minutes
Data loss: Only 45 minutes of transactions after last snapshot
```

**Key:** Snapshots stored in AWS (ransomware can't encrypt them if they delete on-prem volumes).

### Scenario 3: Testing New Application Version

```
Scenario: Want to test database migration (v2.4 → v3.0) safely

Process:
1. Snapshot current production database
2. Restore snapshot to staging environment
3. Run v3.0 code against staging data
4. Test migration (data structure changes, etc.)
5. If test succeeds: Apply to production with downtime
6. If test fails: No impact on production (staging is disposable)
```

**Workflow:**

```yaml
---
# Step 1: Snapshot production
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: pretest-v3.0-migration
  namespace: production
spec:
  volumeSnapshotClassName: prod-snapshot-compliance
  source:
    persistentVolumeClaimName: postgres-production

---
# Step 2: After snapshot is ready (READYTOUSE=true), restore to staging
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-v3-test
  namespace: staging
spec:
  storageClassName: staging-snapshot-30d
  dataSource:
    name: pretest-v3.0-migration
    kind: VolumeSnapshot
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi  # Ensure enough space for migration

---
# Step 3: Staging pod tests new version
apiVersion: v1
kind: Pod
metadata:
  name: postgres-v3-test
  namespace: staging
spec:
  containers:
  - name: postgres-v3
    image: postgres:15  # v3.0 image
    env:
    - name: POSTGRES_VERSION_UPGRADE
      value: "true"
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: postgres-v3-test
```

---

## Restore Validation and Testing

### Post-Restore Verification Checklist

```bash
# 1. PVC Status
kubectl get pvc restored-pvc -n production

# 2. Volume Binding
kubectl describe pvc restored-pvc -n production | grep Events

# 3. Pod Mounting
kubectl get pods -n production -o jsonpath='{.items[*].spec.volumes[*].persistentVolumeClaim.claimName}'

# 4. Data Access
kubectl exec deployment/app -n production -- ls -la /data

# 5. Data Integrity Check
kubectl exec deployment/app -n production -- sha256sum /data/critical-file.txt
# Compare with production snapshot

# 6. Application Logs
kubectl logs deployment/app -n production | grep -i error

# 7. Database Consistency (for stateful apps)
kubectl exec deployment/postgres -n production -- psql -U postgres -c "PRAGMA integrity_check;"
```

### Automated Restore Testing

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: snapshot-restore-test
  namespace: production
spec:
  schedule: "0 3 * * 0"  # Weekly on Sunday 3 AM
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-tester
          containers:
          - name: test-restore
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              echo "Testing snapshot restore process..."
              
              # Get latest snapshot
              LATEST_SNAPSHOT=$(kubectl get volumesnapshot -n production \
                -o jsonpath='{.items[-1].metadata.name}')
              
              echo "Testing with snapshot: $LATEST_SNAPSHOT"
              
              # Create test PVC from snapshot
              TEST_PVC="test-restore-$(date +%s)"
              kubectl apply -f - <<EOF
              apiVersion: v1
              kind: PersistentVolumeClaim
              metadata:
                name: $TEST_PVC
                namespace: production
                labels:
                  test: restore-validation
              spec:
                storageClassName: ebs-gp3
                dataSource:
                  name: $LATEST_SNAPSHOT
                  kind: VolumeSnapshot
                accessModes:
                  - ReadWriteOnce
                resources:
                  requests:
                    storage: 100Gi
              EOF
              
              # Wait for PVC to be bound
              kubectl wait --for=condition=bound pvc/$TEST_PVC -n production --timeout=10m
              
              # Test pod reads from restored volume
              kubectl run -n production test-restore-pod --image=alpine --rm -it --restart=Never -- \
                sh -c "mount | grep /mnt/test && ls -la /mnt/test && echo 'RESTORE TEST PASSED'"
              
              # Cleanup
              kubectl delete pvc $TEST_PVC -n production
              
              echo "Restore validation complete"
          restartPolicy: OnFailure
```

---

## Common Restore Issues and Solutions

### Issue 1: PVC Stuck in Pending After Restore

```bash
kubectl describe pvc restored-pvc

# Error: "waiting for first consumer"

# Cause: volumeBindingMode=WaitForFirstConsumer
# Solution: Pod must be scheduled first

# Fix: Create pod that uses PVC
kubectl apply -f pod.yaml

# Then PVC will bind
```

### Issue 2: Restored Data Larger Than Expected

```bash
# Symptom: Restored 50GB from snapshot, but filesystem shows 100GB used

# Cause: Restored to larger PVC size (e.g., 100Gi from 50Gi snapshot)
# Solution: Grow filesystem in container

kubectl exec pod-name -- resize2fs /dev/nvme1n1
# or
kubectl exec pod-name -- xfs_growfs /data
```

### Issue 3: Slow Performance After Restore

```bash
# Symptom: Restored data slower than expected

# Cause: AWS cache warm-up period (1-5 minutes after restore)

# Solution: Wait or pre-warm by reading data
kubectl exec pod-name -- dd if=/dev/nvme1n1 of=/dev/null bs=1M

# After warm-up, performance returns to normal
```

---

## Summary: Restoring from Snapshots

**Basic Restore:**
- Create PVC with dataSource pointing to VolumeSnapshot
- Specify storageClassName and size (must be ≥ snapshot size)
- Pod immediately has restored data

**Cloning Pattern:**
- Restore same snapshot to multiple PVCs (dev, staging, prod)
- Cost-effective (only differential storage charged)
- Ideal for multi-environment setups

**Size Constraints:**
- Cannot shrink (would lose data)
- Can expand (adds unallocated space)
- Filesystem resize may needed after expansion

**Cross-AZ Restore:**
- Snapshots AZ-bound (EBS limitation)
- Copy snapshot to target AZ first
- Then restore in target AZ

**Best Practices:**
- Always test restoration process
- Validate data integrity after restore
- Document restore runbooks
- Automate restore testing (weekly)
- Monitor restore performance (warm-up time)

**Next Steps:**
- Module 12 Complete!
- All 11 files rebalanced (70% theory, 30% code)
- Theory-focused content on storage architecture
