# etcd Troubleshooting

## Symptoms of etcd Problems

```
Symptom 1: Cluster won't start
├─ API server can't connect to etcd
├─ Control plane components fail
└─ kubectl commands hang

Symptom 2: Slow operations
├─ kubectl get pods takes 30 seconds
├─ API requests timeout
├─ Cluster feels sluggish

Symptom 3: Corruption detected
├─ Checksums don't match
├─ Random resources missing
├─ Pods stuck in weird state

Symptom 4: Member lost
├─ Only 2 of 3 etcd members running
├─ Cluster still works (barely)
├─ No tolerance for another failure
```

---

## Checking etcd Health

### Basic Health Check

```bash
# Check if etcd is running
sudo systemctl status etcd

# Check etcd logs
sudo journalctl -u etcd -n 50

# Check etcd health endpoint
sudo etcdctl --endpoints=localhost:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Output:
# http://localhost:2379 is healthy: successfully committed proposal: took = 2.345ms

# If unhealthy:
# http://localhost:2379 is unhealthy: failed to commit proposal: context deadline exceeded
```

### Cluster Status

```bash
# List all etcd members
etcdctl member list

# Output:
# 8211f1d0f64f3269, started, master-1, http://10.0.1.1:2380, http://10.0.1.1:2379, false
# 8e9e05c52164694d, started, master-2, http://10.0.1.2:2380, http://10.0.1.2:2379, false
# 92fb3c02f1bcd4e1, started, master-3, http://10.0.1.3:2380, http://10.0.1.3:2379, false

# All should show "started"

# Check leader
etcdctl member list --write-out=table
# Look for leader designation
```

---

## Problem: etcd Won't Start

### Symptom

```
systemctl status etcd
# ❌ etcd.service loaded but inactive
# ● etcd.service - etcd
#   Loaded: loaded
#   Active: inactive (dead)
```

### Diagnosis

```bash
# Check logs
sudo journalctl -u etcd -n 100

# Common errors:
# - "address already in use" (port conflict)
# - "permission denied" (file permissions)
# - "connection refused" (peer not reachable)
# - "data dir corrupted" (bad disk)
```

### Solutions

**Solution 1: Port Conflict**

```bash
# Check if port in use
sudo lsof -i :2379

# Kill conflicting process or change etcd port
```

**Solution 2: Permission Issues**

```bash
# Check file permissions
ls -la /var/lib/etcd/

# Should be owned by etcd:etcd
sudo chown -R etcd:etcd /var/lib/etcd

# Restart
sudo systemctl start etcd
```

**Solution 3: Corrupted Data**

```bash
# Backup corrupted data
sudo mv /var/lib/etcd /var/lib/etcd.corrupt

# Start fresh (if single node)
sudo systemctl start etcd

# Or restore from backup
sudo etcdctl snapshot restore /backup/etcd-snapshot.db --data-dir=/var/lib/etcd
sudo chown -R etcd:etcd /var/lib/etcd
sudo systemctl start etcd
```

---

## Problem: Slow etcd

### Symptom

```bash
kubectl get pods
# Takes 10 seconds to respond

kubectl describe node
# Takes 30 seconds

# API server logs show:
# "slow apply request"
```

### Diagnosis

```bash
# Check etcd database size
etcdctl --endpoints=localhost:2379 du

# Output:
# "etcd", 25000000000  # 25 GB! Too large

# Check number of objects
etcdctl --endpoints=localhost:2379 \
  --write-out=json put "marker" "1" | jq '.header.revision'

# Very large revision = lots of history
```

### Solutions

**Solution 1: Compact etcd History**

```bash
# Compact to current revision
REV=$(etcdctl --endpoints=localhost:2379 endpoint status --write-out=json | jq '.endpoints[0].status.header.revision')

etcdctl --endpoints=localhost:2379 compact "$REV"

# Defragment
etcdctl --endpoints=localhost:2379 defrag

# Check size again
etcdctl --endpoints=localhost:2379 du
# Should be smaller
```

**Solution 2: Delete Old Resources**

```bash
# Clean up old pod/job history
# (Reduce number of objects in etcd)

# Find old completed jobs
kubectl get jobs --all-namespaces --sort-by=.metadata.creationTimestamp

# Delete old jobs
kubectl delete jobs --field-selector status.successful=1 --all-namespaces
```

**Solution 3: Increase Hardware**

```
etcd performance depends on:
├─ CPU (more = faster proposals)
├─ RAM (more = better caching)
├─ Disk I/O (SSD > HDD)
└─ Network (low latency between replicas)

If etcd slow:
└─ Upgrade to SSD
└─ Increase RAM to 8+ GB
└─ Reduce network latency
```

---

## Problem: Member Removed

### Symptom

```bash
# Check members
etcdctl member list

# Output shows:
# 8211f1d0f64f3269, unstarted, master-1, http://10.0.1.1:2380, http://10.0.1.1:2379, false
#                   ↑ UNSTARTED!
```

### Diagnosis

```bash
# SSH to that node
ssh ubuntu@master-1

# Check etcd service
sudo systemctl status etcd
# Should be running

# Check logs
sudo journalctl -u etcd | grep -i error
```

### Solutions

**Solution 1: Restart Member**

