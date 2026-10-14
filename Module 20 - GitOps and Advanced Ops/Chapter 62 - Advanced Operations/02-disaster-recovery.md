# Kubernetes Disaster Recovery: Architecture, Tiers, & Runbooks

Disaster Recovery (DR) in Kubernetes is not just about taking backups; it requires an architectural strategy to restore services within strict business SLA limits when hardware fails, data centers burn down, or catastrophic human error occurs (such as accidental `kubectl delete namespace prod`).

---

## 1. RTO, RPO, and Failure Domains

```
   [ Incident Occurs ] <------- Data Loss (RPO) -------> [ Last Consistent Backup ]
            |
            +------------ Recovery Time (RTO) -----------> [ Service Fully Restored ]
```

- **RPO (Recovery Point Objective)**: The maximum acceptable age of files/data that must be recovered from backup storage for normal operations to resume (e.g., maximum 15 minutes of lost data).
- **RTO (Recovery Time Objective)**: The maximum acceptable duration of time that application services can remain offline after a disaster is declared (e.g., cluster restored within 1 hour).

### Kubernetes Failure Domains

| Failure Domain | Scope of Impact | Primary Mitigation |
| :--- | :--- | :--- |
| **Worker Node** | Pods evicted and rescheduled | Multi-replica Deployments, PodDisruptionBudgets, TopologySpreadConstraints |
| **Availability Zone (AZ)** | 1/3 of cluster resources lost | Multi-AZ Worker & Control Plane nodes, Regional PVs |
| **etcd Corruption** | API server unresponsive | Automated hourly etcd snapshots to remote object storage |
| **Entire Region / DC** | Complete cluster down | Cross-region Active-Passive or Active-Active clusters |
| **Human Catastrophe** | Deleted CRDs, namespaces, PVs | GitOps reconciliation + Velero Volume Snapshots |

---

## 2. Disaster Recovery Architecture Tiers

```
+-----------------------------------------------------------------------------------------+
| DR Tier          | RTO / RPO        | Cost   | Architecture                             |
+-----------------------------------------------------------------------------------------+
| Tier 1: Cold     | RTO: Hours       | $      | - Single active cluster.                  |
| Standby          | RPO: 1 - 24 hrs  |        | - GitOps repo + Daily Velero S3 backups. |
|                  |                  |        | - Spin up new cluster only after disaster.|
+-----------------------------------------------------------------------------------------+
| Tier 2: Warm     | RTO: 15 - 30 min | $$     | - Primary cluster active.                 |
| Standby (Pilot)  | RPO: < 15 min    |        | - Secondary standby cluster running mini- |
|                  |                  |        |   mal nodes; data replicated continuously.|
+-----------------------------------------------------------------------------------------+
| Tier 3: Hot      | RTO: Seconds     | $$$$   | - Dual active clusters in two regions.    |
| (Active-Active)  | RPO: Near zero   |        | - Global Anycast/DNS splits traffic.     |
|                  |                  |        | - Distributed database (Cockroach/Spanner)|
+-----------------------------------------------------------------------------------------+
```

---

## 3. Stateless vs. Stateful Workloads

Disaster recovery must strictly separate stateless compute from stateful persistent data:

### Stateless Applications
- **Strategy**: GitOps is your primary DR tool.
- If an entire Kubernetes cluster is destroyed, provisioning a new cluster via Terraform and pointing Flux/ArgoCD at the Git repository automatically restores 100% of the stateless deployments, configmaps, services, and ingress rules.
- **RTO**: Minutes (limited only by cloud VM provisioning speed).
- **RPO**: 0 (Git is the declarative ground truth).

### Stateful Applications
- **Strategy**: GitOps cannot restore Persistent Volume contents.
- Requires coordinated storage snapshotting (CSI VolumeSnapshots, Velero, or application-level database replication like PostgreSQL streaming replication or Kafka MirrorMaker).

---

## 4. etcd Backup & Restoration Strategy

For self-managed Kubernetes control planes (Kubeadm, Talos, bare-metal), etcd is the heart of the cluster.

