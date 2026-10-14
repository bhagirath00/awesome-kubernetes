# Creating Snapshots: Manual and Automated Approaches

## Snapshot Creation Architecture

Before creating snapshots, understand the **three decision points**:

1. **When to snapshot?** (Manual trigger vs automated schedule)
2. **Which volume?** (One PVC or multiple volumes)
3. **How to manage lifecycle?** (Auto-delete or retain indefinitely)

---

## Manual Snapshot Creation: Simple One-Off Backups

### Scenario: Backup Before Risky Database Migration

**Situation:** DBA needs a safety backup before running major schema changes.

```yaml
# Step 1: Verify VolumeSnapshotClass exists
kubectl get volumesnapshotclass

# Output:
# NAME                   DRIVER              DELETIONPOLICY
# ebs-snapshot-class     ebs.csi.aws.com     Delete
```

**Why Check First:** If VolumeSnapshotClass missing, snapshot creation fails silently (looks pending forever).

### Creating the Snapshot

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: pre-migration-backup
  namespace: production
  labels:
    purpose: migration-safety
    created-by: dba-team
    expires: "2024-02-15"
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: postgres-data
```

**What Each Field Means:**

- `volumeSnapshotClassName: ebs-snapshot-class` → "Use AWS EBS snapshot provisioner"
- `persistentVolumeClaimName: postgres-data` → "Snapshot THIS specific PVC"
- `labels` → Tag for organization (can filter/find later)

### Workflow: Create, Monitor, Use

```bash
# Step 1: Create snapshot
kubectl apply -f snapshot.yaml

# Step 2: Monitor progress
kubectl get volumesnapshot pre-migration-backup -w

# Output (initially):
# NAME                     READYTOUSE   SOURCEPVC      SOURCESIZE
# pre-migration-backup     false        postgres-data  50Gi

# Wait 30-60 seconds for AWS to complete snapshot...

# Output (after ready):
# NAME                     READYTOUSE   SOURCEPVC      SOURCESIZE
# pre-migration-backup     true         postgres-data  50Gi

# Step 3: Verify snapshot is in AWS
aws ec2 describe-snapshots --filters "Name=tag:Name,Values=pre-migration-backup"

# Step 4: Now safe to proceed with migration
# If migration fails:
#   - Restore from snapshot (see Chapter 39 File 03)
# If migration succeeds:
#   - Delete snapshot (saves money)
```

### Understanding "Ready to Use" Status

```
READYTOUSE=false:
  ├─ Snapshot being created
  ├─ Data being copied from volume to snapshot storage
  ├─ Can take 30 seconds to 5 minutes (depends on volume size/I/O)
  └─ Cannot restore yet

READYTOUSE=true:
  ├─ Snapshot complete
  ├─ Data safely copied to cloud storage
  └─ Can now restore immediately
```

---

## VolumeSnapshotClass: Defining Snapshot Behavior

### Understanding Parameters

Different storage classes need different snapshot configurations.

### AWS EBS Snapshot Class

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-fast
driver: ebs.csi.aws.com
deletionPolicy: Delete  # Delete AWS snapshot when VolumeSnapshot deleted?
parameters:
  snapshot-encryption: "true"
  tags: "Environment=production,Team=database,BackupType=migration"
```

**deletionPolicy Options:**

| Policy | Behavior | Use Case |
|--------|----------|----------|
| `Delete` | AWS snapshot deleted when VolumeSnapshot object deleted | Temporary backups, save money |
| `Retain` | AWS snapshot kept even after VolumeSnapshot deleted | Important backups, compliance retention |

**Why This Matters:**

```
Scenario 1: deletionPolicy=Delete
  1. Create VolumeSnapshot "pre-migration-backup"
  2. Migration succeeds
  3. Delete VolumeSnapshot
  4. AWS snapshot automatically deleted (costs stop)
  
Scenario 2: deletionPolicy=Retain
  1. Create VolumeSnapshot "compliance-backup-jan-2024"
  2. Delete VolumeSnapshot (by mistake)
  3. AWS snapshot still exists (can restore)
  4. Must manually delete in AWS console
```

### Multi-Environment Snapshot Classes

