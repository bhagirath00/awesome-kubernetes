# etcd Backup

## Backup Strategy

```
Daily routine:
├─ 01:00 UTC: Snapshot etcd
├─ 01:05 UTC: Upload to S3
├─ 01:10 UTC: Verify backup integrity
└─ Done (keeps last 30 daily backups)

Before any major operation:
├─ Cluster upgrade
├─ Major configuration change
├─ Network surgery
└─ Take manual backup
```

---

## Taking etcd Snapshot

### On Self-Managed Cluster

```bash
# SSH to master node
ssh ubuntu@master-1

# List etcd members
etcdctl member list

# Take snapshot
sudo etcdctl --endpoints=localhost:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db

# Output:
# Snapshot saved at /backup/etcd-20240318-020000.db

# Verify snapshot
sudo etcdctl snapshot status /backup/etcd-20240318-020000.db

# Output:
# hash: 8fc6ab6e9e7b0a18
# revision: 2847361
# totalKey: 8942
# totalSize: 2560270000
```

### On AWS EKS

EKS manages etcd, but you can backup cluster state:

```bash
# Backup all Kubernetes resources
kubectl get all --all-namespaces -o yaml > cluster-backup-$(date +%Y%m%d).yaml

# More comprehensive:
for resource in deployments statefulsets daemonsets services configmaps secrets pvc; do
  kubectl get $resource --all-namespaces -o yaml > backup-$resource-$(date +%Y%m%d).yaml
done

# Or use tools like Velero
helm install velero vmware-tanzu/velero \
  --namespace velero --create-namespace \
  --set configuration.backupStorageLocation.bucket=my-backup-bucket \
  --set configuration.schedules.daily.schedule="0 2 * * *"
```

---

## Automated Backup Script

```bash
#!/bin/bash
# backup-etcd.sh

BACKUP_DIR="/backup/etcd"
REMOTE_BUCKET="s3://my-backups/etcd"
RETENTION_DAYS=30

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Generate backup name with timestamp
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="$BACKUP_DIR/etcd-$TIMESTAMP.db"

echo "Starting etcd backup at $TIMESTAMP..."

# Take snapshot
sudo etcdctl --endpoints=localhost:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save "$BACKUP_FILE"

if [ $? -ne 0 ]; then
  echo "❌ Backup failed"
  exit 1
fi

# Verify backup
sudo etcdctl snapshot status "$BACKUP_FILE"

# Compress backup
sudo gzip "$BACKUP_FILE"
COMPRESSED_FILE="$BACKUP_FILE.gz"

# Get file size
SIZE=$(du -h "$COMPRESSED_FILE" | cut -f1)
echo "Backup compressed: $SIZE"

# Upload to S3
echo "Uploading to S3..."
aws s3 cp "$COMPRESSED_FILE" "$REMOTE_BUCKET/etcd-$TIMESTAMP.db.gz"

if [ $? -ne 0 ]; then
  echo "❌ S3 upload failed"
  exit 1
fi

echo "✅ Backup complete"

# Clean up old local backups (keep last 7 days)
find "$BACKUP_DIR" -name "etcd-*.db.gz" -mtime +7 -delete

# Clean up old S3 backups (keep last 30 days)
aws s3api list-objects-v2 \
  --bucket my-backups \
  --prefix etcd \
  --query 'Contents[?LastModified<=`2024-02-16`].Key' \
  --output text | \
  xargs -I {} aws s3 rm "s3://my-backups/{}"
```

---

## Scheduled Backups with Cron

```bash
# Add to /etc/crontab

# Daily backup at 2:00 AM UTC
0 2 * * * root /opt/scripts/backup-etcd.sh >> /var/log/etcd-backup.log 2>&1

# Hourly backup during business hours
0 8-17 * * 1-5 root /opt/scripts/backup-etcd.sh >> /var/log/etcd-backup.log 2>&1

# Before cluster upgrades
# (manual trigger required)
```

---

## Backup on AWS EKS

AWS manages etcd, but backup your applications:

