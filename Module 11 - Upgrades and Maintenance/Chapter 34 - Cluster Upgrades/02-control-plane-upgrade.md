# Upgrading the Control Plane

## Control Plane Components

Your cluster's brain:

```
┌─────────────────────────────────────┐
│      Kubernetes Control Plane       │
├─────────────────────────────────────┤
│ API Server       (listens for requests)
│ Scheduler        (places pods)
│ Controller Mgr   (maintains desired state)
│ etcd             (stores everything)
├─────────────────────────────────────┤
│  Upgraded together (same version)   │
└─────────────────────────────────────┘
```

All control plane components must be the same version.

---

## Version Compatibility Rule

```
Control Plane (v1.27) ← must be same or newer
   ↓
Worker Nodes (v1.26, v1.27) ← within 1 minor version

WRONG:
Control Plane v1.25 + Worker Nodes v1.27 ❌

RIGHT:
Control Plane v1.27 + Worker Nodes v1.26 or v1.27 ✅
```

---

## Pre-Upgrade Checklist

```bash
# 1. Verify current version
kubectl version --short

# 2. Check API server health
kubectl cluster-info

# 3. Verify all nodes are Ready
kubectl get nodes

# 4. Backup etcd
# (See Chapter 36 for full details)

# 5. Ensure you have maintenance window
# No deployments/scaling during upgrade

# 6. Have rollback plan
```

---

## Control Plane Upgrade on Self-Managed Cluster

### Scenario: Self-hosted Kubernetes (not EKS)

```bash
# 1. SSH to master node
ssh ubuntu@master.cluster.local

# 2. Stop kubelet
sudo systemctl stop kubelet

# 3. Upgrade kubeadm tool
sudo apt-get update
sudo apt-get upgrade kubeadm

# 4. Plan upgrade (dry-run)
sudo kubeadm upgrade plan

# Output shows what will be upgraded:
# [upgrade/versions] Target version: v1.27.0
# [upgrade/kubeadm] kubeadm already v1.27.0
# [upgrade] Are you ready to upgrade the cluster? [y/N]: 

# 5. Execute upgrade
sudo kubeadm upgrade apply v1.27.0

# 6. Upgrade kubelet and kubectl
sudo apt-get upgrade kubelet kubectl

# 7. Restart kubelet
sudo systemctl start kubelet

# 8. Verify upgrade
kubectl get nodes
kubectl version --short
```

---

## Control Plane Upgrade on AWS EKS

### Scenario: Managed Kubernetes (EKS)

EKS manages control plane automatically. You only control the cluster version:

```bash
# View current version
aws eks describe-cluster --name my-cluster --query 'cluster.version'

# List available versions
aws eks describe-addon-versions

# Update control plane to v1.27
aws eks update-cluster-version \
  --name my-cluster \
  --kubernetes-version 1.27

# Monitor upgrade (takes 20-30 minutes)
aws eks describe-cluster \
  --name my-cluster \
  --query 'cluster.status'

# Status progression:
# ACTIVE → UPDATING → ACTIVE
```

---

## Real-World Control Plane Upgrade

```bash
# Day 1, 2am: Maintenance window starts

# Step 1: Backup etcd (Chapter 36 covers this)
sudo /opt/kubernetes/backup-etcd.sh

# Step 2: Cordon master node (prevent scheduling)
kubectl cordon master-1

# Step 3: Upgrade control plane
sudo kubeadm upgrade apply v1.27.0 --yes

# Step 4: Upgrade kubelet
sudo systemctl restart kubelet

# Step 5: Uncordon master
kubectl uncordon master-1

# Step 6: Verify
kubectl get nodes
kubectl version --short

# Step 7: Monitor for 1 hour
watch kubectl get pods -A
```

---

## Troubleshooting Control Plane Issues

### Issue 1: API Server Won't Start

```bash
# Error: CrashLoopBackOff on api-server pod
kubectl describe pod kube-apiserver-master-1 -n kube-system

# Check logs
journalctl -u kubelet -n 50

# Common causes:
# 1. Configuration file incompatible
# 2. TLS certificate issues
# 3. etcd not responding

# Fix: Verify etcd is healthy
sudo systemctl status etcd
```

### Issue 2: Pending API Requests

```bash
# During upgrade, API might be unresponsive
# Solution: Wait and retry

# Check if API is responding
kubectl cluster-info

# If timeout, wait for API pod to restart
# Monitor logs
kubectl logs -n kube-system -l component=kube-apiserver -f
```

### Issue 3: Stuck on Version Check

```bash
# kubeadm reports version mismatch
kubeadm upgrade plan
# error: the cluster version is higher than kubeadm version

# Fix: Upgrade kubeadm first
sudo apt-get upgrade kubeadm
```

---

## Control Plane Upgrade with High Availability

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Master-1 v1.26 │    │  Master-2 v1.26 │    │  Master-3 v1.26 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        ↓                      ↓                        ↓
┌─────────────────────────────────────────────────────────────┐
│            Load Balancer (Kubernetes API)                   │
└─────────────────────────────────────────────────────────────┘

Upgrade strategy:
1. Upgrade Master-1 → v1.27 (API requests go to Master-2, Master-3)
2. Upgrade Master-2 → v1.27
3. Upgrade Master-3 → v1.27
4. All at v1.27 ✓
```

---

## Gradual Rollout

For HA clusters, upgrade one master at a time:

```bash
# Master-1: Upgrade to v1.27
sudo kubeadm upgrade apply v1.27.0 --yes
sudo systemctl restart kubelet

# Wait for Master-1 to be Ready
kubectl get nodes

# Master-2: Upgrade to v1.27
ssh master-2
sudo kubeadm upgrade apply v1.27.0 --yes
sudo systemctl restart kubelet

# Master-3: Upgrade to v1.27
ssh master-3
sudo kubeadm upgrade apply v1.27.0 --yes
sudo systemctl restart kubelet

# Verify all Ready
kubectl get nodes
```

---

## Best Practices

1. **Always Backup First**
   - Backup etcd before any upgrade
   - Test restore procedure

2. **Upgrade During Maintenance**
   - Scheduled window
   - All team notified
   - Monitoring active

3. **Test on Staging**
   - Upgrade staging first
   - Run full test suite
   - Then upgrade production

4. **Monitor Continuously**
   - Watch API server logs
   - Monitor node status
   - Check pod status

5. **Have Rollback Plan**
   - Know how to downgrade
   - Test rollback on staging
   - Document procedure

---

## Summary

Control plane upgrades:

**Components:**
- All upgraded together (same version)
- etcd, API server, scheduler, controller-mgr

**Compatibility:**
- Control plane ≥ Worker nodes
- Within 1 minor version

**Process:**
- Backup etcd first
- Drain master node
- Upgrade components
- Restart kubelet
- Verify health

**HA Clusters:**
- One master at a time
- Rest handle API requests
- Gradual rollout

**Next:** File 03 covers worker node upgrades.
