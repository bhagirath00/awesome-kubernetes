# Rolling Updates

## Introduction

A rolling update is how Kubernetes replaces old Pods with new ones during a Deployment update — gradually, without downtime. Instead of deleting all old Pods and creating all new ones simultaneously (which causes downtime), a rolling update creates a few new Pods, waits for them to be healthy, then removes a few old ones, repeating until all Pods run the new version.

Understanding the rolling update parameters — `maxSurge` and `maxUnavailable` — is critical for achieving zero-downtime deployments.

---

## How a Rolling Update Works

Starting state: 3 replicas running `web-app:v1.0`

```
ReplicaSet v1 (web-app:v1.0):  Pod-A  Pod-B  Pod-C
```

You apply a change: `image: web-app:v2.0`

Kubernetes starts the rolling update with `maxSurge: 1, maxUnavailable: 0`:

```
Step 1: Create 1 new Pod (surge)
ReplicaSet v1: Pod-A  Pod-B  Pod-C   (3 running)
ReplicaSet v2: Pod-D                  (1 starting) ← new Pod
Total: 4 Pods (3 desired + 1 surge)

Step 2: Wait for Pod-D to pass readiness probe
Pod-D is now Ready → traffic routes to Pod-D

Step 3: Remove 1 old Pod (maxUnavailable: 0 means we can only remove after new is Ready)
ReplicaSet v1: Pod-B  Pod-C          (2 running)
ReplicaSet v2: Pod-D  Pod-E          (starting)

Step 4: Wait for Pod-E Ready → remove Pod-B
Step 5: Wait for Pod-F Ready → remove Pod-C

Final state:
ReplicaSet v1: (scaled to 0)
ReplicaSet v2: Pod-D  Pod-E  Pod-F   (3 running, web-app:v2.0)
```

---

## The Two Parameters — maxSurge and maxUnavailable

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1              # max extra Pods above desired count during update
    maxUnavailable: 0        # max Pods that can be unavailable during update
```

**`maxSurge`** — how many extra Pods can exist above the desired replica count during the update.
- `maxSurge: 1` with `replicas: 3` → maximum 4 Pods at any time
- `maxSurge: 25%` → rounds up: 25% of 3 = 0.75 → 1 extra Pod
- `maxSurge: 0` → no extra Pods — must terminate old before starting new (rolling in place — slower, never exceeds replica count)

**`maxUnavailable`** — how many Pods can be below the desired count at any time.
- `maxUnavailable: 0` → always maintain full replica count (zero-downtime, default for prod)
- `maxUnavailable: 1` with `replicas: 3` → can drop to 2 healthy Pods during update (faster, brief capacity reduction)
- `maxUnavailable: 25%` → rounds down: 25% of 3 = 0.75 → 0 (same as 0 for 3 replicas)

---

## Choosing the Right Parameters

### Zero-Downtime Production Setting

```yaml
# maxSurge: 1, maxUnavailable: 0
# - New Pods are created before old ones are removed
# - Always full capacity available
# - Needs extra node capacity (briefly runs 4 Pods instead of 3)
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

### Faster Update (Acceptable Brief Capacity Reduction)

```yaml
# maxSurge: 1, maxUnavailable: 1
# - Can create 1 new AND remove 1 old simultaneously
# - Update is 2x faster
# - Briefly at n-1 capacity (OK for over-provisioned services)
strategy:
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

### Canary-Style (Very Gradual)

```yaml
# maxSurge: 1, maxUnavailable: 0
# replicas: 10
# → adds 1 new Pod (10% of traffic), waits for health, removes 1 old Pod
# → continues until all 10 are updated
# → natural 10% canary at each step
strategy:
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

---

## Triggering a Rolling Update

```bash
# Method 1: Update the image in YAML and apply
# Edit deployment.yaml: image: web-app:v2.0
kubectl apply -f deployment.yaml

# Method 2: Set image directly (recorded in rollout history)
kubectl set image deployment/api-server \
  api=123456789.dkr.ecr.us-east-1.amazonaws.com/api-server:v3.0.2 \
  -n production \
  --record      # deprecated but still supported for history

# Method 3: Update image with annotation for history
kubectl set image deployment/api-server \
  api=123456789.dkr.ecr.us-east-1.amazonaws.com/api-server:v3.0.2 \
  -n production
kubectl annotate deployment/api-server \
  kubernetes.io/change-cause="v3.0.2: Fix CPU spike on high load" \
  -n production --overwrite

# Method 4: Restart without image change (forces new Pods — picks up configmap changes)
kubectl rollout restart deployment/api-server -n production
```

