# Node Drain and Cordon

## Understanding Node States

Your cluster has node states:

```
Node: Ready, SchedulingDisabled
├─ Ready: Can receive new pods
├─ NotReady: Cannot receive new pods
├─ SchedulingDisabled: Won't receive new pods (cordoned)
└─ Draining: Moving existing pods away
```

---

## Cordon: Block New Pods

Prevent Kubernetes from scheduling NEW pods on a node:

```bash
# Cordon a node
kubectl cordon worker-1

# Check node status
kubectl get nodes

# Output:
# NAME      STATUS                     ROLES   VERSION
# worker-1  Ready,SchedulingDisabled   <none>  v1.27.0
#           ↑ SchedulingDisabled = Cordoned
```

### When to Cordon

- Node needs maintenance
- Node will be removed
- Network issues expected
- Hardware replacement needed

### Why Cordon First?

```
Without cordon:
└─ New pods schedule on node
   └─ They get evicted immediately
   └─ Wasteful

With cordon:
└─ Scheduler skips cordoned node
└─ New pods go to healthy nodes
└─ Efficient
```

---

## Drain: Evict All Pods

Move all running pods away from a node:

```bash
# Drain a node
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data

# This:
# 1. Cordons the node (stops new scheduling)
# 2. Evicts all pods (moves them to other nodes)
# 3. Keeps DaemonSets running (they're node-specific)
```

### What Gets Evicted?

```
Deployment pods        → Moved to other nodes ✓
StatefulSet pods      → Moved to other nodes ✓
Job pods              → Moved to other nodes ✓
DaemonSet pods        → Stay on node (--ignore-daemonsets) ✓
System pods           → Moved by default ✓
Pod without controller → Will be deleted! ⚠️
```

### Real Drain Example

```bash
# Drain takes time
kubectl drain worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout=10m

# Output shows what's happening:
# node/worker-1 already cordoned
# WARNING: ignoring daemonsets-managed pods: \
#   kube-proxy-abc123, calico-node-xyz789
# evicting pod: default/my-app-deployment-abc123
# evicting pod: default/my-app-deployment-def456
# evicting pod: kube-system/coredns-558bd4d5db-abc123
# pod/my-app-deployment-abc123 evicted
# pod/my-app-deployment-def456 evicted
# pod/coredns-558bd4d5db-abc123 evicted
# node/worker-1 evicted
```

---

## Drain Flags

```bash
# Basic drain
kubectl drain worker-1 --ignore-daemonsets

# Delete pods with emptyDir volumes
kubectl drain worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data

# Force evict pods that won't gracefully terminate
kubectl drain worker-1 \
  --ignore-daemonsets \
  --force

# Extend timeout (default 30s per pod)
kubectl drain worker-1 \
  --ignore-daemonsets \
  --timeout=5m

# Skip pods without controller (dangerous!)
kubectl drain worker-1 \
  --ignore-daemonsets \
  --force \
  --disable-eviction

# Grace period for termination (default 30s)
kubectl drain worker-1 \
  --ignore-daemonsets \
  --grace-period=60
```

---

## Maintenance Workflow

### Scenario: Replace Node Hardware

```bash
#!/bin/bash
# Cordon, drain, maintenance, uncordon

NODE="worker-2"

# Step 1: Cordon
echo "Cordoning $NODE..."
kubectl cordon "$NODE"

# Step 2: Wait for user to be aware
sleep 5

# Step 3: Drain
echo "Draining $NODE..."
kubectl drain "$NODE" \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout=10m

if [ $? -ne 0 ]; then
  echo "Drain failed!"
  kubectl uncordon "$NODE"
  exit 1
fi

echo "$NODE is now empty. Maintenance can proceed."

# Step 4: Perform maintenance
# (SSH to node, replace disk, update software, etc.)
# read -p "Press Enter when maintenance complete..."

# Step 5: Uncordon
echo "Uncordoning $NODE..."
kubectl uncordon "$NODE"

# Step 6: Verify
echo "Verifying $NODE..."
kubectl get node "$NODE" -o wide
```

---

## What Goes Wrong: Stuck Pods

### Issue: Pod Won't Evict

```bash
# Drain hangs
kubectl drain worker-1 --ignore-daemonsets

# Error: cannot evict pod "stubborn-pod", not all pods evicted

# Causes:
# 1. Pod has local data (emptyDir)
# 2. Pod has long termination time
# 3. Pod has readiness check that never completes
# 4. Pod disruption budget blocking eviction
```

### Solution 1: Force Eviction

