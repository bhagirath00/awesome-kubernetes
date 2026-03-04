# Kubernetes Use Cases

## Introduction

Kubernetes is powerful, but it is not the right tool for every situation. Knowing exactly *when* Kubernetes adds value — and why — is what separates an engineer who understands the tool from one who just knows its commands.

In this file, you will explore the most important real-world use cases for Kubernetes. Each use case includes a concrete example, the specific Kubernetes features that make it work, and the AWS services that support it.

---

## Use Case 1: Microservices Architecture

The most common use case for Kubernetes by far. A **microservices architecture** breaks a large application into many small, independent services — each responsible for one specific function.

### The Problem Without Kubernetes

Imagine an e-commerce platform with these services:
- `user-service` — handles registration, login, profiles
- `product-service` — product catalog, search, inventory
- `order-service` — placing and tracking orders
- `payment-service` — payment processing
- `notification-service` — email and SMS alerts
- `api-gateway` — routes external traffic to the right service

Without Kubernetes, you must manually decide which server runs which service, manually configure networking between them, and manually handle failures. At 6 services it is manageable. At 60 services it is chaos.

### How Kubernetes Solves It

Each microservice gets its own Deployment and Service. Kubernetes handles scheduling, networking, health checks, and scaling for all of them:

```yaml
# Each service is its own Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/order-service:v2.1.0
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: order-db-secret
              key: url
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: ecommerce
spec:
  selector:
    app: order-service
  ports:
  - port: 80
    targetPort: 8080
```

Services discover each other by DNS name inside the cluster. `order-service` reaches `payment-service` at `http://payment-service.ecommerce.svc.cluster.local`. No IP addresses to manage.

### AWS Services That Support This
- **Amazon EKS** — the cluster
- **Amazon ECR** — stores all the service images
- **AWS App Mesh** — service mesh for advanced traffic management
- **AWS Load Balancer Controller** — creates ALBs for Ingress resources

---

## Use Case 2: High-Traffic Web Applications

Applications that experience variable traffic — high during business hours, low at night, huge spikes during sales or product launches — are perfect for Kubernetes.

### The Challenge

Traditional infrastructure requires you to provision for peak load. If your peak is 10x your average, you either:
- Overprovision (pay for 10x servers all the time)
- Underprovision (crash during peaks)

### How Kubernetes Solves It

The **Horizontal Pod Autoscaler (HPA)** scales your application automatically based on real metrics. The **Cluster Autoscaler** scales the number of nodes in your EKS cluster when there are not enough resources.

```yaml
# Scale the web app Pods based on CPU and memory
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60   # Scale up when avg CPU > 60%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70   # Scale up when avg memory > 70%
```

When Black Friday traffic hits at midnight:
1. CPU across Pods exceeds 60%
2. HPA adds more Pods (up to 50)
3. If the nodes run out of capacity, Cluster Autoscaler adds more worker nodes to EKS
4. Traffic is handled automatically
5. After the spike, Pods and nodes scale back down — you stop paying

### AWS Services That Support This
- **Amazon EKS** with **Cluster Autoscaler** — automatically adds/removes EC2 nodes
- **AWS Fargate for EKS** — serverless nodes, scale to zero with no node management
- **Application Load Balancer (ALB)** — distributes traffic across Pods

---

## Use Case 3: Batch Processing and Data Pipelines

Not all workloads need to run continuously. Batch jobs — data processing, report generation, ETL pipelines, ML training — need to run, complete, and stop.

### How Kubernetes Handles Batch Workloads

Kubernetes has a **Job** resource for run-to-completion workloads and a **CronJob** resource for scheduled runs:

```yaml
# A Job that processes data from an S3 bucket and exits when done
apiVersion: batch/v1
kind: Job
metadata:
  name: daily-report-generator
  namespace: data-processing
spec:
  parallelism: 4        # Run 4 Pods simultaneously for parallel processing
  completions: 20       # Total work items to process
  backoffLimit: 3       # Retry failed Pods up to 3 times
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: report-generator
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/report-generator:v1.0
        env:
        - name: S3_BUCKET
          value: "mycompany-data-bucket"
        - name: AWS_REGION
          value: "us-east-1"
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
      serviceAccountName: s3-reader  # IAM role for S3 access via IRSA
```