---

## Monitoring a Rolling Update

```bash
# Watch rollout progress (blocks until complete or fails)
kubectl rollout status deployment/api-server -n production

# Waiting for deployment "api-server" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "api-server" rollout to finish: 1 old replicas are pending termination...
# Waiting for deployment "api-server" rollout to finish: 2 of 3 updated replicas are available...
# deployment "api-server" successfully rolled out

# Watch Pods change in real time
kubectl get pods -n production -l app=api-server -w

# NAME                         READY   STATUS              RESTARTS   AGE
# api-server-abc-aaa           1/1     Running             0          2d
# api-server-abc-bbb           1/1     Running             0          2d
# api-server-abc-ccc           1/1     Running             0          2d
# api-server-xyz-ddd           0/1     Pending             0          2s     ← new Pod starting
# api-server-xyz-ddd           0/1     ContainerCreating   0          3s
# api-server-xyz-ddd           1/1     Running             0          18s    ← new Pod Ready
# api-server-abc-aaa           1/1     Terminating         0          2d     ← old Pod removed
# api-server-xyz-eee           0/1     Pending             0          1s     ← next new Pod

# Watch ReplicaSets during the update
kubectl get replicasets -n production -l app=api-server -w

# See events during the rollout
kubectl describe deployment api-server -n production | grep -A 20 "Events:"
```

---

## What Happens When a Rolling Update Gets Stuck

If a new Pod never passes its readiness probe, the rolling update halts — it does not remove old Pods (because new Pods are not healthy):

```bash
# Deployment shows incomplete update
kubectl get deployment api-server -n production
# NAME         READY   UP-TO-DATE   AVAILABLE
# api-server   3/3     1            3          ← 1 UP-TO-DATE but 3 AVAILABLE (2 old + 0 new ready)

# Check what's wrong with the new Pod
kubectl get pods -n production -l app=api-server
# api-server-old-aaa   1/1     Running          ← old
# api-server-old-bbb   1/1     Running          ← old
# api-server-old-ccc   1/1     Running          ← old
# api-server-new-xyz   0/1     Running          ← new, not ready

kubectl describe pod api-server-new-xyz -n production | grep -A 10 "Readiness"
# Readiness probe failed: HTTP probe failed with statuscode: 500

# Check application logs
kubectl logs api-server-new-xyz -n production
# Error: cannot connect to database: dial tcp postgres-service:5432: connection refused

# Fix: after identifying the problem, rollback to the previous version
kubectl rollout undo deployment/api-server -n production
```

---

## `progressDeadlineSeconds` — Automatic Failure Detection

```yaml
spec:
  progressDeadlineSeconds: 300    # fail if rollout doesn't progress within 5 minutes
```

If the rolling update makes no progress (no new Pod becomes Ready) within this time, the Deployment condition `Progressing` is set to `False` with `ProgressDeadlineExceeded`. Kubernetes does NOT automatically rollback — it just reports the failure.

In CI/CD pipelines, use `kubectl rollout status --timeout=5m` to detect stuck rollouts:

```bash
# In your CI/CD pipeline deploy step:
kubectl set image deployment/api-server \
  api=123456789.dkr.ecr.us-east-1.amazonaws.com/api-server:v3.0.2 \
  -n production

# Wait for rollout — exits with error code if it fails or times out
kubectl rollout status deployment/api-server -n production --timeout=300s

# Non-zero exit code triggers CI/CD failure → automatic rollback triggered
if [ $? -ne 0 ]; then
  echo "Deployment failed — rolling back"
  kubectl rollout undo deployment/api-server -n production
  exit 1
fi
```

---

## Summary

- Rolling updates replace Pods gradually — `maxSurge` controls how many extra Pods exist during the update; `maxUnavailable` controls how many Pods can be down at once
- `maxSurge: 1, maxUnavailable: 0` is the zero-downtime production setting — new Pods are created and proven healthy before any old Pods are removed
- Trigger rollouts by updating `image` in YAML and applying; use `kubectl rollout restart` to force new Pods without changing the image
- Monitor with `kubectl rollout status` — it blocks until the rollout completes or times out
- If a new Pod fails its readiness probe, the rollout halts — old Pods are NOT removed (safe fail)
- Set `progressDeadlineSeconds` and use `kubectl rollout status --timeout` in CI/CD pipelines to detect and auto-rollback stuck deployments

In the next file, `04-rollback-strategies.md`, you will learn how Kubernetes rollbacks work — how to undo a deployment, roll back to a specific revision, and design your deployment process so rollbacks are always safe and fast.
