# Cluster Upgrade on AWS EKS

## EKS Architecture

AWS manages the control plane for you:

```
┌────────────────────────────────┐
│  AWS Managed Control Plane     │
│  (etcd, API server, scheduler) │
│  AWS handles upgrades!         │
└────────────────────────────────┘
         ↓
┌────────────────────────────────┐
│   Your Worker Nodes (EC2)      │
│   (you manage upgrades)        │
├────────────────────────────────┤
│ worker-1  worker-2  worker-3   │
└────────────────────────────────┘
```

---

## EKS Cluster Versions

```bash
# Check current cluster version
aws eks describe-cluster \
  --name my-cluster \
  --query 'cluster.version'

# Output: "1.26"

# List all available versions
aws eks describe-addon-versions --query 'addons[*].{Name:addonName,Versions:addonVersions[0].addonVersion}'

# Or check platform versions
aws eks describe-cluster \
  --name my-cluster \
  --query 'cluster.platformVersion'

# Output: "eks.1" (AWS patch version)
```

---

## EKS Cluster Upgrade Process

### Step 1: Plan Upgrade

```bash
# Current: v1.26
# Check what's compatible for upgrade
aws eks describe-cluster --name my-cluster

# Can upgrade directly from v1.26 → v1.27
# (EKS supports up to 2 minor versions back)
```

### Step 2: Initiate Cluster Upgrade

```bash
# This upgrades the control plane only
aws eks update-cluster-version \
  --name my-cluster \
  --kubernetes-version 1.27

# Output:
# {
#     "update": {
#         "id": "abc123-update-id",
#         "status": "InProgress",
#         "type": "VersionUpdate",
#         "params": [...],
#         "createdAt": 1234567890,
#         "errors": []
#     }
# }
```

### Step 3: Monitor Control Plane Upgrade

```bash
# Check upgrade status (takes 20-30 minutes typically)
aws eks describe-cluster --name my-cluster --query 'cluster.status'

# Status progression:
# ACTIVE (running v1.26)
#   ↓
# UPDATING (upgrading to v1.27)
#   ↓
# ACTIVE (running v1.27)

# Full details
aws eks describe-cluster --name my-cluster

# Check version updated
aws eks describe-cluster --name my-cluster --query 'cluster.version'
# Output: "1.27"
```

### Step 4: Upgrade Node Groups

After control plane is upgraded, upgrade worker nodes:

```bash
# List node groups
aws eks list-nodegroups --cluster-name my-cluster

# Output:
# {
#     "nodegroups": [
#         "standard-nodes",
#         "spot-nodes"
#     ]
# }

# Update node group (one at a time)
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name standard-nodes \
  --kubernetes-version 1.27

# Monitor node group upgrade
aws eks describe-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name standard-nodes \
  --query 'nodegroup.status'

# Status progression:
# ACTIVE (v1.26)
#   ↓
# UPDATING (upgrading to v1.27)
#   ↓
# ACTIVE (v1.27)

# Check pods are rescheduling
kubectl get nodes -o wide
# Shows new nodes being added, old ones drained
```

---

## Real-World EKS Upgrade Example

```bash
#!/bin/bash
# upgrade-eks-cluster.sh

CLUSTER_NAME="production"
NEW_VERSION="1.27"

echo "Starting EKS cluster upgrade..."

# Step 1: Check current version
CURRENT_VERSION=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --query 'cluster.version' \
  --output text)

echo "Current version: $CURRENT_VERSION"
echo "Target version: $NEW_VERSION"

# Step 2: Upgrade control plane
echo "Upgrading control plane..."
aws eks update-cluster-version \
  --name $CLUSTER_NAME \
  --kubernetes-version $NEW_VERSION

# Step 3: Wait for control plane upgrade
echo "Waiting for control plane upgrade..."
while true; do
  STATUS=$(aws eks describe-cluster \
    --name $CLUSTER_NAME \
    --query 'cluster.status' \
    --output text)
  
  if [ "$STATUS" = "ACTIVE" ]; then
    CURRENT=$(aws eks describe-cluster \
      --name $CLUSTER_NAME \
      --query 'cluster.version' \
      --output text)
    
    if [ "$CURRENT" = "$NEW_VERSION" ]; then
      echo "Control plane upgrade complete!"
      break
    fi
  fi
  
  echo "Status: $STATUS - Waiting..."
  sleep 30
done

# Step 4: Upgrade node groups
echo "Upgrading node groups..."
NODE_GROUPS=$(aws eks list-nodegroups \
  --cluster-name $CLUSTER_NAME \
  --query 'nodegroups[*]' \
  --output text)

for nodegroup in $NODE_GROUPS; do
  echo "Upgrading nodegroup: $nodegroup"
  
  aws eks update-nodegroup-version \
    --cluster-name $CLUSTER_NAME \
    --nodegroup-name $nodegroup \
    --kubernetes-version $NEW_VERSION
  
  # Wait for node group upgrade
  while true; do
    STATUS=$(aws eks describe-nodegroup \
      --cluster-name $CLUSTER_NAME \
      --nodegroup-name $nodegroup \
      --query 'nodegroup.status' \
      --output text)
    
    if [ "$STATUS" = "ACTIVE" ]; then
      echo "Nodegroup $nodegroup upgrade complete!"
      break
    fi
    
    echo "Nodegroup status: $STATUS - Waiting..."
    sleep 30
  done
done

echo "All upgrades complete!"
kubectl get nodes -o wide
```