```yaml
---
# Development: Aggressive cleanup (save money)
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: dev-snapshot-temp
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  tags: "Environment=dev,RetentionDays=7"

---
# Staging: Moderate retention
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: staging-snapshot-30d
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  tags: "Environment=staging,RetentionDays=30"

---
# Production: Long-term compliance
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: prod-snapshot-compliance
driver: ebs.csi.aws.com
deletionPolicy: Retain  # Never auto-delete
parameters:
  snapshot-encryption: "true"
  tags: "Environment=production,RetentionDays=90,Compliance=required"
```

---

## Automated Snapshots: CronJob-Based Scheduling

### The Problem with Manual Snapshots

```
Manual approach:
  Monday 8am: DBA remembers to create snapshot
  Wednesday: Oops, system fails, snapshot from 3 days old
  Friday: Different DBA forgets to snapshot
  Result: Inconsistent protection, human-dependent
```

### Solution: Automated CronJob-Based Snapshots

**Design Pattern: Daily Snapshots**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-daily-snapshot
  namespace: production
spec:
  schedule: "0 2 * * *"  # 2 AM every day (UTC)
  concurrencyPolicy: Forbid  # Don't run two jobs simultaneously
  successfulJobsHistoryLimit: 3  # Keep last 3 successful jobs
  failedJobsHistoryLimit: 1  # Keep last 1 failed job for debugging
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            job: daily-snapshot
        spec:
          serviceAccountName: snapshot-creator  # RBAC permission
          containers:
          - name: snapshot-creator
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              SNAPSHOT_NAME="postgres-backup-$(date +%Y-%m-%d-%H%M%S)"
              echo "Creating snapshot: $SNAPSHOT_NAME"
              
              kubectl apply -f - <<EOF
              apiVersion: snapshot.storage.k8s.io/v1
              kind: VolumeSnapshot
              metadata:
                name: $SNAPSHOT_NAME
                namespace: production
                labels:
                  automated: "true"
                  pvc: postgres-data
              spec:
                volumeSnapshotClassName: prod-snapshot-compliance
                source:
                  persistentVolumeClaimName: postgres-data
              EOF
              
              # Wait for snapshot to be ready
              kubectl wait --for=condition=ready volumesnapshot/$SNAPSHOT_NAME --timeout=5m
              
              # Success
              echo "Snapshot $SNAPSHOT_NAME ready"
          restartPolicy: OnFailure
```

**RBAC Configuration (Required):**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: snapshot-creator
  namespace: production

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: snapshot-creator
  namespace: production
rules:
- apiGroups: ["snapshot.storage.k8s.io"]
  resources: ["volumesnapshots"]
  verbs: ["get", "list", "create", "patch", "watch"]
- apiGroups: ["snapshot.storage.k8s.io"]
  resources: ["volumesnapshotclasses"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: snapshot-creator
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: snapshot-creator
subjects:
- kind: ServiceAccount
  name: snapshot-creator
  namespace: production
```

### Automated Snapshot Cleanup (Retention Policy)

**Problem:** Snapshots accumulate forever, storage costs grow unbounded.

**Solution: Retention Script**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: snapshot-retention-cleanup
  namespace: production
spec:
  schedule: "30 2 * * *"  # 2:30 AM (30 min after snapshots created)
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-cleanup
          containers:
          - name: cleanup
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              # Delete snapshots older than 30 days
              CUTOFF_DATE=$(date -d '30 days ago' +%s)
              
              echo "Cleaning up snapshots older than 30 days"
              
              kubectl get volumesnapshot -n production --sort-by=.metadata.creationTimestamp | grep postgres-backup | while read line; do
                SNAPSHOT_NAME=$(echo $line | awk '{print $1}')
                CREATION_TIME=$(kubectl get volumesnapshot $SNAPSHOT_NAME -n production -o jsonpath='{.metadata.creationTimestamp}')
                CREATION_EPOCH=$(date -d "$CREATION_TIME" +%s)
                
                if [ $CREATION_EPOCH -lt $CUTOFF_DATE ]; then
                  echo "Deleting old snapshot: $SNAPSHOT_NAME"
                  kubectl delete volumesnapshot $SNAPSHOT_NAME -n production
                else
                  echo "Keeping recent snapshot: $SNAPSHOT_NAME"
                fi
              done
              
              echo "Cleanup complete"
          restartPolicy: OnFailure
```

**RBAC for Cleanup:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: snapshot-cleanup
  namespace: production

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: snapshot-cleanup
  namespace: production
rules:
- apiGroups: ["snapshot.storage.k8s.io"]
  resources: ["volumesnapshots"]
  verbs: ["get", "list", "delete", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: snapshot-cleanup
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: snapshot-cleanup
subjects:
- kind: ServiceAccount
  name: snapshot-cleanup
  namespace: production
```