```bash
#!/bin/bash
# backup-eks-cluster.sh

CLUSTER_NAME="production"
NAMESPACE="production"
BACKUP_BUCKET="s3://my-backups/eks"

# Create backup directory
mkdir -p /tmp/backup

# Backup all resources
echo "Backing up cluster resources..."
kubectl get all --all-namespaces -o yaml > /tmp/backup/all-resources.yaml

# Backup persistent data
echo "Backing up persistent volumes..."
kubectl get pvc --all-namespaces -o yaml > /tmp/backup/pvcs.yaml

# Backup RBAC
echo "Backing up RBAC..."
kubectl get roles,rolebindings,clusterroles,clusterrolebindings --all-namespaces -o yaml > /tmp/backup/rbac.yaml

# Backup network policies
echo "Backing up network policies..."
kubectl get networkpolicies --all-namespaces -o yaml > /tmp/backup/networkpolicies.yaml

# Create tarball
tar -czf /tmp/backup/eks-backup-$(date +%Y%m%d-%H%M%S).tar.gz -C /tmp backup/

# Upload to S3
aws s3 cp /tmp/backup/eks-backup-*.tar.gz "$BACKUP_BUCKET/"

echo "✅ EKS cluster backup complete"
```

---

## Backup Verification

### Verify Backup Integrity

```bash
# Check backup file exists
ls -lh /backup/etcd*.db.gz

# Check file size (should be reasonable)
# Too small = might be corrupted
# Too large = might be uncompressed

# Test if can list contents
sudo etcdctl snapshot status /backup/etcd-20240318-020000.db

# Output shows:
# hash, revision, totalKey, totalSize
# If hash shows "has not been verified" = potential corruption
```

### Backup Rotation

```bash
# Keep only last 30 daily backups
find /backup/etcd -name "etcd-*.db.gz" -mtime +30 -delete

# Verify recent backups exist
ls -lh /backup/etcd-*.db.gz | tail -10
```

---

## Real-World Backup Workflow

```bash
# Monday 2:00 AM: Automated daily backup
./backup-etcd.sh
# Backup: 2.3 GB → 450 MB (compressed)
# Uploaded to S3
# Verified: OK

# Wednesday 3:00 PM: Pre-upgrade backup (manual)
# Before upgrading cluster to v1.27
./backup-etcd.sh
# Backup stored and verified

# Thursday 1:00 AM: Issue detected
# Corruption in etcd
# Restore from Wednesday backup
etcdctl snapshot restore /backup/etcd-20240318-150000.db
# Cluster recovered!
```

---

## Storage for Backups

### Option 1: Local Storage

```
Pros: Fast, immediate
Cons: Lost if master dies

Use when: Testing only
```

### Option 2: Network Storage (NFS)

```
Pros: Separate from master
Cons: If network dies, no backup

Use when: Testing or small clusters
```

### Option 3: Cloud Storage (S3, GCS)

```
Pros: Highly available, durable, geographic redundancy
Cons: Slightly slower, costs money

Use when: Production! (Best practice)

Recommendation: Store in different region
├─ Cluster in us-east-1
├─ Backups in us-west-2
└─ If entire region fails, backups safe
```

---

## Backup Size Management

```
Backup sizes grow over time:
├─ Day 1:   100 MB
├─ Day 30:  200 MB
├─ Day 90:  350 MB
├─ Day 365: 800 MB

Cost multiplier:
├─ 1 daily backup = 1 × daily size
├─ 30 daily backups = 30 × daily size
├─ Compress to reduce: 30 × (daily size * 0.25)

Example:
├─ Uncompressed: 200 MB × 30 = 6 GB
├─ Compressed: 200 MB × 0.25 × 30 = 1.5 GB
└─ Savings: 75%
```

---

## Backup Best Practices

1. **Backup Before Major Operations**
   - Upgrades
   - Configuration changes
   - Network changes

2. **Automate Backup**
   - Cron job for daily backups
   - Script handles upload
   - No manual steps

3. **Store Remotely**
   - S3, GCS, or similar
   - Different region if possible
   - Encrypted in transit

4. **Verify Backups**
   - Regular integrity checks
   - Test restore monthly
   - Document restore procedure

5. **Retention Policy**
   - Keep last 30 daily backups
   - Keep last 12 monthly backups
   - Delete older backups

---

## Backup Checklist

Before upgrading cluster or major changes:

- [ ] Current etcd size: `_____`
- [ ] Take backup: `_____` (date/time)
- [ ] Verify backup: `_____` (status)
- [ ] Upload to S3: `_____` (confirmed)
- [ ] List S3 backups: `_____` (verify present)
- [ ] Test restore procedure: `_____` (date)

---

## Summary

etcd backup:

**When to Backup:**
- Before upgrades
- Daily automated
- Before major changes

**How to Backup:**
- etcdctl snapshot save
- Compress to reduce size
- Upload to remote storage

**Verification:**
- Check file size
- Verify hash
- Test restore

**Retention:**
- Keep 30 daily backups
- Store in different region
- Automated cleanup

**Next:** File 03 covers restoring from backup.