---

## Managing EKS Add-ons

EKS add-ons (like VPC CNI, CoreDNS) also need upgrades:

```bash
# Check add-on versions
aws eks describe-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni

# List all add-ons
aws eks list-addons --cluster-name my-cluster

# Upgrade add-on
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --addon-version v1.13.0-eksbuild.1

# Available versions
aws eks describe-addon-versions \
  --addon-name vpc-cni \
  --query 'addons[0].addonVersions[*].addonVersion'
```

---

## EKS Upgrade Timeline

```
Monday 2pm:
├─ 2:00 - Initiate control plane upgrade
├─ 2:30 - Control plane still UPDATING
├─ 2:45 - Control plane still UPDATING
├─ 3:00 - Control plane upgrade complete ✓

Monday 3:30pm:
├─ 3:30 - Upgrade node group 1
├─ 4:00 - Node group 1 draining old nodes
├─ 4:30 - Node group 1 complete ✓

Monday 4:45pm:
├─ 4:45 - Upgrade node group 2
├─ 5:15 - Node group 2 draining old nodes
├─ 5:45 - Node group 2 complete ✓

Monday 6:00pm: ✅ All upgrades complete!
```

---

## Common EKS Upgrade Issues

### Issue 1: Add-on Incompatible

```bash
# Error: addon version not compatible with cluster version
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --addon-version v1.13.0

# Error: The specified add-on version is not compatible with your cluster version

# Solution: Use compatible version
aws eks describe-addon-versions --addon-name vpc-cni
# Pick version marked as compatible
```

### Issue 2: Pod IP IPAM Issues

```bash
# After upgrade, pods can't get IP addresses
# Symptom: pods stuck in Pending with "UnexpectedAdmissionError"

# Common cause: VPC CNI not upgraded yet
# Solution:
aws eks update-addon --cluster-name my-cluster --addon-name vpc-cni --addon-version v1.13.0

# Wait for VPC CNI to update
kubectl rollout status daemonset/aws-node -n kube-system
```

### Issue 3: Launch Template Issue

```bash
# Error: Node group update fails
# Error: "You cannot use the same launch template version for multiple node groups"

# Solution: Node group uses specific launch template version
# Ensure different node groups use compatible templates
aws eks describe-nodegroup --cluster-name my-cluster --nodegroup-name ng1
```

---

## Best Practices for EKS Upgrades

1. **Upgrade Control Plane First**
   - Then node groups
   - Never node groups before control plane

2. **Upgrade Add-ons**
   - After control plane but before node groups
   - Ensures networking/storage work

3. **Monitor Closely**
   - Check cluster status
   - Watch node group status
   - Monitor pod scheduling

4. **Test on Non-Prod First**
   - Dev cluster upgrade first
   - Staging second
   - Prod last

5. **Backup Before Upgrade**
   - Backup applications
   - Backup configurations
   - Backup etcd (AWS manages but good to know)

---

## Summary

EKS upgrades:

**Process:**
- Upgrade control plane (AWS managed)
- Upgrade add-ons (VPC CNI, CoreDNS, etc.)
- Upgrade node groups (EC2 instances)

**Commands:**
- aws eks update-cluster-version
- aws eks update-addon
- aws eks update-nodegroup-version

**Monitoring:**
- Use aws eks describe-cluster
- Check cluster.status
- Monitor node group status

**Benefits:**
- AWS handles control plane complexity
- Zero-downtime upgrades possible
- Automatic node draining during updates

**Next:** File 05 covers testing and rollback.
