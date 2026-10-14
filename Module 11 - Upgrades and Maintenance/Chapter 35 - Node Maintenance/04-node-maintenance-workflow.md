# Node Maintenance Workflow

## Complete Maintenance Process

You have a production cluster. A node needs maintenance (firmware update, disk replacement, SSL cert install). Here's the safe way to do it:

---

## Pre-Maintenance Checks

```bash
#!/bin/bash
# pre-maintenance-check.sh

NODE="worker-3"

echo "========== PRE-MAINTENANCE CHECKS =========="

# 1. Verify node exists and is healthy
echo "Checking node status..."
kubectl get node "$NODE"
if [ $? -ne 0 ]; then
  echo "❌ Node not found"
  exit 1
fi

# 2. Check what's running on node
echo ""
echo "Pods running on $NODE:"
kubectl get pods --all-namespaces -o wide | grep "$NODE" | wc -l
echo "Pods listed:"
kubectl get pods --all-namespaces -o wide | grep "$NODE"

# 3. Check for critical workloads
echo ""
echo "Critical workloads (databases, etc):"
kubectl get pods --all-namespaces -l tier=critical -o wide | grep "$NODE"

# 4. Check PDBs
echo ""
echo "Pod Disruption Budgets:"
kubectl get pdb --all-namespaces

# 5. Verify backup exists
echo ""
echo "Recent backups:"
ls -lh /backups/etcd* 2>/dev/null | tail -5

echo ""
echo "✅ Pre-checks complete. Safe to proceed."
```

---

## Step-by-Step Maintenance

### Step 1: Notify Team

```bash
# Send message to team
MAINTENANCE_WINDOW="Monday 2024-03-18 02:00 UTC"

# Slack notification
curl -X POST -H 'Content-type: application/json' \
  --data "{\"text\":\"🔧 Maintenance window: $MAINTENANCE_WINDOW. Node worker-3 will be unavailable.\"}" \
  $SLACK_WEBHOOK

# Email team
echo "Maintenance of worker-3 scheduled for $MAINTENANCE_WINDOW" | mail -s "Scheduled maintenance" team@company.com
```

### Step 2: Cordon Node

```bash
NODE="worker-3"

echo "Step 1: Cordoning node..."
kubectl cordon "$NODE"

# Verify
kubectl get node "$NODE"
# Should show: SchedulingDisabled
```

### Step 3: Check Application Status

```bash
echo "Step 2: Checking application health..."

# Before drain, ensure apps are healthy elsewhere
kubectl get deployments --all-namespaces -o wide

# Check replica distribution
for ns in default kube-system; do
  echo "Namespace: $ns"
  kubectl get pods -n "$ns" -o wide | grep "$NODE"
done
```

### Step 4: Drain Node

```bash
echo "Step 3: Draining node (this may take a few minutes)..."

kubectl drain "$NODE" \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout=15m

if [ $? -eq 0 ]; then
  echo "✅ Drain successful"
else
  echo "❌ Drain failed, check errors above"
  echo "Attempting to uncordon..."
  kubectl uncordon "$NODE"
  exit 1
fi

# Verify node is empty
echo ""
echo "Verifying node is empty..."
REMAINING=$(kubectl get pods --all-namespaces -o wide | grep -c "$NODE")
echo "Pods remaining on $NODE: $REMAINING"
if [ "$REMAINING" -gt 0 ]; then
  echo "⚠️  Warning: Pods still on node"
fi
```

### Step 5: Perform Maintenance

```bash
echo "Step 4: Performing maintenance..."
echo "SSH to node and run maintenance commands"

# SSH to node
ssh ubuntu@"$NODE"

# Example maintenance tasks
sudo systemctl stop kubelet          # Stop kubelet (optional)
sudo apt-get update
sudo apt-get upgrade -y              # Update packages
sudo /opt/firmware/update.sh         # Update firmware
sudo /opt/ssl/install-cert.sh        # Install SSL cert
sudo reboot                          # Reboot node
```

### Step 6: Verify Node Recovery

```bash
echo "Step 5: Verifying node recovery after reboot..."

# Wait for node to come back
sleep 30

# Check if node is back online
for i in {1..30}; do
  STATUS=$(kubectl get node "$NODE" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')
  if [ "$STATUS" = "True" ]; then
    echo "✅ Node back online"
    break
  fi
  echo "Waiting for node to be Ready... ($i/30)"
  sleep 10
done

# Check node status
kubectl get node "$NODE" -o wide

# Check kubelet is healthy
kubectl describe node "$NODE"
```

### Step 7: Uncordon Node

```bash
echo "Step 6: Uncordoning node..."

kubectl uncordon "$NODE"

# Verify
kubectl get node "$NODE"
# Should show: Ready (no SchedulingDisabled)
```

### Step 8: Wait for Pods to Reschedule

```bash
echo "Step 7: Waiting for pods to reschedule..."

# Monitor pods coming back
watch kubectl get pods --all-namespaces -o wide | grep "$NODE"

# Or
for i in {1..60}; do
  COUNT=$(kubectl get pods --all-namespaces -o wide | grep -c "$NODE")
  echo "Pods on $NODE: $COUNT"
  
  if [ "$COUNT" -gt 0 ]; then
    # Pods are scheduling back, wait a bit more
    sleep 5
  else
    echo "Waiting for pods to schedule..."
    sleep 10
  fi
done
```

