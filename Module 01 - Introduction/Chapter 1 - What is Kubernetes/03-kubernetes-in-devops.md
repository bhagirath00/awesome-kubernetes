# Kubernetes in DevOps

## Introduction

Kubernetes does not exist in isolation. It is one piece of a larger DevOps toolchain, and understanding exactly where it fits — and where it does not — is essential before you start working with it.

In this file, you will see how Kubernetes connects to the rest of your DevOps pipeline: source control, CI/CD, container registries, monitoring, and infrastructure as code. By the end, you will have a clear mental map of Kubernetes in a real-world DevOps workflow.

---

## The Full DevOps Pipeline — Where Kubernetes Lives

Here is a complete DevOps pipeline for a production application running on AWS:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DEVELOPER WORKSTATION                        │
│  Git commit → git push → GitHub/GitLab                             │
└────────────────────────┬────────────────────────────────────────────┘
                         │ webhook trigger
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         CI/CD PIPELINE                              │
│  Jenkins / GitHub Actions / GitLab CI                               │
│                                                                     │
│  1. Pull source code                                                │
│  2. Run unit tests                                                  │
│  3. Run security scan (Trivy, Snyk)                                 │
│  4. Build Docker image                                              │
│  5. Push image to Amazon ECR                                        │
│  6. Apply Kubernetes manifests to EKS                               │
└────────────────────────┬────────────────────────────────────────────┘
                         │ kubectl apply / helm upgrade
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      KUBERNETES CLUSTER (AWS EKS)                   │
│                                                                     │
│  ┌──────────────────┐   ┌──────────────────┐   ┌────────────────┐  │
│  │   Control Plane  │   │   Worker Node 1  │   │ Worker Node 2  │  │
│  │  (managed by AWS)│   │   Pod Pod Pod    │   │  Pod Pod Pod   │  │
│  └──────────────────┘   └──────────────────┘   └────────────────┘  │
│                                                                     │
│  Kubernetes handles:                                                │
│  - Scheduling Pods on nodes                                         │
│  - Rolling updates (zero downtime)                                  │
│  - Auto-scaling (HPA)                                               │
│  - Self-healing (restart crashed Pods)                              │
│  - Load balancing (Service + Ingress)                               │
└────────────────────────┬────────────────────────────────────────────┘
                         │ metrics, logs, traces
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY STACK                               │
│  Prometheus (metrics) → Grafana (dashboards)                        │
│  Fluentd / Fluent Bit (log forwarding) → CloudWatch / Elasticsearch │
│  AWS CloudWatch Container Insights                                  │
└─────────────────────────────────────────────────────────────────────┘
```

Kubernetes owns the middle layer — the runtime layer. Everything above it builds and delivers software. Everything below it monitors and observes it.

---

## Stage 1: Source Control (Before Kubernetes)

The pipeline starts with a developer pushing code to a Git repository — GitHub, GitLab, or Bitbucket.

Kubernetes connects to Git in a specific way: not for application code, but for **Kubernetes configuration files** (YAML manifests). In a mature DevOps setup, your Git repository contains two things:

1. **Application source code** (`src/`, `app/`, etc.)
2. **Kubernetes manifests** (`k8s/` or `kubernetes/` directory)

```
my-application/
├── src/
│   ├── main.go
│   └── handlers/
├── Dockerfile
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   └── configmap.yaml
├── Jenkinsfile
└── README.md
```

When a developer changes a Kubernetes manifest (for example, bumping the container image version), they commit that change to Git. The CI/CD pipeline picks it up and applies it to the cluster. This practice is called **GitOps** — Git is the single source of truth for both code and infrastructure state.

---

## Stage 2: CI/CD Pipeline (Jenkins + Kubernetes)

The CI/CD pipeline is where code becomes a running container in Kubernetes. Here is a complete Jenkinsfile for a Kubernetes deployment:

```groovy
// Jenkinsfile — CI/CD pipeline that builds and deploys to Kubernetes (EKS)

