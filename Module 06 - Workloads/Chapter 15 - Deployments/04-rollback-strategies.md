# Rollback Strategies

## Introduction

No matter how good your testing is, broken deployments happen in production. When they do, the most important thing is speed — how fast can you restore a working version? Kubernetes Deployments have a built-in rollback mechanism that can restore the previous version in seconds.

This file covers how rollbacks work, how to roll back to any previous revision, and how to design your deployment process so rollbacks are always available and safe.

---

## How Kubernetes Stores Rollback History

Every time you update a Deployment (change image, env vars, or any Pod template field), Kubernetes creates a new **ReplicaSet** for the new version. The old ReplicaSet is kept but scaled to 0 replicas. These old ReplicaSets are your rollback history.

```bash
kubectl get replicasets -n production -l app=api-server

# NAME                      DESIRED   CURRENT   READY   AGE
# api-server-abc123def      3         3         3       2h    ← current (v3.0.2)
# api-server-xyz789pqr      0         0         0       5h    ← revision 4 (v3.0.1)
# api-server-mno456ghi      0         0         0       2d    ← revision 3 (v3.0.0)
# api-server-klm123jkl      0         0         0       5d    ← revision 2 (v2.9.5)
# api-server-efg456abc      0         0         0       10d   ← revision 1 (v2.9.0)
```

The number of old ReplicaSets kept is controlled by `revisionHistoryLimit`:

```yaml
spec:
  revisionHistoryLimit: 5    # keep last 5 revisions (default: 10)
```

Setting this to `0` removes all history — rollback becomes impossible. Always keep at least 3–5 revisions.

---

## Rolling Back to the Previous Revision

```bash
# Immediate rollback to the previous version (most common scenario)
kubectl rollout undo deployment/api-server -n production

# Watch the rollback
kubectl rollout status deployment/api-server -n production
# Waiting for deployment "api-server" rollout to finish: 0 of 3 updated replicas are available...
# deployment "api-server" successfully rolled out

# Verify the image is the previous version
kubectl get deployment api-server -n production \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# 123456789.dkr.ecr.us-east-1.amazonaws.com/api-server:v3.0.1  ← previous version
```

A rollback is just a rolling update in reverse. It follows the same `maxSurge` and `maxUnavailable` parameters — so it is also zero-downtime.

---

## Rolling Back to a Specific Revision

```bash
# View rollout history
kubectl rollout history deployment/api-server -n production

# REVISION  CHANGE-CAUSE
# 1         Initial deployment — v2.9.0
# 2         v2.9.5: Security patch
# 3         v3.0.0: New API endpoints
# 4         v3.0.1: Performance fix
# 5         v3.0.2: Introduced regression — BAD ← current

# Roll back to revision 3 (v3.0.0)
kubectl rollout undo deployment/api-server -n production --to-revision=3

# View details of a revision before rolling back
kubectl rollout history deployment/api-server -n production --revision=3
# Pod Template:
#   Labels: app=api-server, version=3.0.0
#   Image: api-server:v3.0.0
#   ...
```

After a rollback, the revision numbers continue upward — rolling back to revision 3 creates a new revision 6 (identical to revision 3). This preserves the history.

---

## Pausing and Resuming a Rollout

During a gradual rollout, you may want to pause to observe metrics before continuing:

```bash
# Start a rollout
kubectl set image deployment/api-server \
  api=api-server:v3.1.0 -n production

# Immediately pause — stops at current state (some old Pods, some new)
kubectl rollout pause deployment/api-server -n production

# At this point, check your metrics dashboards:
# - Error rate elevated? → rollback immediately
# - Latency acceptable? → resume the rollout

# Resume the rollout if metrics look good
kubectl rollout resume deployment/api-server -n production

# OR rollback if metrics look bad
kubectl rollout undo deployment/api-server -n production
```

---

## Ensuring Rollback is Always Available — Best Practices

### 1. Never Delete Old Images from ECR During Rollback Window