### Taking an etcd Snapshot
```bash
# Save snapshot using etcdctl
ETCDCTL_API=3 etcdctl snapshot save /var/backups/etcd/snapshot-$(date +%Y%m%d%H%M).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot integrity
ETCDCTL_API=3 etcdctl snapshot status /var/backups/etcd/snapshot-*.db --write-out=table
```

### Automated etcd CronJob to AWS S3 / Cloud Storage
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */1 * * *" # Every hour
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          tolerations:
            - key: node-role.kubernetes.io/control-plane
              operator: Exists
              effect: NoSchedule
          containers:
            - name: etcd-backup
              image: bitnami/etcd:3.5.12
              command:
                - /bin/sh
                - -c
                - |
                  export ETCDCTL_API=3
                  BACKUP_FILE="/tmp/etcd-backup-$(date +%s).db"
                  etcdctl snapshot save $BACKUP_FILE \
                    --endpoints=https://127.0.0.1:2379 \
                    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                    --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
                    --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
                  # Ship backup to remote encrypted S3 bucket
                  aws s3 cp $BACKUP_FILE s3://corp-k8s-etcd-backups/prod-cluster/
              volumeMounts:
                - mountPath: /etc/kubernetes/pki/etcd
                  name: etcd-certs
                  readOnly: true
          volumes:
            - name: etcd-certs
              hostPath:
                path: /etc/kubernetes/pki/etcd
          restartPolicy: OnFailure
```

---

## 5. Traffic Failover & Global DNS Routing

During a regional outage, external ingress traffic must be redirected to the surviving DR site without human intervention.

```
                         [ External Users ]
                                  |
                                  v
                  +-------------------------------+
                  |  Route53 / Cloudflare Anycast |
                  |  Health Check: /healthz       |
                  +---------------+---------------+
                                  |
               +------------------+------------------+
               | Primary (Healthy)                   | Secondary (Failover)
               v                                     v
     +-------------------+                 +-------------------+
     | Primary Cluster   |                 | DR Standby        |
     | Region: us-east-1 |                 | Region: us-west-2 |
     | NLB / Ingress     |                 | NLB / Ingress     |
     +-------------------+                 +-------------------+
```

### Route 53 DNS Failover Configuration
- **Primary Record**: Points to Primary Cluster Load Balancer. Evaluated against an HTTP health check querying `https://api.company.com/healthz`.
- **Failover (Secondary) Record**: Points to Standby Cluster Load Balancer. Activated automatically within 60 seconds of 3 consecutive primary health check failures.

---

## 6. Disaster Recovery Runbook & Checklist

### Phase 1: Triage & Disaster Declaration (T+0 to T+10m)
1. Verify cluster API server connectivity: `kubectl get nodes --request-timeout='5s'`.
2. Determine if the issue is an AZ blip or full region outage via Cloud Status dashboards.
3. If primary control plane or cloud region is confirmed irrecoverable, officially declare DR.

### Phase 2: Standby Promotion & GitOps Activation (T+10m to T+30m)
1. In the secondary cluster, scale worker node autoscaling groups:
   ```bash
   aws autoscaling update-auto-scaling-group --auto-scaling-group-name k8s-dr-workers --min-size 10 --desired-capacity 20
   ```
2. Resume GitOps sync on the DR cluster to deploy platform dependencies:
   ```bash
   flux resume kustomization --all
   # OR: argocd app sync --all
   ```
3. Restore stateful persistent volumes from the latest consistent Velero backup:
   ```bash
   velero restore create --from-backup prod-daily-backup-latest --include-namespaces payments,databases
   ```

### Phase 3: Traffic Cutover & Verification (T+30m to T+45m)
1. Run smoke tests against DR ingress endpoints:
   ```bash
   curl -I https://dr-ingress.company.com/healthz
   ```
2. Update DNS pointer or trigger Route53 failover override.
3. Monitor application error rates and request logs in Datadog/Grafana.

### Phase 4: Post-Mortem & Failback Planning
- Never fail back during peak production hours.
- Perform reverse data synchronization before pointing DNS back to the primary cluster.
