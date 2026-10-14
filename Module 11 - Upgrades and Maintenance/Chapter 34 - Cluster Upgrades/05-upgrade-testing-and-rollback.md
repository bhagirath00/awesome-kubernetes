# Upgrade Testing and Rollback

## Why Test Before Upgrading

Imagine you upgrade production at 3pm without testing. Everything breaks. You spend 4 hours debugging. Your customers can't access the service. You lose money. Someone gets blamed.

Testing prevents this.

---

## Pre-Upgrade Testing Checklist

```
□ Backup etcd
□ Test backup restore
□ Review upgrade notes
□ Check API deprecations
□ Test on staging cluster
□ Run smoke tests
□ Check third-party compatibility
□ Verify monitoring works
□ Check logging still works
□ Notify team
□ Have rollback plan ready
```

---

## Testing on Staging Cluster

### Step 1: Create Staging Cluster at New Version

```bash
# Create v1.27 cluster for testing
eksctl create cluster \
  --name staging-test \
  --version 1.27 \
  --region us-east-1 \
  --nodegroup-name staging-nodes

# Or upgrade existing staging cluster to target version
aws eks update-cluster-version \
  --name staging-cluster \
  --kubernetes-version 1.27
```

### Step 2: Deploy Test Workloads

```bash
# Deploy your applications to staging
kubectl apply -f ./deployments/

# Deploy test suite
kubectl apply -f ./tests/

# Example test pod
apiVersion: v1
kind: Pod
metadata:
  name: upgrade-test
spec:
  containers:
  - name: test
    image: my-app:latest
    command: ["/bin/sh", "-c"]
    args:
      - |
        set -e
        echo "Testing application..."
        
        # Health check
        curl -f http://localhost:8080/health
        
        # Database connectivity
        psql -h postgres -d myapp -c "SELECT 1"
        
        # API functionality
        curl -f http://localhost:8080/api/users
        
        echo "All tests passed!"
```

### Step 3: Run Smoke Tests

```bash
#!/bin/bash
# smoke-tests.sh

echo "Running smoke tests..."

# 1. Check cluster health
kubectl get nodes
kubectl get pods --all-namespaces

# 2. Test deployments
kubectl rollout status deployment/my-app -n default --timeout=5m
kubectl rollout status deployment/database -n default --timeout=5m

# 3. Check service connectivity
INGRESS_IP=$(kubectl get ingress -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')

# 4. Test endpoints
curl -f http://$INGRESS_IP/health
curl -f http://$INGRESS_IP/api/v1/users
curl -f http://$INGRESS_IP/api/v1/products

# 5. Check logs
kubectl logs -n default deployment/my-app --tail=10

# 6. Check metrics
kubectl top nodes
kubectl top pods -n default

echo "Smoke tests passed!"
```

### Step 4: Test API Compatibility

```bash
# Check if your API versions still exist
kubectl api-versions | grep -i batch
# Should show: batch/v1

# Try getting resources with old API version
kubectl get cronjobs --api-version=batch/v1

# If it fails:
# - API version was removed
# - Need to update manifests before production upgrade
```

---

## Testing Deprecated APIs

```bash
# Find deprecated APIs in your cluster
kubectl api-resources --sort-by=name

# Check if any of your manifests use old versions
grep -r "apiVersion" ./k8s-manifests/ | grep -v "apps/v1"

# Example: Find batch/v1beta1 (deprecated)
grep "batch/v1beta1" ./k8s-manifests/*.yaml

# Update before upgrade
sed -i 's/batch\/v1beta1/batch\/v1/g' cronjob.yaml
```

---

## Rollback Strategy

### Scenario: Upgrade Goes Wrong

```
10:00 - Upgrade starts
10:30 - Control plane upgraded to v1.27
11:00 - Nodes being upgraded to v1.27
11:15 - Critical error detected!
       - Pods not starting
       - Database connection failing
       - API returning errors
11:20 - Decision: Rollback!
```

---

## Downgrade Process

### On EKS (Managed)

```bash
# Check if downgrade is possible
# AWS allows downgrade for short window (24 hours)

# Downgrade control plane
aws eks update-cluster-version \
  --name my-cluster \
  --kubernetes-version 1.26

# Downgrade node groups
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name standard-nodes \
  --kubernetes-version 1.26

# Verify
aws eks describe-cluster --name my-cluster --query 'cluster.version'
```

### On Self-Managed Cluster (Complex)

```bash
# WARNING: Not officially supported by Kubernetes!
# Only attempted if critical issue

# Step 1: Restore etcd from backup (Chapter 36 covers this)
sudo /opt/kubernetes/restore-etcd.sh /backups/etcd-pre-upgrade.db

# Step 2: Revert kubeadm
sudo apt-get install kubeadm=1.26.0-00

# Step 3: Downgrade control plane
sudo kubeadm upgrade apply v1.26.0 --yes

# Step 4: Revert kubelet on nodes
sudo apt-get install kubelet=1.26.0-00
sudo systemctl restart kubelet

# Step 5: Verify
kubectl version --short
```

---

## Real-World Rollback Scenario

