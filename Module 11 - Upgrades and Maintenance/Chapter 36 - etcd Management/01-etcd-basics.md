# etcd Basics

## What is etcd?

etcd is the brain of Kubernetes. It stores EVERYTHING:

```
┌──────────────────────────────┐
│        etcd Database         │
├──────────────────────────────┤
│ Deployments and Pods        │
│ Services and Endpoints      │
│ ConfigMaps and Secrets      │
│ RBAC roles and bindings     │
│ Persistent volumes          │
│ All cluster state           │
└──────────────────────────────┘

If etcd dies → Your cluster data is gone!
If etcd is corrupt → Cluster is corrupted!
```

---

## How etcd Works

```
API Server ← etcd (read/write)
   ↓ watches
Scheduler, Controller-Manager (read)
   ↓ updates
etcd (back to stored state)
```

Every Kubernetes operation hits etcd.

---

## etcd on EKS vs Self-Managed

### On AWS EKS

```
AWS manages etcd for you:
✓ AWS backs up etcd
✓ AWS replicates etcd
✓ AWS monitors etcd
✓ You don't need to worry
```

### On Self-Managed Kubernetes

```
You manage etcd:
✓ You backup etcd
✓ You replicate etcd
✓ You monitor etcd
✓ You restore if needed
```

---

## Why Backup etcd?

### Scenario 1: Accidental Delete

```bash
# Someone runs:
kubectl delete namespace production

# Everything in that namespace is deleted from etcd
# Result: All pods, services, deployments gone

# Solution: Restore from backup
```

### Scenario 2: Cluster Upgrade Goes Wrong

```bash
# Cluster upgrade fails
# etcd has corrupted data
# Cluster won't start

# Solution: Restore etcd from pre-upgrade backup
```

### Scenario 3: Ransomware Attack

```bash
# Attacker compromises cluster
# Modifies or deletes resources
# Cluster is compromised

# Solution: Restore from clean backup
```

---

## What etcd Stores

```
Key-value pairs:

/registry/namespaces/default
  └─ {"metadata": {"name": "default"}, ...}

/registry/pods/default/my-app-xyz
  └─ {"spec": {"containers": [...]}, ...}

/registry/services/default/my-service
  └─ {"spec": {"ports": [...]}, ...}

/registry/secrets/default/my-secret
  └─ {"data": {"password": "base64-encoded"}, ...}
```

---

## etcd Size and Performance

### Why Size Matters

```
Small cluster:   etcd < 1GB  (fine)
Medium cluster:  etcd 2-5GB  (acceptable)
Large cluster:   etcd 5-10GB (concerning)
Very large:      etcd > 10GB (problem!)

As etcd grows:
- Backup/restore takes longer
- Snapshot size increases
- Performance degrades
```

### Check etcd Size

```bash
# On master node or EKS control plane
etcdctl --endpoints=localhost:2379 du

# Output:
# "etcd", 5368709120  # Bytes

# In GB
echo "scale=2; 5368709120 / 1024 / 1024 / 1024" | bc
# 5.00 GB
```

---

## etcd Replication

### Single Node etcd (Not Recommended)

```
┌─────────────┐
│ Master-1    │
├─────────────┤
│ etcd        │
└─────────────┘

Risk: Master-1 dies → etcd lost
```

### Replicated etcd (Recommended)

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Master-1    │    │ Master-2    │    │ Master-3    │
├─────────────┤    ├─────────────┤    ├─────────────┤
│ etcd        ├────┤ etcd        ├────┤ etcd        │
└─────────────┘    └─────────────┘    └─────────────┘

All 3 have same data (replicated)
Risk: 1 or 2 masters die, 1+ still has data
```

---

## High Availability Architecture

```
Leader election among 3 etcd nodes:
├─ One is leader (receives writes)
└─ Two are followers (receive from leader)

Write path:
1. Client writes to leader
2. Leader replicates to followers
3. Followers ack receipt
4. Write considered committed

If leader dies:
├─ Followers elect new leader
├─ Cluster continues
└─ Zero downtime
```

---

## Common Mistakes with etcd

### Mistake 1: No Backup

```bash
# No backup strategy
# No automation
# etcd fails
# Data is lost

# Consequence: Entire cluster lost
```

### Mistake 2: Backup But Never Test

```bash
# Backup exists
# Never tested restore
# When needed, backup corrupted
# Can't restore

# Consequence: Can't recover from disaster
```

### Mistake 3: Backup Not Frequent Enough

```bash
# Backup once per day
# Cluster changes 1000x per day
# Backup has stale data
# Data loss = up to 24 hours of changes

# Better: Backup every hour (or more)
```

### Mistake 4: Backup Stored Only Locally

```bash
# Backup on master node
# Master dies
# Backup lost with master

# Better: Backup to remote storage
```

---

## etcd Monitoring

### Check etcd Health

```bash
# On master node
etcdctl endpoint health

# Output:
# http://localhost:2379 is healthy: successfully committed proposal: took = 3.123ms

# If unhealthy:
# http://localhost:2379 is unhealthy: failed to commit proposal: context deadline exceeded
```

### Check etcd Status

```bash
# Get cluster status
etcdctl member list

# Output:
# 8211f1d0f64f3269, started, master-1, http://10.0.1.1:2380, http://10.0.1.1:2379, false
# 8e9e05c52164694d, started, master-2, http://10.0.1.2:2380, http://10.0.1.2:2379, false
# 92fb3c02f1bcd4e1, started, master-3, http://10.0.1.3:2380, http://10.0.1.3:2379, false
```

---

## etcd on AWS EKS

On EKS, etcd is managed by AWS:

```bash
# You DON'T see etcd
# AWS runs it for you
# AWS backs it up
# AWS handles replication

# Your job: Trust AWS (they do good job!)
# But you can still backup your own data:
kubectl get all --all-namespaces -o yaml > my-cluster-backup.yaml
```

---

## Real-World Scenario: etcd Corruption

```
Monday 2am: etcd corruption detected
├─ Cluster still running
├─ But data corrupted
├─ Pod creation failing
├─ Services not working correctly

Tuesday 2am: Investigation
├─ Root cause: Hardware issue
├─ etcd data written incorrectly to disk
└─ No good backups available (all corrupted too)

Tuesday 3am: Disaster
├─ Cluster must be rebuilt from scratch
├─ All state lost
└─ Hours of manual reconstruction

Prevention:
└─ Regular etcd backups
└─ Store backups remotely
└─ Test restore procedure
```

---

## Summary

etcd basics:

**What Is etcd:**
- Key-value database
- Stores all cluster state
- Heart of Kubernetes

**Why Backup:**
- Accidental deletes
- Corruption
- Disasters
- Ransomware

**Best Practices:**
- Backup regularly (hourly)
- Store backups remotely
- Test restore procedure
- Monitor etcd health
- HA etcd (3 nodes minimum)

**Next:** File 02 covers backing up etcd.
