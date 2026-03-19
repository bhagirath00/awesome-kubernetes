# Scaling ReplicaSets

## Introduction

Scaling means changing the number of running Pod replicas to match your load. Kubernetes supports three levels of scaling: **manual scaling** (you decide the count), **Horizontal Pod Autoscaler** (Kubernetes adjusts Pod count based on metrics), and **Cluster Autoscaler** (AWS adds or removes EC2 nodes based on pending Pods). All three work together in a complete autoscaling strategy.

---

## Manual Scaling

```bash
# Scale a Deployment (which scales the active ReplicaSet)
kubectl scale deployment api-server --replicas=10 -n production

# Scale to zero — stops all Pods without deleting the Deployment
kubectl scale deployment api-server --replicas=0 -n production

# Scale back up
kubectl scale deployment api-server --replicas=3 -n production

# Watch scaling happen
kubectl get pods -n production -l app=api-server -w

# NAME                         READY   STATUS              AGE
# api-server-abc-aaa           1/1     Running             2d
# api-server-abc-bbb           1/1     Running             2d
# api-server-abc-ccc           1/1     Running             2d
# api-server-abc-ddd           0/1     Pending             2s   ← new
# api-server-abc-eee           0/1     ContainerCreating   3s   ← new
# ...
```

---

## Horizontal Pod Autoscaler (HPA)

The HPA automatically adjusts the `replicas` field of a Deployment (and thus its ReplicaSet) based on observed metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server

  minReplicas: 3        # never scale below 3 (HA: 1 per AZ)
  maxReplicas: 20       # never scale above 20 (cost guard)

  metrics:
  # Scale on CPU utilization
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # scale up when avg CPU > 70% of request

  # Scale on memory utilization
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

  # Scale on custom metric (requests per second from Prometheus)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"       # scale up when avg RPS per Pod > 100

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0       # scale up immediately — no delay
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15                  # can double replicas every 15 seconds
    scaleDown:
      stabilizationWindowSeconds: 300     # wait 5 minutes before scaling down
      policies:
      - type: Percent
        value: 20
        periodSeconds: 60                  # remove at most 20% of Pods per minute
```

```bash
kubectl apply -f hpa.yaml

# View HPA status
kubectl get hpa api-server-hpa -n production

# NAME              REFERENCE            TARGETS           MINPODS  MAXPODS  REPLICAS
# api-server-hpa    Deployment/api       45%/70%, 3/20     3        20       5
# ← currently 5 replicas, CPU at 45% (below 70% target)
```

### Prerequisites for HPA

HPA requires the **Metrics Server** to collect CPU and memory metrics from kubelet:

```bash
# Install Metrics Server on EKS
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl top pods -n production   # should return CPU/memory data
kubectl top nodes                # should return node-level data
```

For custom metrics (RPS, queue depth, etc.), install **Prometheus Adapter** or **KEDA (Kubernetes Event-Driven Autoscaling)**.

---

## KEDA — Event-Driven Autoscaling

KEDA extends HPA with dozens of triggers: SQS queue depth, Kafka consumer lag, scheduled time windows, and more:

```yaml
# Scale the order-processor based on SQS queue depth
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: order-processor      # Deployment to scale
  minReplicaCount: 0           # scale to zero when queue is empty (saves cost!)
  maxReplicaCount: 50
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123456789/orders-queue
      queueLength: "10"        # scale up when queue has > 10 messages per Pod
      awsRegion: us-east-1
    authenticationRef:
      name: keda-aws-credentials    # IRSA reference
```

With KEDA and `minReplicaCount: 0`:
- When the SQS queue is empty → 0 Pods running (zero cost)
- When messages arrive → Pods start within seconds
- When queue is drained → Pods scale back to 0

---

## Cluster Autoscaler — Adding and Removing Nodes

When HPA needs more Pods but no node has available resources, Pods stay `Pending`. The **Cluster Autoscaler** detects `Pending` Pods and adds new EC2 nodes to the cluster:

```bash
# Cluster Autoscaler is installed as a Deployment in kube-system
kubectl get deployment cluster-autoscaler -n kube-system

# Check Cluster Autoscaler logs to see scaling decisions
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=50
# I0115 10:30:00 scale_up.go:300] Scale-up: setting group eks-ng-1 size to 5
# Expanding Node Group eks-ng-1 from 3 to 5 nodes
```

### Configuring Cluster Autoscaler

```yaml
# AWS Auto Scaling Group must have Cluster Autoscaler tags
# Tag the ASG:
# k8s.io/cluster-autoscaler/enabled: "true"
# k8s.io/cluster-autoscaler/<cluster-name>: "owned"

# Set min/max node count in the ASG (respected by Cluster Autoscaler)
# Min: 3 nodes (one per AZ for HA)
# Max: 10 nodes (cost cap)
```

The Cluster Autoscaler scales down idle nodes after 10 minutes (by default) — if a node has no Pods (or only system Pods that can be moved), it is drained and terminated.

---

## The Complete Autoscaling Stack

```
Load increases → HTTP requests per second rises

    HPA detects: RPS per Pod > 100 target
    → HPA increases Deployment replicas: 5 → 10

    Scheduler: tries to schedule new Pods
    → Some nodes full → 5 Pods stay Pending

    Cluster Autoscaler detects: Pending Pods exist
    → Requests 2 new nodes from AWS Auto Scaling Group
    → New EC2 instances join the cluster in ~2 minutes

    Scheduler: now has capacity
    → Places Pending Pods on new nodes
    → All 10 Pods Running

Load decreases → RPS drops

    HPA detects: RPS per Pod < 70 target
    → HPA reduces replicas: 10 → 5

    Old Pods terminated
    → 2 nodes now underutilized

    Cluster Autoscaler detects: nodes idle for 10 minutes
    → Drains and terminates 2 EC2 nodes
    → ASG shrinks: 5 → 3 nodes

    Cost: you pay only for the 3-node baseline
```

---

## Summary

- Manual scaling: `kubectl scale deployment --replicas=N` — instant but not adaptive to load
- HPA scales Pods based on CPU, memory, or custom metrics — use `minReplicas: 3` (one per AZ) and `maxReplicas` as cost guard
- Scale-down is always more conservative than scale-up — use `stabilizationWindowSeconds: 300` to avoid flapping
- KEDA extends HPA with SQS, Kafka, and dozens of other event sources — enables scale-to-zero for cost optimization
- Cluster Autoscaler adds EC2 nodes when Pods are Pending and removes idle nodes — the complete horizontal scaling loop

This completes Chapter 16 — ReplicaSets. In Chapter 17, `01-what-is-a-statefulset.md`, you will learn about StatefulSets — the workload for applications that require stable identity and persistent storage.