pipeline {
    agent any

    environment {
        AWS_REGION      = 'us-east-1'
        ECR_REGISTRY    = '123456789.dkr.ecr.us-east-1.amazonaws.com'
        IMAGE_NAME      = 'my-web-app'
        K8S_NAMESPACE   = 'production'
        CLUSTER_NAME    = 'my-eks-cluster'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                }
            }
        }

        stage('Run Tests') {
            steps {
                sh 'go test ./...'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                               ${ECR_REGISTRY}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Push to ECR') {
            steps {
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS --password-stdin ${ECR_REGISTRY}

                    docker push ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh """
                    # Update kubeconfig to point to EKS cluster
                    aws eks update-kubeconfig \
                        --region ${AWS_REGION} \
                        --name ${CLUSTER_NAME}

                    # Update the image in the deployment
                    kubectl set image deployment/web-deployment \
                        web=${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                        --namespace ${K8S_NAMESPACE}

                    # Wait for rollout to complete
                    kubectl rollout status deployment/web-deployment \
                        --namespace ${K8S_NAMESPACE} \
                        --timeout=300s
                """
            }
        }

    }

    post {
        failure {
            sh """
                # Rollback to the previous version if deployment fails
                kubectl rollout undo deployment/web-deployment \
                    --namespace ${K8S_NAMESPACE}
            """
        }
    }
}
```

This pipeline does the complete journey: test → build image → push to ECR → deploy to EKS → verify rollout → rollback on failure.

---

## Stage 3: Container Registry (Amazon ECR)

Amazon ECR (Elastic Container Registry) is the intermediary between your CI/CD pipeline and Kubernetes. The pipeline pushes Docker images to ECR. Kubernetes pulls images from ECR when starting Pods.

```
CI/CD Pipeline   →   Amazon ECR   →   Kubernetes (EKS)
  docker push         (stores)        docker pull (via kubelet)
```

Kubernetes needs permission to pull images from ECR. On EKS, this is handled via the IAM role attached to your worker nodes — as long as the worker nodes have `ecr:GetAuthorizationToken` and `ecr:BatchGetImage` permissions, they can pull from ECR without any additional configuration.

```bash
# Verify your deployment can pull the image
kubectl describe pod web-deployment-abc123 -n production

# If there is an image pull error, you will see:
# Warning  Failed  ErrImagePull: failed to pull image "123456789.dkr.ecr.us-east-1.amazonaws.com/my-web-app:abc123":
# rpc error: ... unauthorized: authentication required
```

When you see `ErrImagePull`, it is almost always an IAM permission issue on the worker node role.

---

## Stage 4: Kubernetes Deployment (The Core)

Once the image is in ECR and Kubernetes has the updated deployment, it performs a **rolling update**:

```bash
# Watch the rolling update happen in real time
kubectl rollout status deployment/web-deployment -n production

# Output as Kubernetes updates each Pod:
# Waiting for deployment "web-deployment" rollout to finish:
#   1 out of 3 new replicas have been updated...
#   2 out of 3 new replicas have been updated...
#   3 out of 3 new replicas have been updated...
#   3 old replicas are pending termination...
# deployment "web-deployment" successfully rolled out
```

During this process, traffic continues flowing to the old Pods until new Pods pass their readiness probe. Users never experience downtime.

If something goes wrong, rollback is instant:

```bash
# Rollback to the previous version
kubectl rollout undo deployment/web-deployment -n production

# Check rollout history
kubectl rollout history deployment/web-deployment -n production
# REVISION  CHANGE-CAUSE
# 1         Initial deployment
# 2         v1.2.0 release
# 3         v1.3.0 release  ← currently deployed
```

---

## Stage 5: Observability (After Kubernetes)

Kubernetes generates a large amount of data — metrics, logs, and events. The observability stack collects and visualizes this data.

### Metrics with Prometheus and Grafana

**Prometheus** scrapes metrics from Kubernetes components and your application. **Grafana** displays them as dashboards.

A common pattern on EKS is deploying the `kube-prometheus-stack` Helm chart, which installs both:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

### Logs with Fluent Bit and CloudWatch

Kubernetes Pod logs are ephemeral — when a Pod is deleted, its logs are gone. **Fluent Bit** runs as a DaemonSet (one instance per node) and forwards all Pod logs to CloudWatch Logs:

```yaml
# Fluent Bit runs on every node as a DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: amazon-cloudwatch
spec:
  selector:
    matchLabels:
      name: fluent-bit
  template:
    spec:
      containers:
      - name: fluent-bit
        image: amazon/aws-for-fluent-bit:latest
        # Mounts the node's container log directory
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

