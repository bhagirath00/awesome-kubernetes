# Upgrading Worker Nodes

## Worker Node Components

Nodes run your pods:

```
┌──────────────────────┐
│   Worker Node v1.26  │
├──────────────────────┤
│ kubelet (manages pods)
│ kube-proxy (networking)
│ container-runtime (Docker/containerd)
├──────────────────────┤
│ Several pods (app, system)
└──────────────────────┘

Upgrade strategy:
1. Drain pods (move to other nodes)
2. Cordon node (block new pods)
3. Upgrade kubelet
4. Uncordon (accept new pods)
```

---

## Node Upgrade Overview

```
Node state machine during upgrade:

v1.26 (Ready)
   ↓
Cordon (SchedulingDisabled)
   ↓
Drain (pods moved away)
   ↓
Upgrade kubelet
   ↓
Restart
   ↓
v1.27 (Ready)
   ↓
Uncordon (accept new pods)
```

---

## Step-by-Step Node Upgrade

### Step 1: List All Nodes

```bash
kubectl get nodes -o wide

# Output:
# NAME      STATUS   ROLES   VERSION   INTERNAL-IP
# worker-1  Ready    <none>  v1.26.3   10.0.1.10
# worker-2  Ready    <none>  v1.26.3   10.0.1.11
# worker-3  Ready    <none>  v1.26.3   10.0.1.12
```

### Step 2: Cordon First Node

Prevent new pods from being scheduled:

```bash
kubectl cordon worker-1

# Output:
# node/worker-1 cordoned

kubectl get nodes

# Output:
# NAME      STATUS                     ROLES   VERSION
# worker-1  Ready,SchedulingDisabled   <none>  v1.26.3
# worker-2  Ready                      <none>  v1.26.3
```

### Step 3: Drain Node

Move all running pods to other nodes:

```bash
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data

# Flags:
# --ignore-daemonsets    (DaemonSets are node-specific, can't move)
# --delete-emptydir-data (emptyDir volumes deleted, pods recreated)

# This moves:
# - Deployments → other nodes
# - StatefulSets → other nodes
# - Jobs → other nodes
# - DaemonSets → stay on node

# Output shows what's happening:
# WARNING: ignoring daemonsets-managed pods: kube-proxy-abc123
# pod/my-app-xyz789 evicted
# pod/database-replica evicted
```

Wait for drain to complete. Check progress:

```bash
kubectl get pods --all-namespaces -o wide | grep worker-1
# Should show fewer pods gradually
```

### Step 4: SSH to Node and Upgrade

```bash
ssh ubuntu@worker-1

# Check version
kubelet --version
# v1.26.3

# Upgrade kubelet and kubectl
sudo apt-get update
sudo apt-get upgrade kubelet kubectl

# New version
sudo systemctl restart kubelet

# Verify
kubelet --version
# v1.27.2
```

### Step 5: Uncordon Node

Allow new pods to be scheduled:

```bash
kubectl uncordon worker-1

# Output:
# node/worker-1 uncordoned

kubectl get nodes

# Output:
# NAME      STATUS   ROLES   VERSION
# worker-1  Ready    <none>  v1.27.2
```

### Step 6: Verify Node is Healthy

```bash
# Check node status
kubectl get node worker-1 -o yaml | grep -A 5 status

# Check pods are scheduling back
kubectl get pods --all-namespaces -o wide | grep worker-1

# Should see your workloads returning
```

### Step 7: Repeat for Other Nodes

```bash
kubectl cordon worker-2
kubectl drain worker-2 --ignore-daemonsets --delete-emptydir-data
# ... upgrade on worker-2 ...
kubectl uncordon worker-2

kubectl cordon worker-3
kubectl drain worker-3 --ignore-daemonsets --delete-emptydir-data
# ... upgrade on worker-3 ...
kubectl uncordon worker-3
```

---

## Real-World Node Upgrade Workflow