```bash
#!/bin/bash
# emergency-rollback.sh

CLUSTER_NAME="production"
TARGET_VERSION="1.26"

echo "⚠️  EMERGENCY ROLLBACK INITIATED"
echo "Rolling back $CLUSTER_NAME to v$TARGET_VERSION"

# Step 1: Notify team
echo "Sending Slack notification..."
curl -X POST -H 'Content-type: application/json' \
  --data '{"text":"🚨 Production rollback in progress"}' \
  $SLACK_WEBHOOK_URL

# Step 2: Stop accepting new traffic
echo "Stopping new traffic..."
# kubectl label nodes --all maintain-traffic=no

# Step 3: Downgrade control plane
echo "Downgrading control plane..."
aws eks update-cluster-version \
  --name $CLUSTER_NAME \
  --kubernetes-version $TARGET_VERSION

# Wait for completion
while true; do
  STATUS=$(aws eks describe-cluster \
    --name $CLUSTER_NAME \
    --query 'cluster.status' \
    --output text)
  
  if [ "$STATUS" = "ACTIVE" ]; then
    break
  fi
  sleep 30
done

# Step 4: Downgrade node groups
echo "Downgrading node groups..."
NODE_GROUPS=$(aws eks list-nodegroups \
  --cluster-name $CLUSTER_NAME \
  --query 'nodegroups[*]' \
  --output text)

for ng in $NODE_GROUPS; do
  aws eks update-nodegroup-version \
    --cluster-name $CLUSTER_NAME \
    --nodegroup-name $ng \
    --kubernetes-version $TARGET_VERSION
  
  # Wait between node groups
  sleep 300
done

# Step 5: Verify rollback
echo "Verifying rollback..."
CURRENT=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query 'cluster.version' \
  --output text)

if [ "$CURRENT" = "$TARGET_VERSION" ]; then
  echo "✅ Rollback successful!"
  
  # Resume traffic
  # kubectl label nodes --all maintain-traffic=yes
  
  echo "🎉 Cluster restored to v$TARGET_VERSION"
else
  echo "❌ Rollback failed! Manual intervention needed."
fi
```

---

## What Can Go Wrong

### Issue 1: Can't Downgrade (Window Closed)

```bash
# Error: Cannot downgrade beyond version v1.26 on this cluster

# AWS only allows downgrade for 24 hours after upgrade
# After that, you're stuck at new version

# Prevention:
# - Don't upgrade if you can't afford issues
# - Test thoroughly first
# - Have very good backups
```

### Issue 2: etcd Restored But Data Inconsistent

```bash
# After restore, some objects are missing
# Pods, services, deployments gone

# Cause: etcd backup is snapshot in time
# Any changes after backup are lost

# Prevention:
# - Backup immediately before upgrade (not hours before)
# - Minimize time between backup and upgrade
# - Have application-level backups
```

### Issue 3: Pods Won't Start After Downgrade

```bash
# After rolling back to v1.26, pods fail
# Error: "invalid apiVersion"

# Cause: New pod manifest created with v1.27 API
# v1.26 doesn't understand it

# Solution:
# - Update manifests back to compatible versions
# - Or keep pods at compatible state
```

---

## Prevention is Better Than Rollback

Focus on testing, not hoping rollback works:

```
Invest in testing:
├─ Automated smoke tests
├─ API compatibility checks
├─ Staging cluster testing
└─ Pre-upgrade verification

Result:
└─ Catch issues before production
```

---

## Rollback Decision Tree

```
Upgrade failed? (production broken?)
  ├─ NO → Keep new version, debug and fix
  │
  └─ YES → Is it critical?
      ├─ NO → Wait 24 hours, investigate, then decide
      │
      └─ YES → Immediate rollback
          ├─ Can downgrade? (< 24 hours)
          │   ├─ YES → Execute rollback immediately
          │   └─ NO → Restore from backup
          │
          └─ Communicate to team
```

---

## Best Practices

1. **Test Before Production**
   - Staging cluster at new version
   - Run smoke tests
   - Check API compatibility

2. **Backup Everything**
   - etcd backup
   - Application data
   - Configuration

3. **Have Maintenance Window**
   - Scheduled, communicated to team
   - Non-business-critical time
   - Full team on standby

4. **Monitor Continuously**
   - Before upgrade
   - During upgrade
   - 24 hours after upgrade

5. **Document Rollback Plan**
   - Step-by-step procedure
   - Contact list
   - Escalation path

---

## Summary

Upgrade testing:

**Pre-Upgrade:**
- Test on staging first
- Run smoke tests
- Check API compatibility
- Backup everything

**Rollback:**
- EKS: Can downgrade within 24 hours
- Self-managed: Very difficult, requires etcd restore
- Prevention better than rollback

**Best Practice:**
- Invest in testing
- Have solid backups
- Never upgrade without safety net

**Module 11 Chapter 34 Complete - 5 Files!**

You now understand cluster upgrades:
- File 01: K8s versions and strategy
- File 02: Control plane upgrades
- File 03: Worker node upgrades
- File 04: EKS-specific upgrades
- File 05: Testing and rollback

**Next:** Chapter 35 covers node maintenance.