```bash
ssh ubuntu@master-1

# Check what's wrong
sudo systemctl status etcd

# Try restart
sudo systemctl restart etcd

# Verify after restart
etcdctl member list
# Should show "started"
```

**Solution 2: Remove and Re-add Member**

```bash
# From healthy master

# Remove unstarted member
etcdctl member remove 8211f1d0f64f3269

# On that node, clear data
ssh ubuntu@master-1
sudo systemctl stop etcd
sudo rm -rf /var/lib/etcd

# Start etcd (will rejoin cluster)
sudo systemctl start etcd

# Verify
etcdctl member list
```

---

## Problem: Corruption Detected

### Symptom

```bash
# Checksum mismatch
sudo etcdctl --endpoints=localhost:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint hashkv

# Output:
# {
#   "Endpoint": "http://localhost:2379",
#   "Hash": "2345678901",
#   "HashKV": "3456789012"  # Mismatch!
# }

# Resources missing or weird
kubectl get pods
# Some expected pods don't appear
```

### Root Causes

```
1. Hardware failure
   └─ Disk corruption
   └─ Memory error
   └─ Network packet loss

2. Improper shutdown
   └─ Hard power off (not graceful)
   └─ etcd killed without flush
   └─ Data not synced to disk

3. etcd bug
   └─ Race condition
   └─ Memory leak
   └─ Crash during write
```

### Solutions

**Solution 1: Restore from Backup**

```bash
# This is the safe solution

sudo systemctl stop etcd

# Restore from known-good backup
sudo etcdctl snapshot restore /backup/etcd-clean.db --data-dir=/var/lib/etcd

sudo systemctl start etcd

# Verify
etcdctl endpoint hashkv
# Should match other members
```

**Solution 2: Reset Member (Dangerous!)**

```bash
# ONLY if all backups corrupted

sudo systemctl stop etcd

# Backup corrupted data
sudo mv /var/lib/etcd /var/lib/etcd.corrupt

# Reset cluster (WARNING: DATA LOSS)
sudo etcdctl member remove 8211f1d0f64f3269
sudo systemctl start etcd

# This pod will rejoin as new member
# All its data is lost
```

---

## Problem: Certificate Issues

### Symptom

```bash
# etcd won't start with cert error
sudo journalctl -u etcd

# Output:
# "TLS handshake error: certificate has expired"
# "Failed to authenticate peer: certificate signed by unknown CA"
```

### Check Certificates

```bash
# Check cert expiration
openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -noout -dates

# Output:
# notBefore=Jan  1 12:00:00 2024 GMT
# notAfter=Jan  1 12:00:00 2025 GMT

# If expired, renew
kubeadm certs renew etcd-server

# Restart etcd
sudo systemctl restart etcd
```

---

## Problem: Cluster Split Brain

### Symptom

```bash
# 2 of 3 members lost
# 1 member still running

# Cluster can't make decisions
etcdctl endpoint health
# Failed to commit proposal: context deadline exceeded
```

### Why Split Brain

```
Original: 3 members (master-1, master-2, master-3)

master-2 and master-3 lose network connection

Situation:
├─ master-1: Isolated (1/3 members)
├─ master-2: Isolated (2/3 members)  ← Can't reach master-1
└─ master-3: Connected to master-2

Result:
├─ master-1: Can't make decisions (1/3 < quorum)
├─ master-2 & 3: Can make decisions (2/3 > quorum)
└─ Cluster still works! (master-2/3 side)

After network heals:
└─ All 3 converge to same state
```

### Recovery

```bash
# Wait for network to heal
# OR

# Force remove isolated members
etcdctl member remove 8211f1d0f64f3269
etcdctl member remove 92fb3c02f1bcd4e1

# Restart isolated members
ssh ubuntu@master-2
sudo systemctl stop etcd
sudo systemctl start etcd
# (will rejoin cluster)
```

---

## Monitoring etcd

### Prometheus Metrics

```bash
# etcd exposes metrics on :2379/metrics

# Key metrics to monitor:
# - etcd_server_has_leader (should be 1)
# - etcd_disk_backend_compacted_key_total (should increase)
# - etcd_mvcc_db_total_size_in_bytes (watch for growth)
# - grpc_server_handled_total (request count)
```

### Alerting

```yaml
# Alert if etcd down
- alert: etcdDown
  expr: up{job="etcd"} == 0
  for: 1m

# Alert if leader lost
- alert: etcdNoLeader
  expr: etcd_server_has_leader == 0
  for: 1m

# Alert if member removed
- alert: etcdMemberRemoved
  expr: changes(etcd_server_id[5m]) > 0
  for: 1m
```

---

## Summary

etcd troubleshooting:

**Health Checks:**
- etcdctl endpoint health
- etcdctl member list
- Check logs: journalctl -u etcd

**Common Issues:**
- Won't start: permissions, corruption, port conflict
- Slow: database too large, old history
- Corruption: restore from backup
- Certificates: renew, restart
- Split brain: wait for network, remove members

**Best Practice:**
- Monitor health continuously
- Regular backups
- Test restore procedure
- Keep certificates current

**Module 11 Complete! (13 files total)**

You now understand Kubernetes upgrades and maintenance:
- Chapter 34: Cluster upgrades (5 files)
- Chapter 35: Node maintenance (4 files)
- Chapter 36: etcd management (4 files)

**Ready for Module 12!**