```yaml
# A CronJob that runs every night at 2 AM
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-cleanup
  namespace: data-processing
spec:
  schedule: "0 2 * * *"    # 2 AM every day (cron syntax)
  concurrencyPolicy: Forbid  # Do not start a new run if previous is still running
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cleanup
            image: 123456789.dkr.ecr.us-east-1.amazonaws.com/cleanup-job:v1.0
```

Jobs only consume resources while running. You are not paying for idle containers between runs. This is a major cost saving for batch workloads.

### AWS Services That Support This
- **AWS Fargate for EKS** — ideal for batch jobs (no node management, pay per second)
- **Amazon S3** — source and destination for batch data
- **AWS Batch** — alternative if you do not need Kubernetes for batch (simpler but less flexible)

---

## Use Case 4: Multi-Tenant SaaS Platforms

SaaS companies that serve multiple customers (tenants) on shared infrastructure use Kubernetes **namespaces** and **network policies** to isolate tenants from each other.

### The Problem

Customer A must not be able to access Customer B's data, services, or even network traffic — even though they share the same Kubernetes cluster.

### How Kubernetes Solves It

Each customer gets a dedicated namespace. Network policies restrict traffic so Pods in one namespace cannot talk to Pods in another:

```yaml
# Customer A gets their own namespace
apiVersion: v1
kind: Namespace
metadata:
  name: customer-a
  labels:
    tenant: customer-a
---
# Network policy: customer-a Pods can only talk to other customer-a Pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-isolation
  namespace: customer-a
spec:
  podSelector: {}   # applies to all Pods in this namespace
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tenant: customer-a   # only allow traffic from same namespace
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tenant: customer-a
```

Combined with **ResourceQuota** to limit how much CPU/memory each tenant can use:

```yaml
# Ensure customer-a cannot consume more than their allocated resources
apiVersion: v1
kind: ResourceQuota
metadata:
  name: customer-a-quota
  namespace: customer-a
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

This way, a noisy neighbor (one customer using too many resources) cannot impact other customers.

### AWS Services That Support This
- **AWS IAM Roles for Service Accounts (IRSA)** — each tenant's service account gets its own IAM role, scoped to their AWS resources only
- **Amazon EKS** with multiple node groups — dedicated nodes for high-tier customers

---

## Use Case 5: Zero-Downtime Deployments

One of the most valuable things Kubernetes provides in production is the ability to deploy new versions of your application without any downtime.

### How It Works

Kubernetes Deployments use a **rolling update** strategy by default. You can configure exactly how it rolls out:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2         # Allow 2 extra Pods during update (6 + 2 = 8 max)
      maxUnavailable: 1   # At most 1 Pod can be down at any time
  template:
    spec:
      containers:
      - name: web
        image: myapp:v2.0
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
```

With this config, Kubernetes:
1. Starts 2 new v2.0 Pods
2. Waits for both to pass the readiness probe (confirms they are working)
3. Only then removes 1 old v1.0 Pod
4. Repeats until all 6 Pods are running v2.0
5. If any new Pod fails the readiness probe, the rollout stops automatically

Traffic only reaches a Pod after it passes the readiness probe. Users never hit a Pod that is not ready.

For more advanced patterns, you can also implement **Blue/Green deployments** and **Canary deployments** using Kubernetes and an Ingress controller — covered in later modules.

---

## Use Case 6: Machine Learning and GPU Workloads on AWS

Kubernetes on AWS is increasingly used for machine learning training and inference workloads.

### How Kubernetes Handles GPU Workloads

EKS supports GPU-enabled nodes (EC2 `p3`, `p4`, `g4` instance types). Kubernetes schedules Pods that need GPUs only on nodes that have them:

