# etcd Restore

## When to Restore

Disaster scenarios:

```
1. Accidental resource deleted
   └─ Restore from recent backup

2. Corruption detected
   └─ Restore from known-good backup

3. Ransomware attack
   └─ Restore from clean backup

4. Cluster upgrade failed
   └─ Restore from pre-upgrade backup
```

---

## Restore Overview

```
Old etcd data (corrupted)
   ↓
Stop etcd
   ↓
Restore from backup snapshot
   ↓
Start etcd
   ↓
New etcd data (recovered!)
```

---

## Restore Process (Self-Managed Cluster)

### Step 1: Stop etcd

```bash
# SSH to master node
ssh ubuntu@master-1

# Check etcd status
sudo systemctl status etcd

# Stop etcd
sudo systemctl stop etcd

# Verify stopped
sudo systemctl status etcd
# Should show: inactive (dead)
```

### Step 2: Backup Current Data

```bash
# Save current etcd data (just in case)
sudo cp -r /var/lib/etcd /var/lib/etcd.corrupt.backup

# Verify backup exists
sudo ls -la /var/lib/etcd.corrupt.backup
```

### Step 3: Restore Snapshot

```bash
# Restore from backup
sudo etcdctl snapshot restore /backup/etcd-20240318-020000.db \
  --data-dir=/var/lib/etcd

# Output:
# {"level":"info","ts":1710758400.123456,"msg":"restored snapshot"}

# Verify data restored
sudo ls -la /var/lib/etcd/
```

### Step 4: Fix Permissions

```bash
# Ensure correct ownership
sudo chown -R etcd:etcd /var/lib/etcd

# Verify permissions
ls -l /var/lib/etcd/
```

### Step 5: Start etcd

```bash
# Start etcd service
sudo systemctl start etcd

# Check status
sudo systemctl status etcd
# Should show: active (running)

# Verify etcd is healthy
sudo etcdctl --endpoints=localhost:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Output:
# http://localhost:2379 is healthy
```

### Step 6: Verify Cluster

```bash
# Check API server can access etcd
kubectl get nodes

# Check cluster info
kubectl cluster-info

# Verify resources
kubectl get deployments --all-namespaces
kubectl get pods --all-namespaces
```

---

## Restore in HA Cluster (3 Masters)

Restoring single master in HA setup:

```
Master-1: etcd restored (recovering)
Master-2: etcd still running
Master-3: etcd still running

Risk: Restored etcd may be out-of-date
Solution: Let it sync from peers
```

### Process

```bash
# 1. On affected master (master-1)
sudo systemctl stop etcd
sudo etcdctl snapshot restore /backup/snapshot.db --data-dir=/var/lib/etcd
sudo systemctl start etcd

# 2. etcd automatically syncs from master-2 and master-3
# 3. Monitor status
sudo etcdctl member list

# Should show all 3 members started
```

---

## Restore Scenario: Deleted Namespace

```
Developer runs by accident:
$ kubectl delete namespace production

All apps in production gone!

Disaster discovered 5 minutes later.
Backup exists from 1 hour ago.

Recovery:
1. Stop etcd on all masters
2. Restore from 1-hour-old backup
3. Start etcd
4. Lost 1 hour of changes, but apps recovered
```

### Recovery Steps

```bash
#!/bin/bash
# emergency-restore.sh

echo "⚠️  EMERGENCY RESTORE IN PROGRESS"

# 1. Cordon all nodes (prevent new scheduling)
kubectl cordon --all

# 2. Stop etcd on all masters
for master in master-1 master-2 master-3; do
  ssh "ubuntu@$master" "sudo systemctl stop etcd"
done

# 3. Backup current data
for master in master-1 master-2 master-3; do
  ssh "ubuntu@$master" "sudo cp -r /var/lib/etcd /var/lib/etcd.corrupt"
done

# 4. Restore from backup on master-1 only
ssh ubuntu@master-1 << 'EOF'
  sudo etcdctl snapshot restore /backup/etcd-20240318-010000.db \
    --data-dir=/var/lib/etcd
  sudo chown -R etcd:etcd /var/lib/etcd
EOF

# 5. Start etcd on master-1
ssh ubuntu@master-1 "sudo systemctl start etcd"

# 6. Wait for master-1 to be ready
sleep 30

# 7. Start etcd on remaining masters (they sync from master-1)
for master in master-2 master-3; do
  ssh "ubuntu@$master" "sudo systemctl start etcd"
done

# 8. Verify cluster
kubectl get nodes
kubectl get namespaces

# 9. Uncordon nodes
kubectl uncordon --all

echo "✅ Restore complete"
```

