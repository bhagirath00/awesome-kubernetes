# etcd Overview

## Introduction

etcd is the backbone of Kubernetes. It is a distributed key-value store that holds the complete state of your cluster — every Pod, Deployment, Service, Secret, ConfigMap, and Node registration. Without etcd, Kubernetes has no memory. The control plane cannot make decisions, and the cluster effectively ceases to function.

Understanding etcd is important even if you use AWS EKS (where etcd is fully managed). Knowing what it does helps you debug cluster issues, understand cluster performance characteristics, and appreciate why certain Kubernetes operations behave the way they do.

---

## What etcd Is

**etcd** stands for "et cetera distributed." It was created by CoreOS (now part of Red Hat) and is a CNCF graduated project.

Key characteristics:
- **Key-value store** — data is stored as key-value pairs
- **Distributed** — runs as a cluster of nodes for high availability
- **Strongly consistent** — every read returns the most recent write
- **Watch-capable** — clients can subscribe to changes on any key
- **Fast** — designed for low-latency reads and writes (microseconds)

In Kubernetes, etcd is the **only** persistent storage. The API Server, Scheduler, and Controller Manager are all stateless — they read from and write to etcd but store nothing themselves.

---

## What etcd Stores

Every Kubernetes resource object is stored in etcd. The key structure follows this pattern:

```
/registry/{resource-type}/{namespace}/{name}
```

Examples:

```
/registry/pods/default/web-pod-abc123
/registry/pods/production/api-pod-xyz789
/registry/deployments/apps/production/web-deployment
/registry/services/specs/default/web-service
/registry/secrets/default/db-credentials
/registry/configmaps/production/app-config
/registry/namespaces//production
/registry/nodes//ip-10-0-1-100.ec2.internal
```

Cluster-scoped resources (nodes, namespaces) do not have a namespace in the key path. Namespace-scoped resources (pods, deployments, services) include the namespace.

### What a Pod Entry Looks Like in etcd

etcd stores Kubernetes objects in protobuf format (binary). You can read them using `etcdctl`:

```bash
# On a self-managed cluster — NOT on EKS (which does not give you etcd access)
etcdctl get /registry/pods/default/web-pod \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --print-value-only | jq .
```

You would see the full Pod JSON — every field, including computed defaults that Kubernetes added when the Pod was created.

---

## How etcd Achieves Consistency — The Raft Algorithm

Running a single etcd node is a single point of failure. If that node dies, the entire cluster loses its state store and the control plane stops working.

etcd solves this by running as a **cluster of odd-numbered nodes** (3 or 5) and using the **Raft consensus algorithm** to keep all nodes in sync.

### Raft in Simple Terms

Raft works by electing one node as the **Leader**. All writes go to the Leader. The Leader replicates the write to Followers. Once a **majority** (quorum) acknowledges the write, it is committed.

```
          ┌──────────────────────┐
          │    etcd Leader       │
          │  (receives writes)   │
          └─────────┬────────────┘
                    │ replicate
          ┌─────────┼──────────────┐
          │         │              │
          ▼         ▼              ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ Follower │ │ Follower │ │ Follower │
   │ (ack)    │ │ (ack)    │ └──────────┘
   └──────────┘ └──────────┘
   
   Quorum for 3 nodes = 2 acks needed
   Write committed after 2 nodes acknowledge
```

### Quorum and Fault Tolerance

| etcd Nodes | Quorum Needed | Max Failures Tolerated |
|-----------|--------------|----------------------|
| 1 | 1 | 0 (no fault tolerance) |
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

For production, 3 nodes is the minimum. 5 nodes is common for very critical clusters. You almost never go above 7 — the replication overhead outweighs the additional fault tolerance.

AWS EKS runs etcd across 3 Availability Zones automatically. Your cluster can lose an entire AZ and etcd (and your cluster) will continue operating.

### Leader Election

When an etcd Leader fails:

```
Leader fails (crash, network partition, etc.)
        │
        ▼
Followers detect no heartbeat from Leader after ~150-300ms
        │
        ▼
Followers start an election — each votes for itself
        │
        ▼
The first Follower to get a majority of votes becomes the new Leader
        │
        ▼
New Leader starts accepting writes
Total downtime: typically less than 1 second
```

During the election, the API Server cannot write to etcd. Reads that hit cached data may still work, but any operation requiring a write (creating Pods, updating resources) will fail briefly. This is why your `kubectl apply` might return a timeout during a leader election.

---

## etcd Writes — How They Work With Kubernetes

Here is the exact sequence of an etcd write triggered by a `kubectl apply`:

```bash
kubectl apply -f deployment.yaml
```

```
1. kubectl sends HTTP PATCH/PUT to API Server
              │
2. API Server validates the request (auth + admission)
              │
3. API Server serializes the object to protobuf format
              │
4. API Server sends WRITE to etcd Leader:
   PUT /registry/deployments/apps/production/web-deployment
   value: <protobuf-encoded Deployment object>
              │
5. etcd Leader writes to its Write-Ahead Log (WAL)
              │
6. etcd Leader replicates to Followers
              │
7. Two Followers acknowledge (quorum reached)
              │
8. etcd confirms write to API Server
              │
9. API Server returns HTTP 200 to kubectl
              │
10. Controllers watching this key via Watch API are notified
```

The Watch API is what makes controllers reactive. Rather than polling etcd every second, controllers tell etcd "notify me when anything under `/registry/deployments/` changes." etcd pushes notifications instantly when a change is committed.

---

## etcd Performance Characteristics

etcd is designed for consistency, not high throughput. It is optimized for the workload pattern of Kubernetes — many small reads and infrequent writes.

