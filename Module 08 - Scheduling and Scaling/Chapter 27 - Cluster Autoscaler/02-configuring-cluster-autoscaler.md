# Configuring Cluster Autoscaler on AWS EKS

## Prerequisites

- EKS cluster running
- Managed Node Group (ASG)
- kubectl access
- Helm installed

---

## Step 1: IAM Permissions

CA needs permissions to manage ASG.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeTags",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup"
      ],
      "Resource": "arn:aws:autoscaling:*:*:autoScalingGroup:*:autoScalingGroupName/*"
    }
  ]
}
```

Create IAM role with this policy.

---

## Step 2: Install Cluster Autoscaler

```bash
# Add helm repo
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update

# Install
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  -n kube-system \
  --set autoDiscovery.clusterName=my-eks-cluster \
  --set awsRegion=us-east-1 \
  --set rbac.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::ACCOUNT_ID:role/ca-role
```

---

## Step 3: Verify Installation

```bash
# Check deployment
kubectl get deployment -n kube-system cluster-autoscaler

# Check logs
kubectl logs -n kube-system deployment/cluster-autoscaler
```

Look for:
```
Cluster autoscaler auto-discovery enabled
Auto-scaling group found: eks-node-group
```

---

## Configuration: Basic

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-status-config
  namespace: kube-system
data:
  nodes.max: "100"
  nodes.min: "3"
  scale-down-enabled: "true"
  scale-down-unneeded-time: "10m"
  scale-down-delay-after-add: "10m"
```

---

## Configuration: Conservative (Safe)

```yaml
data:
  nodes.max: "50"          # Cap cluster size
  nodes.min: "5"           # Always keep 5 nodes
  scale-down-enabled: "false"  # Manual scale-down
  scale-down-unneeded-time: "30m"
  scale-down-delay-after-add: "30m"
  scale-down-delay-after-failure: "10m"
  skip-nodes-with-local-storage: "true"
```

Good for production (stable, predictable).

---

## Configuration: Aggressive (Cost-Optimized)

```yaml
data:
  nodes.max: "200"         # Allow high scaling
  nodes.min: "1"           # Scale to near-zero
  scale-down-enabled: "true"
  scale-down-unneeded-time: "5m"  # Remove idle quickly
  scale-down-delay-after-add: "1m"
  skip-nodes-with-local-storage: "false"
```

Good for non-critical, cost-sensitive workloads.

---

## Example: E-Commerce Traffic

```yaml
# During shopping season
# Traffic varies 2x-10x
data:
  nodes.max: "200"
  nodes.min: "20"          # Baseline for peak season
  scale-down-enabled: "true"
  scale-down-unneeded-time: "15m"
  scale-down-delay-after-add: "5m"

# During off-season
# Traffic more predictable
data:
  nodes.max: "50"
  nodes.min: "5"
  scale-down-enabled: "true"
  scale-down-unneeded-time: "30m"
  scale-down-delay-after-add: "20m"
```

---

## Monitoring CA Activity

```bash
# Watch nodes scale up/down
kubectl get nodes -w

# Check CA events
kubectl get events -n kube-system | grep autoscal

# View current state
kubectl describe configmap cluster-autoscaler-status -n kube-system

# Monitor costs (CloudWatch)
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names eks-node-group \
  --query 'AutoScalingGroups[0].[DesiredCapacity,MinSize,MaxSize]'
```

---

## Multi-ASG Setup

If you have multiple node groups:

```bash
# Auto-discovery finds all tagged groups
# Tag each ASG:
# Key: k8s.io/cluster-autoscaler/my-cluster
# Value: owned

# CA automatically manages all
# No manual configuration needed
```

---

## Mixed Instance Types

```bash
# CA handles mixed ASG automatically
# ASG has: t3.medium, t3.large, m5.large

# CA provisions as needed
# May use different types based on availability
```

---

## Troubleshooting

### CA Not Scaling Up

```bash
# Check logs for errors
kubectl logs -n kube-system deployment/cluster-autoscaler | grep -i error

# Verify IAM permissions
# Check ASG configuration (max size)
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names eks-node-group

# Check pending pods
kubectl get pods --field-selector=status.phase=Pending
```

### CA Not Scaling Down

```bash
# Pods with local storage block scale-down
kubectl get pods -A -o wide | grep emptyDir

# Pods with local volume
kubectl get pods -A -o yaml | grep -A5 volumes

# Check pod disruption budgets (PDB)
kubectl get pdb -A

# PDB can prevent scale-down (protects pods)
```

### Nodes Stuck in Unhealthy State

```bash
# Check node status
kubectl describe nodes

# If "NotReady", CA won't remove
# Fix: Manually cordon or delete node
kubectl cordon node-name
kubectl delete node node-name
```

---

## Cost Optimization

```yaml
# Use spot instances (cheaper, can be interrupted)
data:
  balance-similar-node-groups: "true"
  balancing-ignore-label: "workload-type"

# Prefer on-demand for critical workloads:
nodeSelector:
  capacity-type: on-demand  # Expensive but stable

# Allow spot for batch:
nodeSelector:
  capacity-type: spot       # Cheap but interruptible
```

---

## Best Practices

1. **Set Realistic Min/Max**
   - Min: Baseline for normal load
   - Max: Cost cap + safety buffer

2. **Monitor Scale-Down**
   - Long delay prevents thrashing
   - Too long: Wastes money

3. **Test First**
   - Test in staging cluster
   - Verify scale-up and scale-down behavior

4. **Alert on Issues**
   - Alert if nodes stuck pending
   - Alert if scale-down disabled

---

## Summary

CA setup on EKS:

**Setup:**
1. IAM policy
2. Helm install
3. Configure

**Configuration:**
- nodes.max/min
- scale-down settings
- Skip policies

**Monitoring:**
- Node count
- CA events
- ASG status

**Next:** File 03 covers CA best practices.