```bash
# Monday 2am: Start maintenance window

NODES=(worker-1 worker-2 worker-3 worker-4 worker-5)

for node in "${NODES[@]}"; do
  echo "Upgrading $node..."
  
  # Cordon
  kubectl cordon "$node"
  
  # Drain
  kubectl drain "$node" \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --timeout=5m
  
  # SSH and upgrade
  ssh "ubuntu@$node" <<'EOF'
    sudo systemctl stop kubelet
    sudo apt-get update
    sudo apt-get upgrade -y kubelet kubectl
    sudo systemctl start kubelet
EOF
  
  # Wait for node to be Ready
  sleep 10
  until kubectl get node "$node" | grep -q "Ready"; do
    echo "Waiting for $node to be Ready..."
    sleep 5
  done
  
  # Uncordon
  kubectl uncordon "$node"
  
  # Wait before next node (let pods stabilize)
  sleep 60
done

echo "All nodes upgraded!"
```

---

## Handling Stuck Pods

What if drain doesn't work?

```bash
# Some pod won't evict
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
# error: cannot evict pod "stubborn-pod", not all pods evicted

# Options:

# Option 1: Force evict
kubectl drain worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force

# Option 2: Manually delete the pod
kubectl delete pod stubborn-pod -n default --grace-period=0

# Then retry drain
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data

# Option 3: Add --skip-wait-for-delete-timeout
kubectl drain worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --skip-wait-for-delete-timeout=true
```

---

## Drain vs Cordon

```
Cordon:
- Prevents NEW pods from scheduling
- Existing pods stay running
- Use: for maintenance, updates

Drain:
- Evicts ALL pods (except DaemonSets)
- Pods moved to other nodes
- Use: before shutdown, upgrade, removal

Typical workflow: Cordon → Drain → Maintenance → Restart → Uncordon
```

---

## Troubleshooting Node Issues

### Issue 1: Pod Stuck in Pending

```bash
# After uncordon, pod stuck Pending
kubectl describe pod stuck-pod

# Common cause: Node resources exhausted
kubectl describe node worker-1

# Solution: Wait for other pods to reschedule
kubectl get pod --all-namespaces -o wide | grep worker-1

# If still stuck, check taints/tolerations
kubectl describe node worker-1 | grep Taints
```

### Issue 2: Drain Timeout

```bash
# Drain takes too long
kubectl drain worker-1 --timeout=10m

# If still stuck:
# - Check what pods won't evict
kubectl get pods --all-namespaces --field-selector spec.nodeName=worker-1

# - Check if Pod Disruption Budgets blocking eviction
kubectl get pdb --all-namespaces
```

### Issue 3: Kubelet Won't Start

```bash
# After upgrade, kubelet won't start
ssh worker-1
sudo systemctl status kubelet

# Check logs
sudo journalctl -u kubelet -n 100

# Common fixes:
sudo systemctl restart docker
sudo systemctl restart kubelet

# If still fails, check configuration
cat /etc/kubernetes/kubelet/kubeconfig.conf
```

---

## Best Practices

1. **Cordon Before Drain**
   - Stops new pods from being scheduled
   - Ensures drain completes faster

2. **Use --ignore-daemonsets**
   - DaemonSets tied to node
   - Can't evict, that's expected

3. **Upgrade Incrementally**
   - One or two nodes at a time
   - Rest handle traffic
   - Slower but safer

4. **Monitor Health**
   - Check node status after uncordon
   - Verify pods are rescheduling
   - Watch for CrashLoopBackOff

5. **Test on Non-Critical Nodes**
   - If multiple node groups
   - Upgrade test nodes first

---

## Summary

Worker node upgrades:

**Process:**
- Cordon (stop scheduling)
- Drain (evict pods)
- Upgrade kubelet
- Restart
- Uncordon (resume scheduling)

**Commands:**
- kubectl cordon node
- kubectl drain node --ignore-daemonsets --delete-emptydir-data
- kubectl uncordon node

**Safety:**
- Gradual rollout (few nodes at time)
- Monitor between upgrades
- Have rollback plan

**Next:** File 04 covers EKS-specific upgrades.