---

## Automated Snapshots with Pre-Quiesce Hooks

### The Problem: Crash-Consistent Snapshots May Corrupt Database

```
Database Timeline:
10:00:00 - Transaction starts: UPDATE accounts SET balance=500
10:00:05 - First half of update written
10:00:10 - Snapshot captured HERE (mid-transaction!)
10:00:15 - Transaction continues
10:00:20 - Transaction completes

Result: Snapshot has incomplete transaction (data inconsistent)
Restore from snapshot: Database won't start (incomplete transaction blocks recovery)
```

### Solution: Database Pre-Snapshot Quiesce

**Pattern: Hooks Before Snapshot**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-snapshot-demo
spec:
  containers:
  - name: postgres
    image: postgres:15
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
    lifecycle:
      preStop:  # Run BEFORE pod shutdown/snapshot
        exec:
          command:
          - /bin/sh
          - -c
          - |
            # Flush buffers
            psql -U postgres -c "SELECT pg_start_backup('backup');"
            # Wait for I/O to complete
            sync
            sleep 5
            # End backup safely
            psql -U postgres -c "SELECT pg_stop_backup();"
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: postgres-data
```

### Automated Quiesce + Snapshot Pattern

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-quiesce-snapshot
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-creator
          containers:
          - name: quiesce-and-snapshot
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              echo "Step 1: Quiesce database (flush to disk)"
              kubectl exec -n production deployment/postgres -- \
                psql -U postgres -c "SELECT pg_start_backup('backup');"
              
              echo "Step 2: Wait for I/O flush"
              sleep 10
              
              echo "Step 3: Create snapshot"
              SNAPSHOT_NAME="postgres-app-consistent-$(date +%Y-%m-%d)"
              kubectl apply -f - <<EOF
              apiVersion: snapshot.storage.k8s.io/v1
              kind: VolumeSnapshot
              metadata:
                name: $SNAPSHOT_NAME
                namespace: production
                labels:
                  consistency: app-consistent
              spec:
                volumeSnapshotClassName: prod-snapshot-compliance
                source:
                  persistentVolumeClaimName: postgres-data
              EOF
              
              echo "Step 4: Wait for snapshot to complete"
              kubectl wait --for=condition=ready volumesnapshot/$SNAPSHOT_NAME --timeout=5m
              
              echo "Step 5: End backup mode"
              kubectl exec -n production deployment/postgres -- \
                psql -U postgres -c "SELECT pg_stop_backup();"
              
              echo "App-consistent snapshot complete"
          restartPolicy: OnFailure
```

---

## Snapshot Creation: Common Patterns and Scenarios

### Pattern 1: Per-Component Snapshots

```yaml
---
# Database snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-data-$(date +%Y%m%d)
spec:
  volumeSnapshotClassName: prod-snapshot-compliance
  source:
    persistentVolumeClaimName: mysql-data

---
# Cache snapshot (if needed for recovery testing)
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: redis-data-$(date +%Y%m%d)
spec:
  volumeSnapshotClassName: dev-snapshot-temp
  source:
    persistentVolumeClaimName: redis-data

---
# Shared file system snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: shared-uploads-$(date +%Y%m%d)
spec:
  volumeSnapshotClassName: prod-snapshot-compliance
  source:
    persistentVolumeClaimName: shared-uploads
```

### Pattern 2: Snapshot Before Deployment

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pre-deployment-snapshot
spec:
  template:
    spec:
      serviceAccountName: snapshot-creator
      containers:
      - name: snapshot
        image: bitnami/kubectl:latest
        command:
        - /bin/sh
        - -c
        - |
          # Before deploying new application version:
          # 1. Snapshot current database
          
          SNAPSHOT_NAME="pre-v2.5-deployment-$(date +%s)"
          kubectl apply -f - <<EOF
          apiVersion: snapshot.storage.k8s.io/v1
          kind: VolumeSnapshot
          metadata:
            name: $SNAPSHOT_NAME
            labels:
              purpose: pre-deployment-safety
              deployment-version: v2.5
          spec:
            volumeSnapshotClassName: prod-snapshot-compliance
            source:
              persistentVolumeClaimName: app-database
          EOF
          
          # Wait for snapshot
          kubectl wait --for=condition=ready volumesnapshot/$SNAPSHOT_NAME --timeout=10m
          
          echo "Snapshot complete - safe to deploy v2.5"
      restartPolicy: Never
