# Reclaim Policies: The Final Lifecycle Decision

## The Data Lifecycle Problem

When a developer runs `kubectl delete pvc my-data`, Kubernetes faces a critical decision:

> **Should the underlying storage be deleted or kept?**

This decision has massive consequences:

```
Delete Policy:
  Automatic cleanup (saves money immediately)
  BUT: No recovery if deletion was accidental

Retain Policy:
  Data preserved forever (no data loss)
  BUT: Storage costs accumulate, manual cleanup needed
```

**This is why every production disaster involves reclaim policies.**

---

## Understanding Reclaim Policies: The Three Levels of Data Safety

### Level 0: Developer's Perspective

```
"I'm done with this PVC, delete everything"
kubectl delete pvc my-data
```

### Level 1: Kubernetes Perspective

```
"What should I do with the underlying PV and cloud storage?"

Option A (Delete):
  ├─ Delete PersistentVolume
  ├─ Call cloud provider API: "Delete the storage resource"
  └─ Cost stopped immediately, data irrecoverable

Option B (Retain):
  ├─ Keep PersistentVolume (status: Released)
  ├─ Keep cloud storage resource
  └─ Cost continues, manual cleanup required
```

### Level 2: Cloud Provider (AWS) Perspective

```
AWS sees request: "Delete EBS volume vol-abc123"

Action:
  ├─ Mark volume for deletion
  ├─ Detach from running instances
  ├─ Erase volume data
  ├─ Free allocation
  └─ Volume permanently gone (unrecoverable)
```

**Key Insight:** After AWS deletion, data is gone forever (not in trash, not in recycle bin).

---

## Policy 1: Delete (Default and Dangerous)

```yaml
reclaimPolicy: Delete  # Default behavior
```

### How Delete Policy Works

```
Step 1: Developer deletes PVC
  └─ kubectl delete pvc postgres-data

Step 2: Kubernetes checks StorageClass reclaimPolicy
  └─ Sees: reclaimPolicy=Delete

Step 3: Kubernetes tells provisioner: "Delete the underlying storage"
  └─ CSI driver calls AWS API

Step 4: AWS deletes EBS volume
  ├─ Data erased
  ├─ Volume deallocated
  ├─ Billing stops
  └─ UNRECOVERABLE

Step 5: PV deleted from Kubernetes
  └─ All traces gone
```

### When Delete Policy Is Appropriate

**Good Uses:**
- Development environments (ephemeral data)
- Temporary scratch volumes
- Test databases (data intentionally discarded)
- Cache volumes (data can be recreated)
- Cost-optimized labs

**Example: Development Database**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: dev-ephemeral-db
provisioner: ebs.csi.aws.com
reclaimPolicy: Delete  # ✅ Appropriate here
parameters:
  type: gp3
  iops: 1000  # Lower performance (dev only)
```

When developer finishes testing and deletes PVC → cost stops immediately.

### Delete Policy Dangers

**Scenario: Accidental Deletion**

```
Friday 4:58 PM - Developer runs cleanup script
  ├─ Script has typo: deletes PRODUCTION database PVC
  ├─ 5 seconds later: $500K of customer data gone
  ├─ reclaimPolicy=Delete means DELETE IMMEDIATELY
  ├─ No trash, no recovery possible
  └─ RTO (Recovery Time Objective): ∞ (unrecoverable without snapshots)

Monday Morning:
  ├─ Discovered data loss
  ├─ Had to restore from tape backup (8 hours)
  ├─ Lost weekend's data
  ├─ Regulatory fine: $1M (data loss incident)
  └─ Developer fired
```

**This actually happens in production.**

---

## Policy 2: Retain (The Safe Default)

```yaml
reclaimPolicy: Retain  # Manual control
```

### How Retain Policy Works

```
Step 1: Developer deletes PVC
  └─ kubectl delete pvc postgres-data

Step 2: Kubernetes checks StorageClass reclaimPolicy
  └─ Sees: reclaimPolicy=Retain

Step 3: Kubernetes does NOT tell provisioner to delete
  ├─ PVC deleted from Kubernetes
  └─ PersistentVolume status: "Released" (orphaned)

Step 4: AWS storage volume remains
  ├─ Data preserved
  ├─ Billing continues
  ├─ Can be manually recovered
  └─ Requires manual deletion

Step 5: CloudWatch/Monitoring shows "Released volumes"
  └─ Admin realizes storage exists and can delete if needed