```bash
# Force evict (dangerous!)
kubectl drain worker-1 \
  --ignore-daemonsets \
  --force \
  --grace-period=0

# WARNING: This can cause:
# - Data loss (if pod has local state)
# - Incomplete transactions
# - Consistency issues

# Use only if you know what you're doing
```

### Solution 2: Delete Pod Directly

```bash
# First, identify the problem pod
kubectl drain worker-1 --ignore-daemonsets
# error: cannot evict pod "db-backup-job-xyz"

# Check the pod
kubectl describe pod db-backup-job-xyz

# If pod is stuck:
kubectl delete pod db-backup-job-xyz --force --grace-period=0

# Then retry drain
kubectl drain worker-1 --ignore-daemonsets
```

### Solution 3: Fix Pod Disruption Budget

```bash
# If PDB blocks eviction
kubectl get pdb

# Edit PDB to allow disruption
kubectl edit pdb my-pdb

# Or temporarily delete PDB
kubectl delete pdb my-pdb

# Drain
kubectl drain worker-1 --ignore-daemonsets

# Recreate PDB
kubectl apply -f pdb.yaml
```

---

## Common Scenarios

### Scenario 1: Node Maintenance

```bash
# Node needs firmware update, SSL certificate replaced, etc.

# 1. Cordon
kubectl cordon node-1

# 2. Drain
kubectl drain node-1 \
  --ignore-daemonsets \
  --delete-emptydir-data

# 3. SSH to node and do work
ssh admin@node-1
sudo /opt/firmware/update.sh
sudo systemctl reboot

# 4. Wait for node to come back
sleep 30
kubectl get nodes

# 5. Uncordon
kubectl uncordon node-1

# 6. Verify pods rescheduling
kubectl get pods --all-namespaces -o wide | grep node-1
```

### Scenario 2: Node Removal

```bash
# Removing node from cluster permanently

# 1. Drain node
kubectl drain node-to-remove \
  --ignore-daemonsets \
  --delete-emptydir-data

# 2. Delete node from Kubernetes
kubectl delete node node-to-remove

# 3. Terminate EC2 instance (if on AWS)
aws ec2 terminate-instances --instance-ids i-12345678
```

### Scenario 3: Emergency Maintenance

```bash
# Node has critical issue, needs immediate attention

# 1. Immediately cordon (stop new pods)
kubectl cordon problematic-node

# 2. Check what's running
kubectl get pods --all-namespaces -o wide | grep problematic-node

# 3. Graceful drain if possible
kubectl drain problematic-node \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout=5m

# 4. If drain times out, force evict
kubectl drain problematic-node \
  --ignore-daemonsets \
  --force \
  --timeout=1m

# 5. Take node down
ssh admin@problematic-node
sudo shutdown -h now

# 6. Replace or repair
# ...

# 7. Bring back up
# ...

# 8. If coming back: uncordon
kubectl uncordon problematic-node
```

---

## Best Practices

1. **Always Cordon Before Drain**
   ```bash
   # WRONG: Just drain
   kubectl drain worker-1 --ignore-daemonsets
   
   # RIGHT: Cordon first (drain does this anyway, but be explicit)
   kubectl cordon worker-1
   kubectl drain worker-1 --ignore-daemonsets
   ```

2. **Use Appropriate Flags**
   ```bash
   # Most maintenance:
   kubectl drain node \
     --ignore-daemonsets \
     --delete-emptydir-data \
     --timeout=10m
   ```

3. **Monitor Pod Movement**
   ```bash
   # Watch pods being rescheduled
   watch kubectl get pods --all-namespaces -o wide
   
   # Check node is empty
   kubectl get pods --all-namespaces -o wide | grep node-being-drained
   # Should be empty
   ```

4. **Don't Forget to Uncordon**
   ```bash
   # After maintenance, bring node back
   kubectl uncordon node
   
   # Verify it's accepting pods
   kubectl get nodes
   # Should show "Ready" (not SchedulingDisabled)
   ```

---

## Summary

Drain and cordon:

**Cordon:**
- Prevents new pods from scheduling
- Existing pods stay running
- Use: temporary hold

**Drain:**
- Evicts all pods
- Cordons automatically
- Use: preparation for maintenance

**Process:**
- Cordon (optional, drain does it)
- Drain --ignore-daemonsets --delete-emptydir-data
- Perform maintenance
- Uncordon

**Flags:**
- --ignore-daemonsets (DaemonSets stay)
- --delete-emptydir-data (recreate pods with emptyDir)
- --force (dangerous, last resort)
- --timeout=10m (wait time for graceful termination)

**Next:** File 02 covers Pod Disruption Budgets.
