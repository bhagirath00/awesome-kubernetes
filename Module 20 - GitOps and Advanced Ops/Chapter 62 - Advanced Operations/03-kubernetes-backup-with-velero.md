# Kubernetes Backup and Restore with Velero

**Velero** (formerly Heptio Ark) is the standard cloud-native disaster recovery, backup, and cluster migration tool for Kubernetes. While GitOps restores your desired manifests, Velero captures both the **Kubernetes API metadata** (namespaces, secrets, configmaps, CRDs) and the **underlying persistent storage data** (EBS volumes, PersistentVolumeClaims, persistent files).

---

## 1. Velero Architecture

```
  +-------------------------------------------------------------------------+
  |                          Kubernetes Cluster                             |
  |                                                                         |
  |  [ Velero CLI ] ---> Creates "Backup" Custom Resource                   |
  |                              |                                          |
  |                              v                                          |
  |  +-------------------------------------------------------------------+  |
  |  |                         Velero Server Pod                         |  |
  |  |  - Queries K8s API server for targeted resources                  |  |
  |  |  - Packages manifests into tar.gz archive                         |  |
  |  |  - Calls Cloud Provider CSI Plugin / VolumeSnapshot API           |  |
  |  +--------------------+----------------------+-----------------------+  |
  |                       |                      |                          |
  |       Node Agent      |                      |                          |
  |     (Kopia / Restic)  |                      |                          |
  |   File-level backup   |                      |                          |
  +-----------+-----------+----------------------+--------------------------+
              |                                  |
              | Uploads Volume Data              | Uploads Manifests (.tar.gz)
              v                                  v
    +-------------------+              +-------------------+
    | Cloud Block Store |              | Object Storage    |
    | (AWS EBS / GCP PD |              | (S3 / GCS / Blob) |
    |  Snapshots)       |              |                   |
    +-------------------+              +-------------------+
```

### Core Concepts:
- **`BackupStorageLocation` (BSL)**: Defines the remote object storage bucket (e.g., AWS S3, MinIO, Google Cloud Storage) where tarballs of Kubernetes manifests are stored.
- **`VolumeSnapshotLocation` (VSL)**: Defines the cloud provider infrastructure used to take block-level snapshots of Persistent Volumes (e.g., AWS EBS CSI snapshotter).
- **Node-Agent (Kopia/Restic)**: A DaemonSet that performs file-level backups directly from the node filesystem when cloud CSI block snapshots are unavailable or across hybrid environments.

---

## 2. Installing Velero (AWS S3 & EBS Example)

### Step 1: Prepare Cloud Credentials & Storage
Create an S3 bucket and IAM policy with permissions for S3 (`PutObject`, `GetObject`) and EC2 EBS (`CreateSnapshot`, `CreateVolume`).

```bash
# S3 Bucket
aws s3api create-bucket \
  --bucket corp-velero-backups-prod \
  --region us-east-1

# Save IAM credentials file
cat <<EOF > credentials-velero
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
EOF
```

### Step 2: Install Velero via CLI
Install the Velero server with the AWS plugin and enable the node-agent:

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket corp-velero-backups-prod \
  --secret-file ./credentials-velero \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --use-node-agent \
  --wait
```

### Step 3: Verify Controller & Storage Locations
```bash
velero backup-location get
# NAME      PROVIDER   BUCKET/PREFIX               PHASE       LAST VALIDATED
# default   aws        corp-velero-backups-prod    Available   2026-10-14 14:00:00

kubectl get pods -n velero
```

---

## 3. Creating Backups: On-Demand & Scheduled

### On-Demand Backup
Create a snapshot of specific production namespaces, excluding monitoring noise:

```bash
velero backup create prod-backup-payments-20261014 \
  --include-namespaces payment-system,orders \
  --exclude-resources events,event.events.k8s.io \
  --snapshot-volumes=true \
  --ttl 720h0m0s  # Retain for 30 days
