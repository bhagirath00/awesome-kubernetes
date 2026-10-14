# Upgrading Releases

## Upgrade Overview

Update installed release to new version or configuration:

```bash
helm upgrade [release-name] [chart] [options]

# Examples:
helm upgrade my-app bitnami/nginx              # Upgrade to latest
helm upgrade my-app bitnami/nginx --version 14 # Upgrade to specific version
helm upgrade my-app ./local-chart              # Upgrade from local chart
```

---

## Basic Upgrade

```bash
# Upgrade to new chart version
helm upgrade my-app bitnami/nginx

# Upgrade in namespace
helm upgrade my-app bitnami/nginx -n production

# Upgrade with new values
helm upgrade my-app bitnami/nginx \
  --set replicaCount=10 \
  --set image.tag=v2.0.0

# Upgrade from values file
helm upgrade my-app bitnami/nginx -f new-values.yaml
```

---

## Test Before Upgrade

```bash
# Dry-run (see what would change)
helm upgrade my-app bitnami/nginx --dry-run --debug

# Show diff (with helm-diff plugin)
helm diff upgrade my-app bitnami/nginx -f new-values.yaml

# Install dry-run plugin
helm plugin install https://github.com/databus23/helm-diff
```

---

## Real-World Upgrade Scenarios

### Scenario 1: Version Upgrade

```bash
# Current version
helm list
# my-app  1   2024-01-15 DEPLOYED  nginx-13.2.0

# Upgrade to new chart version
helm upgrade my-app bitnami/nginx --version 14.0.0

# Verify upgrade
helm list
# my-app  2   2024-01-15 DEPLOYED  nginx-14.0.0
```

### Scenario 2: Configuration Change

```bash
# Add persistence (upgrade existing)
helm upgrade my-app bitnami/nginx \
  --set persistence.enabled=true \
  --set persistence.size=50Gi

# Verify
helm get values my-app
```

### Scenario 3: Rolling Update

```bash
# Update image tag (rolling deployment)
helm upgrade my-app bitnami/nginx \
  --set image.tag=v2.0.0

# Kubernetes performs rolling update
# Pods gradually replaced with new image

# Watch progress
kubectl rollout status deployment/my-app
```

---

## Upgrade Strategies

### Strategy 1: Atomic (All or Nothing)

```bash
# Rollback if upgrade fails
helm upgrade my-app bitnami/nginx --atomic

# If anything fails:
# - Rollback automatically
# - Release stays at previous version
```

### Strategy 2: Force Replace

```bash
# Delete and recreate pods (downtime)
helm upgrade my-app bitnami/nginx --force

# Use when:
# - ConfigMap must change
# - PVC needs recreation
# - Rolling update doesn't work
```

### Strategy 3: With Cleanup

```bash
# Remove old resources on upgrade
helm upgrade my-app bitnami/nginx --cleanup-on-fail

# If upgrade fails:
# - New resources cleaned up
# - Old resources remain active
```

---

## Preservation Options

```bash
# Keep current values (don't override with defaults)
helm upgrade my-app bitnami/nginx --preserve-values

# Useful when:
# - Adding new keys to chart values.yaml
# - Want to keep user-set values
# - Don't want defaults to override

# Don't reset pods
helm upgrade my-app bitnami/nginx --reuse-values

# Example:
# Old chart: replicas=3
# New chart: replicas=2 (new default)
# With --reuse-values: stays 3
# Without --reuse-values: becomes 2
```

---

## Upgrade Hooks

Execute scripts during upgrade:

```yaml
# Pre-upgrade (before update)
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: pre-upgrade
spec:
  template:
    spec:
      containers:
      - name: backup
        image: postgres:14
        command: ["/bin/sh", "-c", "pg_dump > backup.sql"]

# Post-upgrade (after update)
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: post-upgrade
spec:
  template:
    spec:
      containers:
      - name: migration
        image: myapp:v2.0.0
        command: ["python", "migrate.py"]
```

---

## Idempotent Install/Upgrade

Unified command (create if not exists, update if exists):

```bash
# Creates release if missing, upgrades if exists
helm upgrade --install my-app bitnami/nginx

# Useful for CI/CD pipelines
# Don't need to check if exists first

# Example pipeline:
# helm upgrade --install prod-app ./mychart \
#   -n production \
#   -f values-prod.yaml
```

---

## Upgrade with GitOps

```yaml
# Kustomization (ArgoCD integration)
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

helmCharts:
- name: nginx
  repo: https://charts.bitnami.com/bitnami
  version: 14.0.0
  values:
    replicaCount: 3
```

---

## Real-World Upgrade Workflow

```bash
# Day 1: Install version 1
helm install prod-app bitnami/nginx \
  -n production \
  -f values-prod.yaml

helm list -n production
# prod-app  1   2024-01-15 DEPLOYED  nginx-13.2.0

# Day 30: New version available
helm repo update

# Day 30: Check what changed
helm diff upgrade prod-app bitnami/nginx -f values-prod.yaml

# Day 30: Upgrade to production
helm upgrade prod-app bitnami/nginx \
  -n production \
  -f values-prod.yaml \
  --atomic \
  --wait

# Day 30: Verify upgrade
helm status prod-app -n production
helm history prod-app
```

---

## Troubleshooting Upgrades

### Issue 1: Upgrade Stuck

```bash
# Pods not replacing
kubectl describe pod <pod-name>

# Check ReadinessProbe
# Increase timeout
helm upgrade myapp chart --timeout 10m

# Or force
helm upgrade myapp chart --force
```

### Issue 2: Old Resources Not Removed

```bash
# Manual cleanup needed
kubectl delete deployment old-deployment

# Or recreate with cleanup-on-fail
helm upgrade myapp chart --cleanup-on-fail
```

### Issue 3: Can't Upgrade, Stuck in Pending

```bash
# Check resource availability
kubectl top nodes

# Check PVC binding
kubectl get pvc

# Resolve before upgrade
```

---

## Upgrade Checklist

- [ ] Save current values
  ```bash
  helm get values myapp > backup-values.yaml
  ```

- [ ] Check new version
  ```bash
  helm search repo chart --versions
  ```

- [ ] Review changes
  ```bash
  helm diff upgrade myapp chart
  ```

- [ ] Dry-run
  ```bash
  helm upgrade myapp chart --dry-run
  ```

- [ ] Backup data (if needed)
  ```bash
  kubectl exec pod -- backup.sh
  ```

- [ ] Upgrade with atomic
  ```bash
  helm upgrade myapp chart --atomic --wait
  ```

- [ ] Verify status
  ```bash
  helm status myapp
  ```

---

## Summary

Upgrading releases:

**Basic:**
- helm upgrade release chart
- --dry-run to preview
- -f values.yaml for customization

**Safety:**
- --atomic (rollback on fail)
- --wait (wait for ready)
- Test first with dry-run

**Advanced:**
- --force (delete and recreate)
- Hooks (pre/post upgrade)
- helm diff plugin (see changes)

**Next:** File 03 covers rollbacks.
