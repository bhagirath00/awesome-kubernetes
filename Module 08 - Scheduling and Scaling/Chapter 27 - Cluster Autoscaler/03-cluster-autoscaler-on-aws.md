# Cluster Autoscaler on AWS

## AWS-Specific Features

### Spot Instances

Mix on-demand and spot for cost savings:

```bash
# Create ASG with both on-demand and spot
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name mixed-asg \
  --instance-types t3.medium t3.large m5.large \
  --mixed-instances-policy \
    '{"InstancesDistribution":{"OnDemandPercentageAboveBaseCapacity":20,"SpotInstancePools":2}}'

# CA automatically uses mix
# 80% spot (cheap), 20% on-demand (stable)
```

---

## Launch Template Configuration

```bash
# Create launch template with custom settings
aws ec2 create-launch-template \
  --launch-template-name eks-template \
  --launch-template-data '{
    "ImageId":"ami-0c55b159cbfafe1f0",
    "InstanceType":"t3.medium",
    "TagSpecifications":[
      {
        "ResourceType":"instance",
        "Tags":[
          {"Key":"cluster","Value":"my-cluster"}
        ]
      }
    ]
  }'

# ASG uses template
# CA provisions matching nodes
```

---

## Real-World Example: E-Commerce

```bash
# Staging cluster (cost-optimized)
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name staging-asg \
  --min-size 1 \
  --max-size 20 \
  --desired-capacity 2

# CA configuration
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-config
  namespace: kube-system
data:
  nodes.max: "20"
  nodes.min: "1"
  scale-down-enabled: "true"
  scale-down-unneeded-time: "5m"
EOF

# Production cluster (high-availability)
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name prod-asg \
  --min-size 10 \
  --max-size 100 \
  --desired-capacity 20

# CA configuration
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-config
  namespace: kube-system
data:
  nodes.max: "100"
  nodes.min: "10"  # Always have baseline
  scale-down-enabled: "false"  # Manual scale-down
  scale-down-unneeded-time: "30m"
EOF
```

---

## Capacity Reservation

Reserve capacity for critical workloads:

```bash
# Create capacity reservation in AWS
aws ec2 create-capacity-reservation \
  --availability-zone us-east-1a \
  --instance-type t3.xlarge \
  --instance-platform Linux/UNIX \
  --quantity 5

# ASG uses reservation
# CA provisioning prioritizes it
```

---

## CloudWatch Metrics

Monitor CA via CloudWatch:

```bash
# Get current ASG metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/AutoScaling \
  --metric-name GroupDesiredCapacity \
  --dimensions Name=AutoScalingGroupName,Value=eks-node-group \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 300 \
  --statistics Average
```

Visualize in CloudWatch dashboard.

---

## Cost Tracking

```bash
# Track cluster autoscaling costs
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=TAG,Key=cluster

# Compare before/after CA
# Expected: 30-50% cost reduction (with proper config)
```

---

## High Availability Setup

Multi-AZ with CA:

```bash
# ASG spans multiple AZs
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name multi-az-asg \
  --availability-zones us-east-1a us-east-1b us-east-1c \
  --min-size 9  # 3 per AZ
  --max-size 30
  --desired-capacity 9

# CA provisions nodes across zones
# Automatic spreading for HA
```

---

## Integration with ALB

```yaml
# Pod using ALB
apiVersion: v1
kind: Service
metadata:
  name: web-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: web

# When CA scales nodes:
# 1. New nodes join
# 2. New pods scheduled
# 3. ALB discovers new pods
# 4. Traffic routed automatically
```

---

## Reserved Instance Strategy

```bash
# Use Reserved Instances (RI) for baseline
# Use On-Demand for burst
# Use Spot for overflow

# Baseline: 10 nodes (on RI)
# Burst capacity: up to 100 nodes (on-demand + spot)

# Cost: Cheap baseline + flexibility for spikes
```

---

## Troubleshooting AWS Issues

### Node Launch Failures

```bash
# Check AWS errors
kubectl logs -n kube-system deployment/cluster-autoscaler | grep "failed to"

# Check ASG/Launch Template
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names eks-asg

# Check VPC/Subnet capacity
aws ec2 describe-subnets --subnet-ids subnet-xxx
```

### IAM Permission Issues

```bash
# Verify role has permissions
aws iam get-role-policy \
  --role-name eks-ca-role \
  --policy-name eks-ca-policy

# If missing: Add policy
aws iam put-role-policy \
  --role-name eks-ca-role \
  --policy-name eks-ca-policy \
  --policy-document file://ca-policy.json
```

---

## Monitoring Dashboard

```bash
# Create CloudWatch dashboard
aws cloudwatch put-dashboard \
  --dashboard-name EKS-Autoscaling \
  --dashboard-body '{
    "widgets": [
      {
        "metrics": [
          ["AWS/AutoScaling","GroupDesiredCapacity","AutoScalingGroupName","eks-asg"],
          ["AWS/AutoScaling","GroupTerminatingInstances","AutoScalingGroupName","eks-asg"]
        ]
      }
    ]
  }'
```

---

## Best Practices for AWS

1. **Use Spot instances** (20-80% cost savings)
2. **Multi-AZ** (high availability)
3. **Mix reserved + on-demand** (predictable cost)
4. **Monitor CloudWatch** (track spending)
5. **Test scale-down** (prevent cost surprises)

---

## Summary

CA on AWS EKS:

**Setup:**
- IAM policy
- Helm install
- Configure via ConfigMap

**Features:**
- Spot instances for cost
- Multi-AZ for HA
- Launch templates
- Capacity reservations

**Monitoring:**
- CloudWatch metrics
- Cost tracking
- Cluster dashboard

**Next:** File 04 covers best practices for all autoscaling.
