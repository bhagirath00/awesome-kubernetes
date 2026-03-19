# CronJob Best Practices

## Introduction

In a production environment, a CronJob that just "runs a script" is not enough. Scheduled tasks are often the most critical parts of an infrastructure — they handle backups, billing, data cleanup, and security audits. If these tasks fail silently or run incorrectly, the damage can be catastrophic.

To build a "Gold Standard" DevOps pipeline, you must follow strict best practices around idempotency, observability, resource management, and security. This guide goes deep into how to make your CronJobs resilient and production-ready.

---

## 1. The Principle of Idempotency

**Idempotency** is the most important rule of batch processing. An idempotent task is one that can be run multiple times, and the result will always be the same as if it were run once.

Kubernetes provides an **"at least once"** delivery guarantee for CronJobs. This means that under rare circumstances (like a controller restart or network hiccup), the same Job might be triggered twice for the same schedule. If your script is not idempotent, you might double-bill a customer or corrupt a database.

### Bad Pattern (Non-Idempotent)
```bash
# This script appends a row. Running it twice creates two rows.
echo "Run started at $(date)" >> /mnt/data/log.txt
sql-client "INSERT INTO DailySales (total) SELECT SUM(amount) FROM Orders WHERE date = 'yesterday'"
```

### Good Pattern (Idempotent)
```bash
# This script ensures it only runs once per day.
# 1. Check if work is already done
if [ -f /mnt/data/done_$(date +%F).txt ]; then
    echo "Task already completed for today. Exiting."
    exit 0
fi

# 2. Perform work with 'Upsert' or 'Delete-then-Insert' logic
sql-client "DELETE FROM DailySales WHERE date = 'yesterday'; \
            INSERT INTO DailySales (total) SELECT SUM(amount) FROM Orders WHERE date = 'yesterday'"

# 3. Mark as done
touch /mnt/data/done_$(date +%F).txt
```

---

## 2. Strategic Concurrency Management

Never leave `concurrencyPolicy` to the default (`Allow`) for critical tasks. You must choose based on the nature of the data.

| Policy | Use Case | Example |
| :--- | :--- | :--- |
| **Forbid** | Data integrity is critical. Multiple writers would cause a race condition. | Database backup, billing invoice generation, financial ledger sync. |
| **Replace** | Only the latest state matters. Old data is immediately stale. | Generating a "Cluster Health" dashboard, refreshing an in-memory cache. |
| **Allow** | High-volume, stateless tasks that don't share resources. | Sending transactional email notifications, small log scraping. |

**Pro-Tip**: If you use `Forbid`, monitor the "Missed Schedules" metric. If a job is frequently forbidden, it means your task is taking longer than the interval you provided. You need to either optimize the code or increase the schedule interval.

---

## 3. Resource Isolation and Limits

CronJobs often run heavy, resource-intensive tasks. If you don't set limits, a single CronJob can "starve" your production web servers by consuming all the CPU on a node.

### The "Bursty" Nature of Jobs
Unlike web servers, which have steady traffic, Jobs are bursty. They go from 0 to 100% CPU usage instantly.

```yaml
resources:
  requests:
    cpu: "500m"     # Guarantee half a core
    memory: "1Gi"   # Guarantee 1GB RAM
  limits:
    cpu: "2"        # Kill the job if it tries to take more than 2 cores
    memory: "4Gi"   # Prevent OOM (Out Of Memory) from crashing the entire node
```

**AWS Optimization**: On EKS, consider using **Spot Instances** for your CronJobs. Since Jobs are meant to be retried and can handle interruptions, running them on Spot can save up to 90% in costs compared to On-Demand instances.

---

## 4. Observability: Don't Let it Fail Silently

A CronJob that fails in the middle of the night and stays failed for a week is a nightmare. You must have automated alerting.

### Pattern A: Prometheus Metrics
Kubernetes exposes metrics about Jobs. You should set up alerts in Prometheus for:
- `kube_job_status_failed > 0`: Alert immediately if any Job fails.
- `time() - kube_cronjob_next_schedule_time > 3600`: Alert if a Job hasn't run even though it should have.

### Pattern B: The "Dead Man's Snitch" (Heartbeats)
This is the most reliable way to monitor CronJobs. The script itself "pings" an external service at the end of its run. If the service doesn't receive the ping, it sends an alert (Slack/PagerDuty).

```bash
# Final command in your container
./my-critical-script.sh && curl -fsS -m 10 --retry 5 https://hchk.io/your-unique-id
```

---

## 5. Security and Least Privilege

CronJobs often need access to high-level secrets (database passwords, AWS keys). Do not use the default ServiceAccount.

### Dedicated ServiceAccounts with IRSA (AWS)
Instead of putting AWS credentials in a Secret, use **IAM Roles for Service Accounts (IRSA)**.

1.  Create an IAM Role with a policy like `S3FullAccess`.
2.  Create a ServiceAccount and annotate it with the Role ARN.
3.  Assign the ServiceAccount to the CronJob.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backup-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/BackupRole
```

This way, even if someone "hacks" the container, they don't get a static key; they only get the temporary permissions assigned to that Pod.

---

## 6. Managing Time Zones and Starting Deadlines

### Time Zone Awareness
Standard CronJobs run in **UTC**. If you schedule a job for `0 0 * * *` (midnight) and you are in New York, it will run at 7:00 PM or 8:00 PM depending on Daylight Savings. 

In Kubernetes 1.27+, use the `timeZone` field to fix this.

```yaml
spec:
  schedule: "0 0 * * *"
  timeZone: "America/New_York" # Handles Daylight Savings automatically
```

### The Safety Net: `startingDeadlineSeconds`
If the CronJob controller crashes or the cluster is busy, a Job might miss its start time. By default, Kubernetes will try to "catch up" by running all missed jobs.

```yaml
spec:
  startingDeadlineSeconds: 60 # If we are more than 60s late, just skip this run.
```
**Never omit this.** It prevents a "thundering herd" of 100 missed jobs from running at once when a cluster recovers.

---

## 7. Troubleshooting and History

Keep enough history to debug, but not enough to clutter `etcd`.

```yaml
spec:
  successfulJobsHistoryLimit: 3 # Last 3 successes are enough to see patterns
  failedJobsHistoryLimit: 10    # Keep more failures so you can investigate logs
```

**Pro-Tip**: Use labels like `job-version: v1.2` in your `jobTemplate`. When a job fails, you can immediately see which version of the code was running.

---

## Summary

- **Idempotency** is your protection against duplicate data and race conditions.
- **Concurrency Policies** (especially `Forbid`) prevent resource exhaustion on your nodes.
- **Resource Limits** ensure that background tasks don't crash your user-facing applications.
- **Monitoring** via Prometheus or Heartbeats is mandatory; don't assume a Job is running because it is scheduled.
- **Security** should always use IRSA for AWS resources to avoid static credentials.
- **`startingDeadlineSeconds`** is a critical safety feature to prevent massive bursts of old jobs after a failure.

In the next file, `05-cronjob-troubleshooting.md`, you will learn exactly how to fix these tasks when they fail, focusing on log analysis and event tracing.
