# Persistent Volume Claims

## Introduction

A **PersistentVolumeClaim (PVC)** is a request for storage by a user. Instead of referencing a specific EBS volume or NFS share, you describe what you need: "I need 50Gi of storage with ReadWriteOnce access." Kubernetes (or the StorageClass) fulfills this request by binding the PVC to a suitable PersistentVolume.

PVCs are namespace-scoped — a PVC in the `production` namespace can only be used by Pods in the `production` namespace. A Pod references the PVC by name, and Kubernetes mounts the underlying storage.

---

## Anatomy of a PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: production
  labels:
    app: postgres
    environment: production
spec:
  # Which StorageClass to use for dynamic provisioning
  # If omitted, uses the cluster's default StorageClass
  storageClassName: gp3
  
  # Access mode — must be supported by the StorageClass/PV
  accessModes:
  - ReadWriteOnce     # EBS: one node at a time
  
  # How much storage to request
  resources:
    requests:
      storage: 100Gi

  # Optional: request a specific volume by label selector (static provisioning)
  # selector:
  #   matchLabels:
  #     environment: production
```

```bash
kubectl apply -f postgres-pvc.yaml

kubectl get pvc postgres-pvc -n production

# NAME           STATUS   VOLUME                   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# postgres-pvc   Bound    pvc-abc123-def456-...   100Gi      RWO            gp3            10s

# STATUS:
# Pending → no matching PV yet (dynamic provisioning in progress)
# Bound   → PV found and bound — ready to use
# Lost    → bound PV has disappeared (node/storage failure)
```

---

## Using a PVC in a Pod

A Pod references the PVC in its volume spec:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: production
spec:
  replicas: 1           # EBS PVCs (RWO) can only be used by 1 Pod at a time
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc    # reference the PVC by name
```

---

## PVC Status States

```bash
# Watch a PVC go through states
kubectl get pvc postgres-pvc -n production -w

# NAME           STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# postgres-pvc   Pending                                      gp3            2s
# postgres-pvc   Pending                                      gp3            4s
# postgres-pvc   Bound    pvc-abc  100Gi       RWO            gp3            8s
# ← ~5 seconds for EBS volume to be provisioned and attached
```

**Pending** is normal when the Pod using the PVC has not started yet (dynamic provisioning begins only when a Pod actually claims the PVC — unless `volumeBindingMode: Immediate`).

---

## Expanding a PVC — Resizing Storage

EBS volumes and EFS support online expansion — you can increase a PVC's size without downtime:

```bash
# Increase postgres-pvc from 100Gi to 200Gi
kubectl patch pvc postgres-pvc -n production \
  --type='merge' \
  -p='{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'

# Watch the expansion
kubectl get pvc postgres-pvc -n production -w

# NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES
# postgres-pvc   Bound    pvc-abc...    100Gi      RWO     ← before
# postgres-pvc   Bound    pvc-abc...    200Gi      RWO     ← after expansion

# The underlying EBS volume is expanded by the EBS CSI driver
# For ext4/xfs filesystems, the filesystem is also expanded automatically
```

```bash
# Verify inside the Pod that the filesystem expanded
kubectl exec postgres-pod -n production -- df -h /var/lib/postgresql/data
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/xvdbf      196G   25G  171G  13% /var/lib/postgresql/data
# ← shows the new 200Gi size
```

> ⚠️ **Note:** You can only increase a PVC's storage, not decrease it. PVC shrinking is not supported because safely reducing a filesystem size risks data loss.

The StorageClass must have `allowVolumeExpansion: true` for this to work. The EKS default `gp3` StorageClass has this enabled.

---

## PVC Retention After Pod Deletion

Deleting a Pod does NOT delete its PVC or PV. The storage is preserved:

```bash
# Delete the Pod
kubectl delete pod postgres-pod-abc123 -n production

# PVC still exists — data preserved
kubectl get pvc postgres-pvc -n production
# NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS
# postgres-pvc   Bound    pvc-abc...    100Gi      RWO            gp3

# New Pod starts and reattaches the same PVC → all data intact
```

Deleting the PVC is what triggers the reclaim policy. If the StorageClass uses `Delete` reclaim policy, deleting the PVC also deletes the EBS volume:

```bash
# This deletes BOTH the PVC AND the EBS volume (if StorageClass uses Delete policy)
kubectl delete pvc postgres-pvc -n production
# WARNING: All data in the EBS volume is permanently lost!
```

Always protect production PVCs with RBAC — restrict who can delete PVCs in production namespaces.

---

## Checking What Storage a PVC Uses

```bash
# Get the PV name from the PVC
kubectl get pvc postgres-pvc -n production -o jsonpath='{.spec.volumeName}'
# pvc-abc123-def456-7890-1234-abcdef123456

# Describe the PV for physical storage details
kubectl describe pv pvc-abc123-def456-7890-1234-abcdef123456

# Source:
#   Driver:       ebs.csi.aws.com
#   VolumeHandle: vol-0abc123def456789    ← actual EBS volume ID
#   FSType:       ext4

# View the EBS volume in AWS
aws ec2 describe-volumes \
  --volume-ids vol-0abc123def456789 \
  --query 'Volumes[].{Size:Size,State:State,AZ:AvailabilityZone,Iops:Iops}'

# [{"Size": 100, "State": "in-use", "AZ": "us-east-1a", "Iops": 3000}]
```

---

## Common PVC Issues and Fixes

### PVC Stays in `Pending`

```bash
kubectl describe pvc postgres-pvc -n production

# Events:
#   Warning  ProvisioningFailed  storageclass.storage.k8s.io "gp3" not found
#   → Fix: check StorageClass name
kubectl get storageclass

#   Warning  ProvisioningFailed  quota exceeded — no more PVCs allowed
#   → Fix: check ResourceQuota limits
kubectl describe resourcequota -n production

#   Warning  ProvisioningFailed  no nodes available in AZ us-east-1c
#   → Fix: check if your nodes are in the same AZ the PVC is requesting
```

### PVC Is `Bound` But Pod Is `Pending`

```bash
kubectl describe pod postgres-pod -n production

# Events:
#   Warning  FailedAttachVolume  Multi-Attach error: volume is already attached to node...
#   → EBS is being detached from old node — wait 60-90 seconds
#   → or force detach in AWS console if node is gone

#   Warning  FailedMount  AttachVolume.Attach failed: ... instance and EBS are in different AZs
#   → Pod scheduled in us-east-1b, EBS volume is in us-east-1a
#   → Add topology constraints or use EFS for multi-AZ
```

---

## Summary

- PVCs are namespace-scoped storage requests — they describe what the Pod needs (size, access mode, StorageClass) without specifying which physical volume to use
- A PVC goes through: `Pending` (no PV yet) → `Bound` (PV found) → `Lost` (PV gone)
- Reference PVCs in Pod specs with `volumes[].persistentVolumeClaim.claimName`
- Expand a PVC by patching `spec.resources.requests.storage` — EBS and EFS support online expansion; filesystem expands automatically
- Deleting a Pod does NOT delete the PVC — storage is preserved and reattached when the Pod restarts
- Deleting a PVC with `Delete` reclaim policy destroys the EBS volume permanently — protect production PVCs with RBAC

In the next file, `03-storage-classes.md`, you will learn about StorageClasses — the provisioning templates that define how dynamic PVs are created, what disk type to use, and whether volumes can expand.