---

## Point-in-Time Recovery

```
Backups:
- 1:00 AM: Full backup
- 2:00 AM: Full backup
- 3:00 AM: Full backup

Issue discovered: 3:45 AM
├─ Some data corrupted
├─ Corruption happened at 3:15 AM
└─ 3:00 AM backup is clean

Recovery: Restore from 3:00 AM backup
└─ Lose data from 3:00-3:45 AM (45 minutes)

More frequent backups = Smaller data loss window
├─ Hourly backups: max 1 hour loss
├─ Every 15 min: max 15 min loss
├─ Every 5 min: max 5 min loss
```

---

## Restore with Different Cluster Size

```
Original cluster: 3 masters
Restored cluster: 5 masters

After restore:
├─ Only 3 masters have restored etcd data
├─ 2 new masters are empty
├─ Can add new masters to cluster
└─ They sync from existing masters
```

---

## Restore with Topology Changes

```
Old setup (backed up):
- Master-1, Master-2, Master-3
- All in us-east-1

New setup (disaster recovery):
- Need to restore in us-west-2
- Different IP addresses

Process:
1. Restore etcd snapshot on us-west-2 master
2. Update kubeconfig with new IPs
3. API server connects to restored etcd
4. Workers reconnect to new API server
```

---

## What Gets Restored

Restored from backup:

```
✓ All namespaces
✓ All deployments
✓ All pods (definition, not running yet)
✓ All services
✓ All configmaps
✓ All secrets
✓ All RBAC (roles, rolebindings)
✓ All persistent volumes
✓ All storage classes
✓ ALL cluster state
```

NOT restored:

```
✗ Running pod data (only pod definition)
✗ Pod logs
✗ Container images (still on nodes)
✗ Worker node state
```

---

## After Restore: What to Expect

```
Immediately after restore:
├─ Pods are in Running state (in etcd)
├─ But might not actually be running
├─ Controllers see "pods should be running"
├─ Controllers recreate pods
└─ After 5-10 minutes: pods actually running again

Database restoration:
├─ Data restored from etcd
├─ But applications might be confused
├─ May need manual intervention
└─ Verify app state after restore

Example:
- Database pod restored to old etcd state
- Pod definition says "3 replicas"
- Controllers ensure 3 replicas running
- But database data on disk unchanged
- Database might be out-of-sync with expected state
```

---

## Testing Restore Procedure

NEVER test restore for first time during disaster!

```bash
#!/bin/bash
# test-restore.sh

echo "Testing etcd restore procedure..."

# 1. Create test cluster (separate from production)
# Or use staging cluster

# 2. Take current backup as baseline
etcdctl snapshot save /tmp/test-baseline.db

# 3. Make some test changes
kubectl apply -f test-namespace.yaml

# 4. Verify changes
kubectl get namespace test-namespace

# 5. Restore from baseline
sudo systemctl stop etcd
etcdctl snapshot restore /tmp/test-baseline.db --data-dir=/var/lib/etcd
sudo systemctl start etcd

# 6. Verify restore worked
kubectl get namespace test-namespace
# Should NOT exist (because it was in baseline)

# 7. Check if we lost data we wanted to lose
# Success!

echo "✅ Restore procedure tested successfully"
```

Run this monthly or before production restore.

---

## Restore Checklist

```
□ Identify backup to restore (date/time)
□ Verify backup integrity
□ Notify team: "Starting restore"
□ Cordon all nodes
□ Backup current etcd (safety)
□ Stop etcd on all masters
□ Restore snapshot on first master
□ Start etcd on first master
□ Wait for startup
□ Start etcd on other masters
□ Verify cluster is healthy
□ Uncordon nodes
□ Monitor for issues (30+ minutes)
□ Declare recovery complete
□ Post-mortem analysis
```

---

## Summary

etcd restore:

**When Needed:**
- Accidental deletes
- Corruption
- Disasters
- Failed upgrades

**Process:**
- Stop etcd
- Backup current data
- Restore snapshot
- Verify cluster

**Testing:**
- Test restore on staging
- Document procedure
- Practice before disaster

**Post-Restore:**
- Verify all resources
- Monitor applications
- Check for side effects
- 30+ minutes monitoring

**Next:** File 04 covers troubleshooting.