```yaml
# A Pod that requires a GPU for model training
apiVersion: v1
kind: Pod
metadata:
  name: model-training-job
spec:
  containers:
  - name: trainer
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/ml-trainer:v1.0
    resources:
      limits:
        nvidia.com/gpu: 2    # Request 2 GPUs
    env:
    - name: MODEL_S3_PATH
      value: "s3://mycompany-models/training-data/"
    - name: OUTPUT_S3_PATH
      value: "s3://mycompany-models/output/"
  nodeSelector:
    eks.amazonaws.com/nodegroup: gpu-nodes   # Schedule only on GPU nodes
  tolerations:
  - key: "nvidia.com/gpu"
    operator: "Exists"
    effect: "NoSchedule"
```

GPU nodes are expensive. By using `nodeSelector` and **taints/tolerations**, GPU nodes only run GPU workloads — your regular web services cannot accidentally land on them and waste expensive GPU capacity.

### AWS Services That Support This
- **Amazon EKS** with GPU node groups (`g4dn.xlarge`, `p3.2xlarge`)
- **AWS Neuron** for Inferentia chips (cheaper than GPU for inference)
- **Amazon S3** for training data and model storage
- **Amazon SageMaker** — alternative if you want fully managed ML (less Kubernetes, more AWS)

---

## Use Case 7: Disaster Recovery and Multi-Region Deployments

For critical applications, you can run Kubernetes clusters in multiple AWS regions and route traffic between them:

```
                   ┌──────────────────┐
     Users ────────►   Amazon Route53  │
                   │  (health checks  │
                   │  + DNS failover) │
                   └─────────┬────────┘
                             │
               ┌─────────────┴──────────────┐
               │                            │
               ▼                            ▼
    ┌──────────────────┐        ┌──────────────────┐
    │   EKS Cluster    │        │   EKS Cluster    │
    │   us-east-1      │        │   us-west-2      │
    │   (Primary)      │        │   (Standby/DR)   │
    └──────────────────┘        └──────────────────┘
```

Route 53 performs health checks on both clusters. If `us-east-1` fails, Route 53 automatically routes all traffic to `us-west-2`. Users experience minimal disruption.

Each cluster has its own:
- EKS cluster
- VPC and subnets
- RDS database (with cross-region read replica for failover)
- ECR images (ECR replication configured)

---

## When NOT to Use Kubernetes

Kubernetes adds operational overhead. It is the wrong choice when:

| Situation | Better Alternative |
|-----------|-------------------|
| Simple monolith app, small team | AWS ECS, App Runner, or Elastic Beanstalk |
| Serverless API with variable traffic | AWS Lambda + API Gateway |
| One service, one environment | Docker Compose on a single EC2 instance |
| Team has no Kubernetes experience | The learning curve cost outweighs the benefits |
| Budget is extremely tight | ECS Fargate is cheaper and simpler for small workloads |

The rule of thumb: if you have more than 10 containers, multiple environments, and auto-scaling needs — Kubernetes is the right tool. Below that threshold, simpler options are usually better.

---

## Summary

- **Microservices** — Kubernetes is the standard platform for running dozens of independent services, with service discovery via DNS and independent scaling per service
- **High-traffic web apps** — HPA and Cluster Autoscaler provide automatic scaling from a few Pods to hundreds, then back down — you only pay for what you use
- **Batch processing** — Jobs and CronJobs run workloads to completion and stop, ideal for data pipelines and scheduled tasks with AWS Fargate
- **Multi-tenant SaaS** — Namespaces, NetworkPolicies, and ResourceQuotas isolate tenants from each other on shared infrastructure
- **Zero-downtime deployments** — Rolling updates with readiness probes ensure users never hit a Pod that is not ready; instant rollback with `kubectl rollout undo`
- **ML and GPU workloads** — Node selectors and taints keep expensive GPU nodes reserved for GPU workloads on EKS
- **Multi-region DR** — Multiple EKS clusters with Route 53 failover provide high availability across AWS regions
- Not every application needs Kubernetes — simpler workloads are better served by ECS, Lambda, or App Runner

This completes Chapter 1 — What is Kubernetes. You now have a solid foundation: what Kubernetes is, how it compares to Docker Swarm, where it fits in the DevOps pipeline, and the real-world scenarios where it delivers value.

In Chapter 2, `01-cluster-architecture.md`, you will go inside the Kubernetes cluster — understanding the control plane, worker nodes, and every component that makes the cluster work.