### Step 9: Verify Application Health

```bash
echo "Step 8: Verifying application health..."

# Check all pods are Running
kubectl get pods --all-namespaces -o wide | grep -v Running | grep -v Completed

# Check deployments are healthy
kubectl get deployments --all-namespaces

# Check services have endpoints
kubectl get services --all-namespaces

# Health checks
for deployment in $(kubectl get deployments -o jsonpath='{.items[*].metadata.name}'); do
  kubectl rollout status "deployment/$deployment" --timeout=5m
done

echo "✅ All applications healthy"
```

### Step 10: Notify Completion

```bash
echo "Step 9: Notifying team of completion..."

# Slack notification
curl -X POST -H 'Content-type: application/json' \
  --data '{"text":"✅ Maintenance of worker-3 complete. All systems healthy."}' \
  $SLACK_WEBHOOK

# Email
echo "Maintenance complete. All systems operational." | mail -s "Maintenance Complete" team@company.com
```

---

## Complete Automation Script

```bash
#!/bin/bash
# full-maintenance.sh

NODE="${1:-worker-3}"
SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

error() {
  echo "[ERROR] $1"
  exit 1
}

slack() {
  if [ -n "$SLACK_WEBHOOK" ]; then
    curl -X POST -H 'Content-type: application/json' \
      --data "{\"text\":\"$1\"}" \
      "$SLACK_WEBHOOK" 2>/dev/null
  fi
}

# 1. Pre-checks
log "Pre-maintenance checks..."
kubectl get node "$NODE" > /dev/null || error "Node not found"

# 2. Notify
log "Notifying team..."
slack "🔧 Maintenance starting on $NODE"

# 3. Cordon
log "Cordoning node..."
kubectl cordon "$NODE" || error "Failed to cordon node"

# 4. Drain
log "Draining node (this may take a few minutes)..."
kubectl drain "$NODE" \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout=15m || error "Failed to drain node"

# 5. SSH and maintain
log "Ready for maintenance. SSH to $NODE to proceed."
log "Run: ssh ubuntu@$NODE"
log "After maintenance, press Enter to continue..."
read -p "Press Enter when maintenance complete..."

# 6. Verify recovery
log "Verifying node recovery..."
sleep 30

for i in {1..30}; do
  STATUS=$(kubectl get node "$NODE" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null)
  if [ "$STATUS" = "True" ]; then
    log "✅ Node ready"
    break
  fi
  log "Waiting for node... ($i/30)"
  sleep 10
done

# 7. Uncordon
log "Uncordoning node..."
kubectl uncordon "$NODE" || error "Failed to uncordon"

# 8. Wait for pods
log "Waiting for pods to reschedule..."
sleep 60

# 9. Verify health
log "Verifying application health..."
kubectl get nodes
kubectl get deployments --all-namespaces

log "✅ Maintenance complete!"
slack "✅ Maintenance complete. $NODE back in service."
```

---

## Multiple Nodes Maintenance

```bash
#!/bin/bash
# maintain-multiple-nodes.sh

NODES=("worker-1" "worker-2" "worker-3")

for node in "${NODES[@]}"; do
  echo "========== Maintaining $node =========="
  
  # Cordon
  kubectl cordon "$node"
  
  # Drain
  kubectl drain "$node" \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --timeout=15m
  
  # SSH and maintain
  echo "Maintaining $node - press Enter when done"
  read
  
  # Uncordon and wait
  kubectl uncordon "$node"
  
  # Wait before next node
  sleep 120
done

echo "All nodes maintained!"
```

---

## Troubleshooting During Maintenance

### Issue: Drain Hangs

```bash
# Drain stuck? Check what's blocking
kubectl get pdb --all-namespaces

# Check problematic pod
kubectl describe pod <stuck-pod>

# Options:
# 1. Wait (sometimes pods take time to terminate)
# 2. Delete problematic pod manually
# 3. Use --force flag (dangerous)
```

### Issue: Pod Won't Reschedule

```bash
# After uncordon, pods not scheduling back
kubectl describe node "$NODE"

# Check for taints
kubectl describe node "$NODE" | grep Taints

# Check resources
kubectl top node "$NODE"

# Solution: May need to wait, check events
kubectl get events -n default
```

---

## Best Practices Checklist

- [ ] Notify team before maintenance
- [ ] Verify backup exists
- [ ] Cordon before draining
- [ ] Check drain completes successfully
- [ ] Perform maintenance
- [ ] Verify node comes back
- [ ] Uncordon node
- [ ] Wait for pods to reschedule
- [ ] Verify applications are healthy
- [ ] Notify team of completion

---

## Summary

Node maintenance workflow:

**Pre-Maintenance:**
- Notify team
- Verify backups
- Check application status

**Maintenance:**
- Cordon node
- Drain pods
- SSH and perform work
- Monitor recovery

**Post-Maintenance:**
- Verify node health
- Uncordon node
- Wait for pods
- Verify applications
- Notify completion

**Automation:**
- Script the process
- Handle multiple nodes
- Log everything
- Error handling

**Module 11 Chapter 35 Complete - 4 Files!**

You now understand node maintenance:
- File 01: Drain and cordon
- File 02: Pod Disruption Budgets
- File 03: Graceful shutdown
- File 04: Complete maintenance workflow

**Next:** Chapter 36 covers etcd management.
