# What is Cluster Autoscaler?

## Introduction

You have HPA scaling pods, but what if you run out of node capacity?

```
Traffic spike:
  HPA tries to scale: 3 replicas → 30 replicas
  
Scheduler looks for nodes:
  Result: "No nodes available"
  Pods: Pending forever
```

**Cluster Autoscaler (CA)** solves this by adding nodes to the cluster.

---

## How Cluster Autoscaler Works

```
1. Pod created but can't be scheduled
   (no node has capacity)
   ↓
2. Cluster Autoscaler detects pending pod
   ↓
3. CA checks cloud provider (AWS, GCP, Azure)
   ↓
4. CA adds new node to cluster
   ↓
5. kubelet joins cluster
   ↓
6. Scheduler places pending pod on new node
```

---

## Three Layers of Scaling

```
HPA: 3 replicas → 30 replicas
  Pods scale, but on existing nodes

Cluster Autoscaler: 3 nodes → 10 nodes
  Nodes scale when pods can't fit

VPA: Pod requests 500m → 200m
  Pods get right-sized resources
```

All three work together:

```
User traffic surge:
  1. VPA: Each pod is right-sized (150m CPU)
  2. HPA: Add more pods (30 replicas)
  3. CA: Add nodes to fit new pods (30 nodes needed)
```

---

## Requirements

1. **Cloud Provider Integration** (AWS, GCP, Azure, etc.)
   - CA talks to AWS to add EC2 instances

2. **Node Groups/Scaling Groups** (Auto Scaling Group)
   - ASG configured with min/max size

3. **Metrics Server** (for capacity calculation)
   - CA knows node capacity

---

## Example: AWS

### Setup Cluster Autoscaler

```bash
# Install Cluster Autoscaler
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  -n kube-system \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=us-east-1
```

### Tag Nodes for Auto-Discovery

```bash
# Tag AWS ASG
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name eks-node-group \
  --tags "Key=k8s.io/cluster-autoscaler/my-cluster,Value=owned"
```

CA finds this ASG and manages its size.

---

## Scale-Up Behavior

```
Pending pod:
  Requires: 2CPU, 4GB

Node capacity:
  3 nodes, each 2CPU, 4GB
  Each node full
  No space for pending pod

Cluster Autoscaler:
  1. Detects pending pod
  2. Checks: "Can I add a node that fits?"
  3. Yes: Add new node to ASG
  4. Wait for node to join
  5. Scheduler places pod
```

---

## Scale-Down Behavior

```
Nodes become idle:
  Node1: 10% utilization
  Node2: 15% utilization
  Mostly empty

Cluster Autoscaler (after delay):
  1. Checks if pods can move to other nodes
  2. If yes: Drain node (move pods)
  3. Remove empty node from ASG
  4. Terminate EC2 instance
  5. Save money
```

Scale-down is slower than scale-up (careful not to disrupt).

---

## Real-World Timeline

```
Time 0:00 — Normal traffic
  Cluster: 3 nodes, 50% full
  Pods: 9 running
  Pending: None

Time 0:05 — Traffic spike 10x
  HPA detects high CPU
  HPA scales: 9 pods → 100 pods
  Scheduler: 91 pods pending (no space)

Time 0:10 — Cluster Autoscaler detects
  CA: "91 pods can't fit"
  CA: "Need 10 more nodes"
  CA: Adds 10 nodes to ASG

Time 0:20 — Nodes launching
  AWS: EC2 instances starting
  kubelet: Joining cluster

Time 0:30 — Nodes ready
  Scheduler: Places 91 pending pods
  Cluster: 13 nodes, 90% full

Time 2:00 — Traffic returns normal
  HPA scales down: 100 pods → 9 pods
  Many nodes become idle

Time 5:00 — Cluster Autoscaler evaluates
  CA: "3 nodes are 10% full, can consolidate"
  CA: Drains and removes 3 nodes

Time 5:05 — Scale-down complete
  Cluster: Back to 10 nodes
  Cost: Reduced
```

---

## CA vs. HPA vs. VPA

```
HPA: Add pod replicas
  Response time: 30 seconds
  Scales: Replicas (1-1000)

CA: Add cluster nodes
  Response time: 2-5 minutes
  Scales: Nodes (limited by ASG)

VPA: Adjust pod resources
  Response time: Pod recreation
  Scales: Per-pod efficiency
```

**Timeline:**
1. Pod needs resources: VPA sizes it
2. No room on nodes: HPA requests more replicas
3. Nodes full: CA adds nodes
4. Traffic subsides: HPA scales down
5. Nodes idle: CA removes nodes

---

## CA Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-status-config
  namespace: kube-system
data:
  nodes.max: "100"         # Max nodes in cluster
  nodes.min: "3"           # Min nodes (always keep)
  scale-down-enabled: "true"
  scale-down-delay-after-add: "10m"  # Wait 10 min after scale-up
  scale-down-delay-after-failure: "5m"
  scale-down-unneeded-time: "10m"    # Node idle for 10 min
  skip-nodes-with-local-storage: "true"  # Don't remove nodes with local data
  skip-nodes-with-system-pods: "false"   # Remove nodes with system pods
```

---

## Common Mistakes

### Mistake 1: ASG Not Configured

```yaml
# WRONG — CA can't find ASG
# CA installed, but ASG size is static
AWS ASG: min: 3, max: 3
# Can't scale up (max = 3)
```

**Fix:** Set max > current size.

### Mistake 2: Pods with Local Storage

```yaml
# WRONG — CA won't remove node
volumeMounts:
- name: data
  mountPath: /data
volumes:
- name: data
  emptyDir: {}  # Local data

# CA: "Can't remove node, data will be lost"
# Node stays forever
```

### Mistake 3: Pod Anti-Affinity

```yaml
# WRONG — CA can't consolidate
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchExpressions:
      - key: app
        operator: In
        values: [cache]
    topologyKey: kubernetes.io/hostname

# Each pod on different node
# CA: "Can't move pods, violates affinity"
# Keeps all nodes
```

---

## Monitoring CA

```bash
# Check CA logs
kubectl logs -n kube-system deployment/cluster-autoscaler

# Check scale events
kubectl describe nodes | grep "scale"

# View ASG status
aws autoscaling describe-auto-scaling-groups

# Monitor node count
kubectl get nodes -w
```

---

## Summary

Cluster Autoscaler adds nodes when needed:

**How it works:**
1. Detect pending pods
2. Add nodes to ASG
3. Scheduler places pods
4. Remove idle nodes

**Requires:**
- Cloud provider (AWS, GCP, Azure)
- Node scaling group (ASG)
- Min/max size configured

**With HPA + VPA:**
- VPA: Right-size pods
- HPA: Scale replicas
- CA: Scale nodes

**Next:** File 02 covers configuring CA on AWS EKS.
