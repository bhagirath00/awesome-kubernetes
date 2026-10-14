# Rollback with Helm

## What is Rollback?

Rollback reverts release to previous version:

```
Release v1 (current, broken)
        ↓ (problem)
Rollback → Release v2 (working, previous)

Or

Release v1 ← Release v2 ← Release v3 (current, broken)
        ↑
Rollback to any previous version
```

---

## View Release History

```bash
# List all revisions
helm history my-app

# Output:
# REVISION  UPDATED                       STATUS     CHART          APP VERSION
# 1         Mon Jan 15 10:00:00 2024      superseded nginx-13.2.0   1.20.0
# 2         Mon Jan 15 14:00:00 2024      superseded nginx-13.3.0   1.21.0
# 3         Mon Jan 15 18:00:00 2024      deployed   nginx-14.0.0   1.22.0

# With namespace
helm history my-app -n production
```

---

## Basic Rollback

```bash
# Rollback to previous revision
helm rollback my-app

# Rollback to specific revision
helm rollback my-app 1

# Rollback in namespace
helm rollback my-app 2 -n production

# Test rollback (dry-run)
helm rollback my-app --dry-run --debug
```

---

## Real-World Rollback Scenarios

### Scenario 1: Bad Deployment

```bash
# Deployed problematic version
helm upgrade my-app bitnami/nginx --version 14.1.0

# App broken
# Immediate rollback
helm rollback my-app

# Release goes back to 13.3.0 (previous working version)
```

### Scenario 2: Configuration Error

```bash
# Upgraded with wrong database credentials
helm upgrade my-app ./mychart \
  --set database.password=wrong-password

# Connection failures
# Rollback
helm rollback my-app

# Back to previous config (correct password)
```

### Scenario 3: Multiple Revisions Back

```bash
# History:
# Revision 1: v1.0.0 (original)
# Revision 2: v1.1.0 (minor update)
# Revision 3: v1.2.0 (new feature)
# Revision 4: v1.3.0 (current, broken)

# Rollback to v1.1.0 (revision 2)
helm rollback my-app 2
```

---

## Rollback with Cleanup

```bash
# Rollback and clean up failed resources
helm rollback my-app --cleanup-on-fail

# If rollback fails:
# - Failed release cleaned up
# - Previous release restored
```

---

## Rollback Timeline

```
Time 1:00  Version v1.0 deployed (Revision 1)
          Status: DEPLOYED

Time 2:00  Upgrade to v1.1 (Revision 2)
          Status: DEPLOYED

Time 3:00  Upgrade to v1.2 (Revision 3)
          Status: DEPLOYED

Time 3:30  Rollback to Revision 2
          Status: DEPLOYED
          (Revision 3 becomes SUPERSEDED)

Time 4:00  View history shows:
          Rev 1: SUPERSEDED
          Rev 2: DEPLOYED (current)
          Rev 3: SUPERSEDED
```

---

## Helm History Analysis

```bash
# Check what changed between revisions
helm get values my-app --revision 2
helm get values my-app --revision 3

# Compare configs
diff <(helm get values my-app --revision 2) \
     <(helm get values my-app --revision 3)

# Check manifest
helm get manifest my-app --revision 2
helm get manifest my-app --revision 3
```

---

## Automated Rollback

### With Monitoring

```bash
#!/bin/bash
# Automatic rollback if health check fails

RELEASE="my-app"
NAMESPACE="production"
HEALTH_CHECK_URL="http://my-app:8080/health"

# Deploy new version
helm upgrade $RELEASE bitnami/nginx --version 14.1.0

# Wait for deployment
kubectl rollout status deployment/$RELEASE -n $NAMESPACE

# Health check
if ! curl -f $HEALTH_CHECK_URL; then
  echo "Health check failed, rolling back..."
  helm rollback $RELEASE -n $NAMESPACE
  exit 1
fi

echo "Health check passed, upgrade successful"
```

### With Kubernetes Liveness Probe

```yaml
# Pod with liveness probe
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: app
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        # If probe fails 3x, pod is restarted
        # If too many restarts, consider rollback
```

---

## Prevention: Pre-Upgrade Testing

```bash
# Test chart before upgrade
helm install my-app-test ./mychart --namespace test --create-namespace

# Run tests
kubectl run test-job \
  --image=test-runner \
  --command -- pytest /tests/

# If successful, upgrade production
helm upgrade my-app ./mychart -n production

# If tests fail, don't upgrade
```

---

## Rollback Checklist

```bash
# 1. Check current status
helm status my-app

# 2. View history
helm history my-app

# 3. Identify problematic revision
# 4. Check what changed
helm get values my-app --revision 2
helm get values my-app --revision 3

# 5. Dry-run rollback
helm rollback my-app 2 --dry-run

# 6. Execute rollback
helm rollback my-app 2

# 7. Verify status
helm status my-app

# 8. Check application
curl http://my-app/health
kubectl logs deployment/my-app
```

---

## Best Practices

1. **Keep Release History**
   - Don't delete old revisions
   - Enables quick rollback

2. **Test Before Production**
   - Upgrade staging first
   - Verify working
   - Then production

3. **Health Checks**
   - Always have readiness/liveness probes
   - Detect failures early

4. **Document Changes**
   - Know what changed between versions
   - Easier root cause analysis

5. **Automate When Possible**
   - Automatic health checks
   - Alert on failures
   - Automate rollback if needed

---

## Troubleshooting Rollback

### Issue 1: Rollback Stuck

```bash
# Pods not terminating
kubectl delete pod <stuck-pod> -n production --force

# Check events
kubectl describe deployment my-app -n production

# Manual fix if needed
```

### Issue 2: Data Loss After Rollback

```bash
# Ensure persistent volumes preserved
helm rollback my-app --cleanup-on-fail=false

# Data on PVC retained even after rollback
```

---

## Summary

Rollback:

**View History:**
- helm history release
- See all revisions

**Rollback:**
- helm rollback release (to previous)
- helm rollback release N (to specific revision)

**Safety:**
- --dry-run to preview
- --cleanup-on-fail (clean up failures)

**Best Practice:**
- Test before production
- Health checks essential
- Document changes

**Module 10 Chapter 33 Complete - 5 Files!**

You now understand Helm in practice:
- File 01: Installing charts
- File 02: Upgrading releases
- File 03: Rollback with Helm
- File 04: Creating custom charts (next)
- File 05: Helm repositories (next)

**Next:** File 04 covers creating custom charts.