```

### When Retain Policy Is Appropriate

**Must Use:**
- Production databases (data is irreplaceable)
- Critical business data
- Financial records (regulatory requirement)
- Customer data (compliance: GDPR, PCI-DSS, HIPAA)
- Long-running stateful applications
- Backup systems

**Example: Production Database**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: prod-postgres-durable
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain  # ✅ CRITICAL for production
parameters:
  type: io2
  iops: 20000
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123456789:key/12345678
```

When developer deletes PVC by mistake → data still safe in AWS, can recover.

### Retain Policy Consequences

**Cost Accumulation:**

```
Scenario: PVC deleted with reclaimPolicy=Retain
  
  Day 1: PVC deleted, volume retained
    ├─ Storage: 100GB
    ├─ Daily cost: $0.10 × 100GB = $10
    └─ Retained status: "Released"

  Week 1: Nobody notices
    ├─ Accumulated cost: $70
    └─ Volume still allocated

  Month 1: Forgotten about
    ├─ Accumulated cost: $300
    └─ Wasting money

  Year 1: Unknown volumes cost $3,600
    ├─ AWS bill suddenly high
    ├─ Finance team investigates
    ├─ Finds 50 forgotten volumes × $3,600 = $180,000 wasted
    └─ Admin fired for cost overruns
```

**Consequences:**
- Monitoring required (must track Released volumes)
- Manual cleanup overhead
- Cost accumulation if forgotten
- Orphaned volume management

---

## Policy Comparison Matrix

| Factor | Delete | Retain |
|--------|--------|--------|
| **Data Safety** | ❌ Lost immediately | ✅ Preserved |
| **Recovery Possible** | ❌ Never | ✅ Manual recovery |
| **Cost Management** | ✅ Auto-stop | ❌ Manual cleanup needed |
| **Good for** | Ephemeral data | Production data |
| **Risk Level** | CRITICAL | Low (requires monitoring) |
| **Typical Use** | Dev/test | Production |
| **Accidental Deletion Impact** | Catastrophic | Recoverable |
| **Compliance Ready** | ❌ No | ✅ Yes |

---

## Multi-Environment Strategy: Tiered Reclaim Policies

### Strategy: Different Policies Per Environment

```yaml
---
# Development: Delete (ephemeral)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: dev-auto-cleanup
provisioner: ebs.csi.aws.com
reclaimPolicy: Delete
parameters:
  type: gp3
  tags: "Environment=dev,DataType=temporary"

---
# Staging: Retain (test impact recovery)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: staging-safe-delete
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain
parameters:
  type: gp3
  tags: "Environment=staging,DataType=non-critical"

---
# Production: Retain (critical)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: prod-critical-data
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain
parameters:
  type: io2
  iops: 20000
  encrypted: "true"
  tags: "Environment=production,DataType=critical"
```

### Developer Education: Right Storage Class

```
Developer workflow:
  1. Choose environment: "I'm deploying to staging"
  2. Pick StorageClass: "Use staging-safe-delete"
  3. Automatically gets reclaimPolicy=Retain
  4. If deleted by accident: Data recoverable
  5. Manual cleanup prevents cost spiraling
```

---

## Operational Patterns for Retained Volumes

### Pattern 1: Monitoring Retained Volumes

```bash
# Find all Released volumes (from retained PVCs)
kubectl get pv --field-selector status.phase=Released

# Output:
# NAME                      CAPACITY  STATUS   CLAIM
# pvc-abc123-released       100Gi     Released default/postgres-data
# pvc-def456-released       50Gi      Released production/cache

# Check creation time (how long retained?)
kubectl get pv -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,AGE:.metadata.creationTimestamp

# Check associated PVC (why was it deleted?)
kubectl describe pv pvc-abc123-released | grep Claim
```

### Pattern 2: Deciding What to Keep

```yaml
# Add labels for retention decisions
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  labels:
    retention-policy: keep-90-days  # Explicit retention period
    backup-status: snapshotted      # Has backup?
    recovery-date-required: "2024-03-01"  # When can delete?
    owner: dba-team
spec:
  storageClassName: prod-critical-data
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

### Pattern 3: Cleanup Automation (After Safe Period)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-old-retained-volumes
spec:
  schedule: "0 3 * * 0"  # Weekly Sunday 3 AM
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: volume-cleanup
          containers:
          - name: cleanup
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              # Find Released PVs older than 90 days
              CUTOFF=$(date -d "90 days ago" +%s)
              
              kubectl get pv -o json | jq -r '.items[] | select(.status.phase=="Released") | .metadata.name' | while read pv; do
                CREATION=$(kubectl get pv $pv -o jsonpath='{.metadata.creationTimestamp}')
                CREATION_EPOCH=$(date -d "$CREATION" +%s)
                
                if [ $CREATION_EPOCH -lt $CUTOFF ]; then
                  # Check if snapshot exists (backup safety)
                  PVC_NAME=$(kubectl get pv $pv -o jsonpath='{.spec.claimRef.name}')
                  SNAPSHOT_COUNT=$(kubectl get volumesnapshot --all-namespaces -o jsonpath='{.items[?(@.spec.source.persistentVolumeClaimName=="'$PVC_NAME'")].metadata.name}' | wc -w)
                  
                  if [ $SNAPSHOT_COUNT -gt 0 ]; then
                    echo "Deleting PV $pv (has backups)"
                    kubectl delete pv $pv
                  else
                    echo "Keeping PV $pv (no backups found)"
                  fi
                fi
              done
          restartPolicy: OnFailure
```