| Metric | Typical etcd Performance |
|--------|-------------------------|
| Write latency | < 10ms (on fast SSDs) |
| Read latency | < 1ms (from cache) |
| Write throughput | 1,000–10,000 writes/second |
| Storage | A healthy cluster uses 1–8 GB |

### Disk Performance Matters

etcd is extremely sensitive to disk write latency. It writes to disk (WAL) on every committed write. If your disks are slow (network-attached storage with high latency), etcd performance degrades and your API Server becomes slow.

For self-managed clusters, always use SSDs (or NVMe drives) for etcd. On AWS, this means **gp3** or **io1/io2** EBS volumes.

On EKS, AWS handles this — etcd runs on optimized, SSD-backed storage that you do not manage.

### etcd Database Size Limit

etcd has a default storage quota of **8GB**. If the database grows beyond this, etcd switches to a read-only state — no new writes are accepted, and the API Server cannot create or update any resources.

This happens when clusters have:
- Large numbers of objects (hundreds of thousands of Pods, events, ConfigMaps)
- Kubernetes Events that accumulate over time (Events have a TTL of 1 hour by default)
- Old ReplicaSet objects not cleaned up after Deployment updates

```bash
# Check etcd database size (self-managed cluster)
etcdctl endpoint status \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  -w table

# Output:
# ENDPOINT              ID        IS LEADER  VERSION  DB SIZE  IN USE  IN USE%
# https://127.0.0.1:2379  abc123  true       3.5.9    3.2 GB   2.1 GB  65%
```

---

## etcd Backup and Restore

### Why Backup etcd

If etcd data is lost:
- All Kubernetes objects are gone — every Deployment, Service, ConfigMap, Secret
- Worker nodes still run containers (the containers themselves are not in etcd), but Kubernetes cannot manage them anymore
- The cluster cannot schedule new Pods, update existing ones, or respond to failures

Without a backup, recovery means starting the cluster from scratch. With a backup, you can restore the complete cluster state.

### Creating an etcd Snapshot (Self-Managed Clusters)

```bash
# Take a snapshot of etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify the snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-20240101-020000.db -w table

# Output:
# HASH        REVISION  TOTAL KEYS  TOTAL SIZE
# 4b825dc6    12543     1892        12 MB
```

### Automating etcd Backup with a CronJob

For self-managed clusters, run etcd backups as a Kubernetes CronJob that saves snapshots to S3:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"    # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          containers:
          - name: etcd-backup
            image: amazon/aws-cli:latest
            command:
            - /bin/sh
            - -c
            - |
              SNAPSHOT_FILE="/tmp/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db"
              
              # Take snapshot
              ETCDCTL_API=3 etcdctl snapshot save $SNAPSHOT_FILE \
                --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/server.crt \
                --key=/etc/kubernetes/pki/etcd/server.key
              
              # Upload to S3
              aws s3 cp $SNAPSHOT_FILE s3://my-cluster-etcd-backups/
              
              echo "Backup completed: $SNAPSHOT_FILE"
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          restartPolicy: OnFailure
```

### Restoring from a Snapshot (Self-Managed Clusters)

```bash
# 1. Stop the API Server and other control plane components first
# (on kubeadm clusters: move the static Pod manifests out of /etc/kubernetes/manifests)

# 2. Restore the snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot-20240101-020000.db \
  --data-dir=/var/lib/etcd-restored \
  --name=master-1 \
  --initial-cluster=master-1=https://192.168.1.10:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://192.168.1.10:2380

# 3. Update etcd configuration to use the new data directory
# 4. Restart control plane components
```

### EKS: You Do Not Manage etcd Backups

On AWS EKS, etcd backup and restore is fully managed by AWS. AWS:
- Continuously backs up etcd to S3 (managed by AWS)
- Restores etcd automatically if a control plane node fails
- Maintains the etcd cluster across 3 AZs

This is the single biggest operational advantage of using EKS over self-managed Kubernetes — you never have to worry about etcd backups.

---

## etcd Compaction and Defragmentation

etcd keeps a history of all changes (revisions) to all keys. Over time, this history accumulates and the database grows. **Compaction** removes old revisions, keeping only the most recent state.

Kubernetes configures etcd to auto-compact every 5 minutes by default. But compaction does not immediately free disk space — it just marks old data as reclaimable. **Defragmentation** actually reclaims the space:

```bash
# Defragment etcd to free disk space (self-managed clusters)
# Run this during low-traffic periods — it takes the member offline briefly
ETCDCTL_API=3 etcdctl defrag \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Output: Finished defragmenting etcd member[https://127.0.0.1:2379]
```

Run defragmentation on one etcd member at a time (never all simultaneously) to avoid taking down the entire cluster.

---

## Summary

- etcd is the distributed key-value store that holds the complete state of a Kubernetes cluster — all resource objects are stored here
- Data is organized under `/registry/{resource-type}/{namespace}/{name}` keys
- etcd uses the **Raft consensus algorithm** — all writes go to the Leader, which replicates to Followers, committing after a majority quorum
- A 3-node etcd cluster tolerates 1 node failure; a 5-node cluster tolerates 2 node failures
- Kubernetes controllers use etcd's **Watch API** to receive instant notifications of changes without polling
- etcd is sensitive to disk latency — always use SSDs for self-managed clusters
- The default storage quota is 8GB — exceeding it puts etcd in read-only mode
- **Backup etcd regularly** on self-managed clusters — use CronJobs to save snapshots to S3 every few hours
- **On AWS EKS, etcd is fully managed** — backups, high availability, and failover are handled automatically

In the next file, `05-api-server.md`, you will go deep into the Kubernetes API Server — the single entry point for all cluster operations, how authentication and authorization work, admission controllers, and how to interact with the API directly.
