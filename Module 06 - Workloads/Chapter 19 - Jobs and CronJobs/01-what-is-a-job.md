# What is a Job

## Introduction

A **Job** creates one or more Pods to perform a specific task and ensures that task completes successfully. Unlike a Deployment (which keeps Pods running indefinitely) or a DaemonSet (which runs Pods on every node), a Job runs to completion — and when the Pod finishes successfully, the Job is done.

Jobs are the right workload for: batch data processing, database migrations, one-time report generation, ETL pipelines, and any task with a definite end.

---

## How a Job Works

```
Job created
    │
    ▼
Pod(s) created to run the task
    │
    ├─ Pod exits with code 0 (success) → Job records 1 success → done
    │
    └─ Pod exits with non-zero (failure)
         │
         └─ Job creates a new Pod → retries up to backoffLimit times
              ├─ Eventually succeeds → Job done
              └─ All retries exhausted → Job marked as Failed
```

A Job is complete when the required number of successful Pod completions is reached (controlled by `completions`).

---

## Job YAML

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: database-migration
  namespace: production
spec:
  # Number of successful completions required
  completions: 1          # default: 1 — one successful run

  # How many Pods to run in parallel
  parallelism: 1          # default: 1 — run one Pod at a time

  # Max retries on failure before marking the Job as Failed
  backoffLimit: 3         # default: 6

  # Delete the Job (and Pods) automatically after N seconds
  ttlSecondsAfterFinished: 86400   # keep for 24 hours after completion

  # How long to allow the Job to run before killing it
  activeDeadlineSeconds: 600       # fail if not complete within 10 minutes

  template:
    metadata:
      labels:
        app: database-migration
        job: v3.0-migration
    spec:
      # REQUIRED for Jobs — never restart (one-shot execution)
      restartPolicy: Never
      # restartPolicy: OnFailure → restarts the SAME Pod on failure (uses backoffLimit differently)
      
      containers:
      - name: migrate
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/migrations:v3.0
        command:
        - python
        - manage.py
        - migrate
        env:
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_HOST
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-password
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
```

```bash
kubectl apply -f database-migration.yaml

# Monitor the Job
kubectl get job database-migration -n production -w

# NAME                  COMPLETIONS   DURATION   AGE
# database-migration    0/1           5s         5s    ← running
# database-migration    1/1           23s        23s   ← complete!

# See the Pod that ran the Job
kubectl get pods -n production -l job-name=database-migration

# NAME                        READY   STATUS      RESTARTS
# database-migration-abc12    0/1     Completed   0    ← finished, exit code 0

# View Job output
kubectl logs database-migration-abc12 -n production
# Running migrations...
# Applying migrations: 0001_add_orders_table... OK
# Applying migrations: 0002_add_users_index... OK
# All migrations applied successfully.
```

---

## `restartPolicy` — `Never` vs `OnFailure`

The `restartPolicy` field for Jobs is critical:

**`restartPolicy: Never`** (recommended):
```
Pod exits with code 1 (failure)
→ Pod is marked as Failed
→ Job creates a NEW Pod for the retry
→ Old failed Pods stay (for log inspection) up to backoffLimit

After backoffLimit retries, Job is marked Failed
All failed Pods remain — you can inspect their logs
```

**`restartPolicy: OnFailure`**:
```
Pod exits with code 1 (failure)
→ Kubelet restarts the SAME container within the same Pod
→ Logs from previous attempt are lost after restart
→ Harder to debug — you see only the last run's output
```

Use `Never` for production Jobs — it preserves logs from each failed attempt for debugging.

---

## Checking Job Status and Completion

```bash
# Job summary
kubectl describe job database-migration -n production

# Name:           database-migration
# Namespace:      production
# Parallelism:    1
# Completions:    1
# Start Time:     Mon, 15 Jan 2024 10:00:00
# Completed At:   Mon, 15 Jan 2024 10:00:23
# Duration:       23s
# Pods Statuses:  0 Running / 1 Succeeded / 0 Failed

# Conditions:
#   Type     Status   Reason
#   ----     ------   ------
#   Complete True
```

---

## Deleting a Job — Keeping the Logs Available

By default, Jobs and their Pods are not automatically deleted. Configure `ttlSecondsAfterFinished` to clean up:

```yaml
spec:
  ttlSecondsAfterFinished: 3600   # delete Job and Pods 1 hour after completion
```

Without this, completed Jobs accumulate indefinitely. With this, Kubernetes garbage-collects them after the specified time — but you only have that window to inspect logs.

```bash
# Manually delete a completed Job (also deletes its Pods)
kubectl delete job database-migration -n production

# Delete Job but keep Pods (for log inspection)
kubectl delete job database-migration -n production --cascade=orphan
```

---

## Running a Job in CI/CD Pipelines

Jobs are the standard way to run database migrations in a Kubernetes-based CI/CD pipeline:

```bash
#!/bin/bash
# deploy.sh — run migration then deploy application

NAMESPACE="production"
IMAGE="123456789.dkr.ecr.us-east-1.amazonaws.com/migrations:${VERSION}"

echo "Running database migrations..."

# Create the Job from template
cat << EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: migration-${VERSION//[.]/-}   # replace dots with dashes for valid name
  namespace: ${NAMESPACE}
spec:
  ttlSecondsAfterFinished: 3600
  backoffLimit: 2
  activeDeadlineSeconds: 300
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: ${IMAGE}
        command: ["python", "manage.py", "migrate"]
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
EOF

# Wait for Job completion
echo "Waiting for migration to complete..."
kubectl wait job/migration-${VERSION//[.]/-} \
  -n ${NAMESPACE} \
  --for=condition=complete \
  --timeout=300s

if [ $? -ne 0 ]; then
  echo "Migration FAILED — aborting deployment"
  kubectl logs -l job-name=migration-${VERSION//[.]/-} -n ${NAMESPACE}
  exit 1
fi

echo "Migration successful — deploying application..."
kubectl set image deployment/api-server api=${APP_IMAGE} -n ${NAMESPACE}
```

---

## Summary

- A Job creates Pods to run a task to completion — the Job is done when the required number of successful Pod completions is reached
- `restartPolicy: Never` is recommended for production Jobs — each retry creates a new Pod, preserving failed Pod logs for debugging
- `backoffLimit` controls how many times a failed Pod is retried before the Job is marked Failed
- `ttlSecondsAfterFinished` automatically deletes completed Jobs and their Pods after a specified time
- `activeDeadlineSeconds` is the maximum runtime — Jobs that exceed this time are killed and marked Failed
- In CI/CD, use `kubectl wait job/<name> --for=condition=complete` to block the pipeline until the migration Job succeeds

In the next file, `02-job-patterns.md`, you will learn the patterns for running Jobs at scale — parallel completion, work queue, indexed, and batch processing patterns used in data pipelines.
