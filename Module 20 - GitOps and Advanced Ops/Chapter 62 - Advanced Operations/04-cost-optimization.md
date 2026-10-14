# Kubernetes Cost Optimization & Cloud FinOps

Unmonitored Kubernetes clusters can quickly become cloud cost centers. Teams frequently over-allocate CPU and memory requests "just in case," resulting in clusters running at 15–20% average hardware utilization while paying for 100% of underlying cloud VM capacity.

Kubernetes cost optimization requires a three-pillar FinOps loop: **Visibility (Kubecost)**, **Workload Rightsizing (VPA/Goldilocks)**, and **Infrastructure Optimization (Karpenter & Spot Instances)**.

---

## 1. The Kubernetes Cost Waste Architecture

```
  +-------------------------------------------------------------------------+
  |                      Cloud Node Capacity (e.g. 16 vCPU, 64 GB)          |
  +-------------------------------------------------------------------------+
  |  Reserved Pod Requests: 14 vCPU / 56 GB  (87% Allocated by Scheduler)  |
  +-------------------------------------------------------------------------+
  |  ACTUAL USAGE: 2.5 vCPU / 12 GB          (15% Real Consumption)         |
  +-------------------------------------------------------------------------+
                                      ^
                                      |
                     [ 72% UNUSED PAID CAPACITY (WASTE) ]
```

When pods specify requests far higher than their actual usage, the Kubernetes scheduler refuses to place new pods on that node, forcing the cluster autoscaler to spin up expensive additional nodes unnecessarily.

---

## 2. Cost Visibility & Allocation: Kubecost

**Kubecost** (built on CNCF OpenCost) calculates exact dollar costs for every namespace, deployment, label, and persistent volume by marrying real-time Kubernetes metric telemetry with cloud billing APIs (AWS CUR, GCP Cloud Billing, Azure Cost Management).

### Quick Installation via Helm
```bash
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm repo update

helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost \
  --create-namespace \
  --set kubecostToken="Y29zdC1vcHRpbWl6YXRpb25AZGV2b3BzLmNvbQ==" \
  --set prometheus.server.persistentVolume.size="32Gi"
```

### Accessing Cost Allocations
```bash
kubectl port-forward -n kubecost deployment/kubecost-cost-analyzer 9090:9090
```
- Open `http://localhost:9090` to view:
  - Monthly cost per namespace and department.
  - Idle capacity waste breakdown.
  - Immediate rightsizing recommendations.

---

## 3. Workload Rightsizing: Vertical Pod Autoscaler (VPA)

Rather than guessing pod resource requests, use **VPA** in recommendation mode to analyze historical usage patterns and calculate statistical p95/p99 recommendations.

### VPA in Recommendation Mode (`updateMode: "Off"`)
Deploying VPA with `updateMode: "Off"` safely provides recommendations without restarting active pods:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: payment-service-vpa
  namespace: payments
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: payment-service
  updatePolicy:
    updateMode: "Off" # Generates recommendations without terminating pods
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "4000m"
          memory: "8Gi"
```

### Viewing VPA Recommendations
```bash
kubectl get vpa payment-service-vpa -n payments -o yaml
```

**Sample Output:**
```yaml
recommendation:
  containerRecommendations:
    - containerName: payment-service
      target:
        cpu: 250m
        memory: 512Mi
      lowerBound:
        cpu: 100m
        memory: 256Mi
      upperBound:
        cpu: 800m
        memory: 1024Mi
```

---

## 4. Intelligent Node Provisioning: Karpenter

Traditional Cluster Autoscalers scale fixed Auto Scaling Groups (ASGs), leading to slow provisioning times (3–5 minutes) and bin-packing fragmentation. 

**Karpenter** bypasses ASGs, launching the exact instance types required in under 45 seconds and actively consolidating underutilized nodes.

```
       Incoming Pod (Request: 7 vCPU, 28GB)
                       |
                       v
         +---------------------------+
         |     Karpenter Engine      |
         +-------------+-------------+
                       |
         Selects cheapest spot instance (e.g. c6i.2xlarge)
         and provisions VM directly via Cloud EC2 API
                       |
                       v
         +---------------------------+
         | EC2 Instance Ready (35s)  |
         +---------------------------+
```

### Karpenter NodePool with Spot Consolidation

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: spot-cost-optimized
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"] # Prefers spot (70-90% discount)
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64", "arm64"]     # Graviton (ARM64) is ~20% cheaper
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
      nodeClassRef:
        name: default-ec2nodeclass
  
  # Aggressive Cost Consolidation
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h # Replace nodes after 30 days
```

When pods terminate, Karpenter automatically identifies nodes with low utilization, drains them, and replaces multiple partially-filled nodes with a single smaller, cheaper instance.

---

## 5. Spot Interruption Handling

Spot instances offer up to 90% savings compared to on-demand, but cloud providers can reclaim them with short notice (2 minutes on AWS, 30 seconds on Azure).

Workloads running on Spot must be fault-tolerant:
1. **Spread Pods Across Failure Domains**:
   ```yaml
   topologySpreadConstraints:
     - maxSkew: 1
       topologyKey: "topology.kubernetes.io/zone"
       whenUnsatisfiable: DoNotSchedule
       labelSelector:
         matchLabels:
           app: payment-api
   ```
2. **Handle Interruption Signals Gracefully**:
   Karpenter natively monitors cloud interruption queues via SQS/EventBridge, immediately marking nodes as unschedulable (`cordon`), launching replacement nodes ahead of time, and evicting pods gracefully.

---

## 6. Network & Storage Cost Optimization

### Avoiding Cross-AZ Data Transfer Fees
In cloud environments (AWS, GCP, Azure), data transferred between two Availability Zones inside the same region incurs cross-AZ egress charges.

Enable **Topology Aware Hints** to instruct Kubernetes Service routing to keep requests within the caller's local availability zone:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
  annotations:
    service.kubernetes.io/topology-mode: "Auto"
spec:
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 8080
```

### Cleaning Up Orphaned PVCs & Disks
When namespaces or statefulsets are deleted, PersistentVolumes are frequently left orphaned in `Released` or detached cloud state, continuing to accrue storage costs.

```bash
# Find unbound PVs
kubectl get pv | grep -E "Released|Failed"

# Delete detached EBS volumes via AWS CLI
aws ec2 describe-volumes --filters Name=status,Values=available --query "Volumes[*].VolumeId" --output text
```

---

## 7. Kubernetes FinOps Checklist

| Category | Optimization Action | Potential Savings |
| :--- | :--- | :--- |
| **Compute** | Replace On-Demand worker nodes with Spot for stateless apps | **60% - 90%** |
| **Architecture**| Migrate x86 nodes to ARM64 (AWS Graviton / GCP Tau) | **20%** |
| **Workloads** | Apply VPA p95 recommendations to CPU/Memory requests | **30% - 50%** |
| **Scaling** | Enable Karpenter `WhenUnderutilized` consolidation | **15% - 25%** |
| **Networking**| Implement `service.kubernetes.io/topology-mode: Auto` | **10% - 30% on network bill** |
| **Environments**| Automatically shut down dev/staging clusters outside business hours | **65% on non-prod spend** |