With this in place, all container logs are in CloudWatch Logs for searching, alerting, and retention.

---

## Kubernetes and Infrastructure as Code

In a mature DevOps environment, the Kubernetes cluster itself is created and managed with Infrastructure as Code. On AWS, Terraform is the standard tool:

```hcl
# Creating an EKS cluster with Terraform
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "my-production-cluster"
  cluster_version = "1.29"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    workers = {
      instance_types = ["t3.medium"]
      min_size       = 2
      max_size       = 10
      desired_size   = 3
    }
  }

  tags = {
    Environment = "production"
    ManagedBy   = "Terraform"
  }
}
```

Terraform provisions the cluster, VPC, and node groups. Kubernetes then manages everything running inside that cluster. They operate at different layers:
- **Terraform** = what infrastructure exists (cluster, nodes, VPC, IAM)
- **Kubernetes** = what runs inside that infrastructure (Pods, Services, Deployments)

---

## GitOps — The Modern Pattern

The most advanced DevOps teams use a pattern called **GitOps** with Kubernetes. Instead of Jenkins pushing to Kubernetes, a tool called **Argo CD** watches a Git repository and automatically applies any changes to the cluster:

```
Developer merges PR to main branch
         ↓
Git repository now has updated Kubernetes manifests
         ↓
Argo CD detects the change (polling or webhook)
         ↓
Argo CD applies the manifests to the cluster
         ↓
Kubernetes performs the rolling update
         ↓
Argo CD reports sync status (✅ Synced / ❌ OutOfSync)
```

The benefit is that Git becomes the complete audit trail for every change to your cluster. Want to know who changed the replica count from 3 to 10? Check Git blame.

You will work with GitOps patterns in later modules of this course.

---

## The DevOps Engineer's Kubernetes Responsibilities

When you work as a DevOps engineer at a company using Kubernetes, your day-to-day responsibilities typically include:

| Responsibility | Tools Used |
|----------------|-----------|
| **Cluster provisioning and upgrades** | Terraform, `eksctl`, AWS Console |
| **Deploying applications** | `kubectl apply`, Helm, Argo CD |
| **Writing Kubernetes manifests** | YAML, Kustomize, Helm charts |
| **Debugging failed Pods** | `kubectl logs`, `kubectl describe`, `kubectl exec` |
| **Setting up CI/CD pipelines** | Jenkins, GitHub Actions |
| **Configuring auto-scaling** | HPA, Cluster Autoscaler |
| **Setting up monitoring** | Prometheus, Grafana, CloudWatch |
| **Managing secrets** | Kubernetes Secrets, AWS Secrets Manager |
| **Network policy and security** | Network Policies, Pod Security Standards, IAM |
| **Cost optimization** | Right-sizing nodes, spot instances, resource limits |

---

## Summary

- Kubernetes sits in the middle of the DevOps pipeline — after CI/CD builds and pushes the image, before observability collects metrics and logs
- A complete pipeline on AWS: GitHub → Jenkins → Amazon ECR → EKS → CloudWatch/Prometheus
- The CI/CD pipeline uses `kubectl` or `helm` to apply changes to the Kubernetes cluster
- Amazon ECR stores Docker images; EKS worker nodes pull images from ECR using IAM permissions
- Kubernetes rolling updates ensure zero-downtime deployments; `kubectl rollout undo` provides instant rollback
- The Kubernetes cluster itself is provisioned with Terraform; what runs inside it is managed by Kubernetes YAML manifests
- GitOps (with Argo CD) is the modern pattern where Git is the source of truth for cluster state
- DevOps engineers using Kubernetes are responsible for provisioning clusters, writing manifests, debugging, CI/CD integration, monitoring, and security

In the next file, `04-kubernetes-use-cases.md`, you will explore the real-world scenarios where Kubernetes is the right choice — including microservices, batch processing, multi-tenant platforms, and machine learning workloads on AWS.