```

---

## Monitoring Snapshots: Status and Health Checks

### Checking Snapshot Status

```bash
# List all snapshots in cluster
kubectl get volumesnapshot --all-namespaces

# Watch snapshot creation in real-time
kubectl get volumesnapshot -n production -w

# Get detailed status
kubectl describe volumesnapshot postgres-backup-2024-01-15

# Output includes:
# Status:
#   Bound Volume Snapshot Content Name:  snapcontent-abc123
#   Creation Time:                       2024-01-15T02:00:00Z
#   Ready To Use:                        true
#   Restore Size:                        50Gi
# Events:
#   Type    Reason            Age   Message
#   ----    ------            ---   -------
#   Normal  SnapshotCreated   2m    Successfully created snapshot
```

### Monitoring Snapshot Storage Costs

```bash
# Find all snapshots
kubectl get volumesnapshot --all-namespaces -o custom-columns=NAME:.metadata.name,NAMESPACE:.metadata.namespace,SOURCEPVC:.spec.source.persistentVolumeClaimName,SIZE:.status.restoreSize,READYTOUSE:.status.readyToUse

# Check AWS snapshots (can be orphaned if Kubernetes object deleted)
aws ec2 describe-snapshots --owner-ids self --filters "Name=status,Values=completed" \
  --query 'Snapshots[*].[SnapshotId,StartTime,VolumeSize]' \
  --output table

# Calculate storage cost (example: us-east-1 at $0.025/GB/month)
TOTAL_SIZE=$(aws ec2 describe-snapshots --owner-ids self --filters "Name=status,Values=completed" \
  --query 'sum(Snapshots[*].VolumeSize)' --output text)
MONTHLY_COST=$(echo "scale=2; $TOTAL_SIZE * 0.025" | bc)
echo "Total snapshot storage: ${TOTAL_SIZE}GB"
echo "Estimated monthly cost: $${MONTHLY_COST}"
```

---

## Snapshot Error Handling and Troubleshooting

### Error 1: SnapshotCreation Failed

```bash
kubectl describe volumesnapshot postgres-backup

# Error message:
# Events:
#   Type     Reason                  Age   Message
#   ----     ------                  ---   -------
#   Warning  SnapshotCreationFailed  2m    Snapshot creation failed: invalid provisioner

# Causes:
# 1. CSI driver not installed
# 2. Wrong provisioner name in VolumeSnapshotClass
# 3. CSI driver pod crashed

# Fix:
# Verify driver installed
kubectl get pods -n kube-system | grep csi

# Check VolumeSnapshotClass provisioner
kubectl get volumesnapshotclass -o yaml | grep driver

# Check driver pod logs
kubectl logs -n kube-system deployment/ebs-csi-controller
```

### Error 2: Snapshot Stuck in Pending

```bash
# Symptom: READYTOUSE never becomes true
kubectl get volumesnapshot -w

# Cause: AWS API limit exceeded or slow processing

# Fix: Wait up to 10 minutes, then check AWS directly
aws ec2 describe-snapshots --filters "Name=description,Values=postgres-backup"

# If snapshot exists in AWS, issue is Kubernetes side
# Restart CSI controller pods
kubectl rollout restart deployment/ebs-csi-controller -n kube-system
```

---

## Summary: Creating Snapshots

**Manual Snapshots:** One-off backups before risky operations
- Simple to create
- Good for specific milestones
- Requires manual intervention

**Automated Snapshots:** CronJob-based scheduled backups
- Consistent protection
- Can setup retention policies
- Reduces human error

**VolumeSnapshotClass:** Defines snapshot behavior
- Multiple classes for different retention needs
- Controls auto-deletion behavior
- Can set environment-specific policies

**Pre-Quiesce Hooks:** Database-consistent snapshots
- Prevent corruption on restore
- Use lifecycle hooks or pre-snapshot scripts
- Important for stateful applications

**Best Practices:**
- Use consistent naming (include date/time)
- Label snapshots for organization
- Set retention policies (don't keep forever)
- Monitor costs (incremental but can accumulate)
- Test restoration regularly

**Next File:** Chapter 39 File 03 covers restoring volumes from snapshots.