---

## Reclaim Policy Mistakes (And How to Avoid Them)

### Mistake 1: Wrong Policy in Production

```yaml
# ❌ WRONG: Production with Delete policy
kind: StorageClass
metadata:
  name: prod-bad
provisioner: ebs.csi.aws.com
reclaimPolicy: Delete  # Disaster waiting to happen

# ✅ CORRECT:
kind: StorageClass
metadata:
  name: prod-safe
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain  # Safe
```

**Mitigation:** Code review all StorageClasses before production use.

### Mistake 2: Forgetting About Retained Volumes

```
Result: AWS bill spikes unexpectedly
  └─ Dozens of forgotten retained volumes accumulating

Prevention:
  1. CloudWatch alerts on volume count
  2. Monthly audit of Released PVs
  3. Retention period labels
  4. Automated cleanup after retention period
```

### Mistake 3: Wrong StorageClass for Data Type

```yaml
# ❌ WRONG: Using dev StorageClass in production
kind: Pod
metadata:
  name: prod-database
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
      storageClassName: dev-auto-cleanup  # Wrong class!

# If PVC deleted: Catastrophe (Delete policy in production)
```

**Prevention:** Namespace-bound StorageClasses

```yaml
# Namespace default StorageClass per environment
# production namespace: defaults to StorageClass=prod-critical
# development namespace: defaults to StorageClass=dev-ephemeral

# PVC without explicit storageClassName uses namespace default
```

---

## Recovery from Retained Volumes

### Scenario: Need to Recover Deleted PVC

```
Situation: PVC deleted, had reclaimPolicy=Retain, data needed

Recovery Steps:

Step 1: Find the Released PV
  └─ kubectl get pv --field-selector status.phase=Released

Step 2: Create new PVC that claims the PV
  ├─ Create PVC with volumeName=<released-pv-name>
  └─ Kubernetes binds PVC to existing PV

Step 3: Pod mounts recovered data
  └─ Data available again
```

**Example Recovery:**

```bash
# 1. Find Released PV
PV_NAME=$(kubectl get pv --field-selector status.phase=Released -o jsonpath='{.items[0].metadata.name}')
echo "Found Released PV: $PV_NAME"

# 2. Get storage size
SIZE=$(kubectl get pv $PV_NAME -o jsonpath='{.spec.capacity.storage}')
echo "Size: $SIZE"

# 3. Recover with new PVC
kubectl apply -f - << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: recovered-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ""  # Don't use StorageClass (static PV)
  volumeName: $PV_NAME
  resources:
    requests:
      storage: $SIZE
EOF

# 4. Check recovery
kubectl get pvc recovered-data
# Should show: STATUS=Bound

# 5. Use in pod
kubectl run recover-test --image=alpine --rm -it -- sh -c "mount | grep /data && ls /data"
```

---

## Summary: Reclaim Policies as Safety Nets

**Delete Policy:**
- Use: Development, ephemeral data only
- Pro: Automatic cleanup, saves money
- Con: Unrecoverable if deleted by mistake
- Risk: CRITICAL for production

**Retain Policy:**
- Use: Production, critical data
- Pro: Data preserved, recoverable
- Con: Requires monitoring, manual cleanup
- Safety: Essential for compliance

**Best Practices:**
1. **Production must use Retain** (no exceptions)
2. **Label all PVCs** with retention policy
3. **Audit Released PVs** monthly
4. **Auto-clean after retention period** (snapshot-verified)
5. **Educate developers** on right StorageClass
6. **Test recovery process** regularly

**Lesson Learned:** Reclaim policy is the last defense against accidental data loss. Choose wisely.

**Next File:** Chapter 37 File 04 covers custom storage classes for specific workloads.