```

### Automated Schedules (`velero schedule`)
Production clusters require automated schedules with predefined retention windows.

```bash
# Create a daily backup schedule at 2:00 AM UTC
velero schedule create prod-daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces payment-system,database \
  --snapshot-volumes=true \
  --ttl 168h  # Keep for 7 days
```

### Declarative Schedule Manifest
You can also manage Velero schedules natively via GitOps:

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: critical-workloads-hourly
  namespace: velero
spec:
  schedule: "0 * * * *"
  template:
    includedNamespaces:
      - payment-system
    excludedResources:
      - events
    snapshotVolumes: true
    ttl: 72h0m0s
```

---

## 4. Consistent Database Backups with Pre/Post Hooks

Backing up a running database (PostgreSQL, MySQL, MongoDB) by snapshotting disk blocks while writes are occurring can cause database corruption. Velero supports **Pre and Post backup hooks** to flush caches and freeze tables.

Annotate the database Pod manifest:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-primary
  namespace: database
spec:
  template:
    metadata:
      annotations:
        # Pre-backup hook: Put database into backup mode & flush WAL
        pre.hook.backup.velero.io/container: postgres
        pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "psql -U postgres -c \"SELECT pg_backup_start('velero_backup', true);\""]'
        pre.hook.backup.velero.io/timeout: 30s
        pre.hook.backup.velero.io/on-error: Fail

        # Post-backup hook: Release backup lock
        post.hook.backup.velero.io/container: postgres
        post.hook.backup.velero.io/command: '["/bin/bash", "-c", "psql -U postgres -c \"SELECT pg_backup_stop();\""]'
        post.hook.backup.velero.io/timeout: 30s
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
```

---

## 5. Restoration Runbooks

### Scenario A: Namespace Inadvertently Deleted
A junior engineer deletes the `payment-system` namespace.

```bash
# 1. Identify the latest successful backup
velero backup get

# 2. Trigger restoration
velero restore create restore-payment-system-$(date +%s) \
  --from-backup prod-daily-backup-20261014020000 \
  --include-namespaces payment-system \
  --restore-pvcs=true

# 3. Monitor restore progress
velero restore describe restore-payment-system-XXXXX
```

### Scenario B: Restoring into a Different Cluster (Migration / Disaster)
In the new target cluster, install Velero pointing to the same S3 bucket:

```bash
# 1. Point target cluster Velero to the existing backup bucket in read-only mode
velero backup-location set default --access-mode=ReadOnly

# 2. Wait for backup synchronization
velero backup get

# 3. Restore all namespaces
velero restore create disaster-recovery-full-restore \
  --from-backup prod-daily-backup-20261014020000 \
  --wait
```

---

## 6. Verification and Troubleshooting

### Health & Diagnostic Commands

```bash
# View backup logs directly from S3
velero backup logs <BACKUP_NAME>

# Inspect restore failures
velero restore logs <RESTORE_NAME> | grep -i "error"

# Check Node-Agent pod logs for file-level backup issues
kubectl logs -n velero -l name=node-agent --tail=100
```

### Common Gotchas and Remediations

| Issue | Root Cause | Solution |
| :--- | :--- | :--- |
| `BackupStorageLocation unavailable` | Bad IAM credentials or S3 bucket network policy | Verify IAM permissions; ensure nodes or IRSA have access to the S3 bucket endpoint. |
| `Volume snapshot timeout / CSI error` | CSI Driver missing `VolumeSnapshotClass` | Verify default class: `kubectl get volumesnapshotclass`. Add annotation `snapshot.storage.kubernetes.io/is-default-class: "true"`. |
| `Restore PartiallyFailed` | In-use resource conflicts (e.g. Service ClusterIP) | Velero skips existing resources by default. Use `--existing-resource-policy update` if overwriting. |
| `File lock error during Kopia backup` | Multiple operations accessing repo simultaneously | Clear stale locks: `kubectl -n velero exec -it <node-agent-pod> -- kopia maintenance run`. |
