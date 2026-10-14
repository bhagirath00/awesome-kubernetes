# Kubernetes Version Updates

## Why Upgrade?

Imagine you've been running Kubernetes 1.20 for a year. Your cluster is stable, everything works. But now you're hearing about new features in 1.26, security patches, performance improvements, and deprecations in your current version. You face a choice: upgrade or risk falling behind.

Cluster upgrades are necessary but can be risky if not done properly.

---

## Version Strategy

```
Kubernetes versioning: MAJOR.MINOR.PATCH

Example: v1.27.3
         ↓  ↓  ↓
         Major.Minor.Patch

- MAJOR: Rare (v1 has been stable for years)
- MINOR: Quarterly releases (v1.25, v1.26, v1.27, etc.)
- PATCH: Bug fixes and security patches
```

---

## Upgrade Timeline

```
v1.25 (released April 2022)
  ├─ v1.25.0, v1.25.1, ..., v1.25.12 (patch releases)
  └─ Supported for ~9 months

v1.26 (released May 2023)
  ├─ v1.26.0, v1.26.1, ..., v1.26.10 (patch releases)
  └─ Kubernetes "release train"

v1.27 (released August 2023)
  ├─ Latest versions get 1 year support
  └─ Older versions get security patches only
```

You should upgrade within ~1 year to stay supported.

---

## Support Window

```
v1.24: ❌ UNSUPPORTED (April 2022 - Feb 2023)
v1.25: ⚠️  SECURITY ONLY (April 2022 - Aug 2023)
v1.26: ✅ FULLY SUPPORTED (May 2023 - Jan 2024)
v1.27: ✅ CURRENT (Aug 2023 - Aug 2024)

Always run at least v1.24 or later to get patches
```

---

## What I See Teams Get Wrong

**WRONG — Skip Minor Versions**
```bash
# Dangerous: jumping multiple versions
# v1.24 → v1.27 (skipped 1.25, 1.26)
helm upgrade cluster-k8s v1.27.0

# Problem:
# - Deprecations not handled gradually
# - Breaking changes accumulate
# - Harder to debug failures
```

**RIGHT — Upgrade Incrementally**
```bash
# Better: one or two minor versions at a time
v1.24 → v1.25 → v1.26 → v1.27

# Benefits:
# - Catch issues early
# - Easier rollback
# - Gradual API deprecation
```

---

## Upgrade Path for Your Cluster

### Step 1: Know Your Current Version

```bash
# Check cluster version
kubectl version --short

# Output:
# Client Version: v1.27.0
# Server Version: v1.26.3

# Check node versions
kubectl get nodes -o wide

# Output:
# NAME              STATUS   ROLES   VERSION
# master-1          Ready    master  v1.26.3
# worker-1          Ready    <none>  v1.26.3
# worker-2          Ready    <none>  v1.26.3
```

### Step 2: Plan Upgrade Target

```bash
# Current: v1.26.3
# Target: v1.27.2

# Decision: upgrade within same minor version first (safe)
v1.26.3 → v1.26.10 (patch) → v1.27.2 (minor)
```

### Step 3: Check Compatibility

```bash
# What changed in v1.27?
# - Removed beta APIs
# - New features in alpha/beta
# - Changed default behaviors

# Check deprecations
kubectl api-resources

# Verify your resources are not deprecated
kubectl explain deployment
```

---

## Upgrade Strategies

### Strategy 1: Blue-Green (Low Downtime)

```
Blue Cluster (v1.26)  ←→  Load Balancer  ←→  Green Cluster (v1.27)

1. Create identical v1.27 cluster
2. Migrate workloads to v1.27
3. Test everything
4. Switch traffic
5. Keep v1.26 as fallback
```

**Pros:** Minimal downtime, easy rollback
**Cons:** Requires duplicate infrastructure, expensive

### Strategy 2: In-Place (Traditional)

```
Cluster v1.26
  ├─ Upgrade control plane → v1.27
  ├─ Drain nodes one by one
  ├─ Upgrade worker nodes → v1.27
  └─ Done (but can't easily rollback)
```

**Pros:** Single cluster, familiar
**Cons:** Downtime possible, rollback harder

### Strategy 3: Rolling (Managed Services)

```
EKS automatically manages:
1. Creates new node group (v1.27)
2. Gradually migrates pods
3. Removes old nodes (v1.26)
4. Zero downtime
```

**Pros:** Automatic, zero downtime
**Cons:** Less control, depends on managed service

---

## Real Cluster Upgrade Workflow

```
Week 1: Planning
├─ Check current version
├─ Research what's new
├─ Check for breaking changes
└─ Notify team

Week 2: Testing
├─ Test upgrade on staging cluster
├─ Verify applications work
├─ Check monitoring, logging, metrics
└─ Document any issues

Week 3: Backup
├─ Backup etcd database
├─ Backup application data (if needed)
├─ Test restore procedure
└─ Verify backups

Week 4: Upgrade
├─ Maintenance window scheduled
├─ Execute upgrade plan
├─ Immediate health checks
├─ Rollback plan ready if issues
└─ Monitor closely for 24 hours

Week 5: Stabilization
├─ Monitor for new issues
├─ Check all workloads
├─ Verify performance
└─ Post-upgrade cleanup
```

---

## Common Issues During Upgrade

### Issue 1: API Version Not Found

```bash
# Error during upgrade:
# "error: unknown API version batch/v1beta1"

# Problem: API removed in new version
# Solution: Update manifests before upgrade

# Check what APIs are deprecated
kubectl api-resources

# Fix manifests
sed -i 's/batch\/v1beta1/batch\/v1/g' cronjob.yaml
```

### Issue 2: Pod Stuck Pending

```bash
# After upgrade, pods not starting
kubectl describe pod stuck-pod

# Common causes:
# 1. Image pull failed (check image registry)
# 2. Resource limits too high
# 3. Node taint mismatch

# Fix:
kubectl delete pod stuck-pod  # Let controller recreate
```

### Issue 3: Control Plane Won't Start

```bash
# Master node stuck after upgrade
systemctl status kubelet

# Check logs
journalctl -u kubelet -n 100

# Common fix:
systemctl restart kubelet
systemctl restart docker
```

---

## Downgrade (If Needed)

```bash
# WARNING: Not officially supported!
# If upgrade goes wrong:

1. Isolate cluster from production traffic
2. Restore from etcd backup
3. Revert to previous version
4. Extensive testing before re-enabling

# This is why backups matter!
```

---

## Summary

Kubernetes upgrades:

**Versioning:**
- Quarterly minor releases (v1.25, v1.26, v1.27)
- Support window ~1 year
- Always stay within supported versions

**Strategies:**
- Blue-green: low risk, expensive
- In-place: traditional, single cluster
- Rolling: managed services, automatic

**Process:**
- Plan and test first
- Backup everything
- Execute during maintenance
- Monitor closely

**Best Practice:**
- Upgrade within 1 year
- Upgrade incrementally (skip ≤2 minor versions)
- Always have rollback plan

**Next:** File 02 covers control plane upgrades.