```bash
# Keep N recent image tags in ECR
# Do NOT run cleanup scripts that delete images within the rollback window (e.g., last 7 days)
aws ecr batch-delete-image \
  --repository-name api-server \
  --image-ids imageTag=v2.8.0   # only safe to delete very old versions
```

### 2. Maintain Database Backward Compatibility

The most dangerous deployment scenario: you deploy a new application version that runs a database migration — and then need to rollback. If the old application cannot work with the new schema, you are stuck.

**Solution: Expand-Contract pattern (also known as blue-green migrations)**

```
Step 1: Deploy a database migration that only ADDS new columns (backward compatible)
        Old app: ignores new columns — still works ✅
        New app: uses new columns ✅

Step 2: Deploy new application version
        If rollback needed → old app still works with new schema ✅

Step 3 (days later): Remove old columns (contract phase)
        Only after you are confident the rollback window is closed
```

Never remove columns or change column types in the same migration as the application deployment.

### 3. Use `revisionHistoryLimit` Appropriately

```yaml
spec:
  revisionHistoryLimit: 10    # for production: keep at least 10 revisions
                              # 10 revisions × typical 1 deploy/day = 10 days of rollback history
```

### 4. Test Rollback in Staging

Add a rollback test to your staging pipeline:
1. Deploy version N
2. Deploy version N+1
3. Rollback to version N
4. Verify the service is healthy after rollback

---

## Rollback in CI/CD Pipelines — Automated Rollback on Failure

```bash
#!/bin/bash
# deploy.sh — deploy and auto-rollback on failure

set -e
DEPLOYMENT="api-server"
NAMESPACE="production"
NEW_IMAGE="123456789.dkr.ecr.us-east-1.amazonaws.com/api-server:${VERSION}"

echo "Deploying ${NEW_IMAGE}..."
kubectl set image deployment/${DEPLOYMENT} \
  api=${NEW_IMAGE} \
  -n ${NAMESPACE}

echo "Waiting for rollout..."
if ! kubectl rollout status deployment/${DEPLOYMENT} \
  -n ${NAMESPACE} --timeout=300s; then
  
  echo "Deployment FAILED — rolling back..."
  kubectl rollout undo deployment/${DEPLOYMENT} -n ${NAMESPACE}
  
  echo "Waiting for rollback to complete..."
  kubectl rollout status deployment/${DEPLOYMENT} -n ${NAMESPACE}
  
  echo "Rollback complete"
  exit 1  # signal CI/CD failure to trigger alert
fi

echo "Deployment successful: ${NEW_IMAGE}"
```

---

## Checking What Changed Between Revisions

```bash
# Get the Pod template for two revisions and diff them
kubectl rollout history deployment/api-server -n production --revision=4 -o json > rev4.json
kubectl rollout history deployment/api-server -n production --revision=5 -o json > rev5.json
diff rev4.json rev5.json

# Or use kubectl directly to inspect the ReplicaSet spec for each revision
# Find the ReplicaSet for revision 5
kubectl get replicaset -n production -l app=api-server -o json | \
  jq -r '.items[] | select(.metadata.annotations["deployment.kubernetes.io/revision"] == "5") | .spec.template.spec.containers[0].image'
# api-server:v3.0.2  ← what revision 5 deployed
```

---

## Summary

- Rollback history is stored as old ReplicaSets (scaled to 0) — `revisionHistoryLimit` controls how many are kept
- `kubectl rollout undo` rolls back to the previous revision instantly — it is a zero-downtime rolling update in reverse
- `kubectl rollout undo --to-revision=N` rolls back to any specific historical revision
- Pause a rollout mid-way with `kubectl rollout pause` to observe metrics before continuing or rolling back
- Make database migrations backward-compatible (Expand-Contract pattern) — always ensure the previous app version can run against the new schema
- Automate rollback in CI/CD pipelines: check `kubectl rollout status` exit code — non-zero triggers automatic `rollout undo`

In the next file, `05-deployment-strategies.md`, you will learn advanced deployment strategies — Blue-Green, Canary, and Recreate — with their trade-offs and how to implement them on Kubernetes.
