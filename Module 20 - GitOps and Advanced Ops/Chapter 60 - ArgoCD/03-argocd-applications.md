# The ArgoCD Application CRD: Declarative Delivery

## 1. Architectural Concept & Deep Dive

In ArgoCD, the core abstraction that links a Git repository to a target Kubernetes cluster is the **`Application` Custom Resource Definition (CRD)** (`argoproj.io/v1alpha1`).

The `Application` manifest defines four fundamental parameters:
1. **Source**: *Where does the desired state live?* (Git repo URL, branch/tag/revision, and directory path).
2. **Destination**: *Where should the resources be deployed?* (Target Kubernetes cluster API URL and namespace).
3. **Project**: *What administrative boundary governs this app?* (`AppProject` controlling allowed repositories, destinations, and RBAC).
4. **Sync Policy**: *How should changes be applied?* (Manual vs Automated, Pruning, Self-Healing).

```
+─────────────────────────────────────────────────────────────────────────+
|                       ARGOCD APPLICATION CRD MAP                        |
|                                                                         |
|  apiVersion: argoproj.io/v1alpha1                                       |
|  kind: Application                                                      |
|  metadata:                                                              |
|    name: payment-microservice                                           |
|                                                                         |
|  spec:                                                                  |
|    project: default                                                     |
|                                                                         |
|    source:  [ Where is it in Git? ]                                     |
|      repoURL: https://github.com/myorg/gitops.git                       |
|      targetRevision: main (or v1.4.0)                                   |
|      path: apps/payment/overlays/production                             |
|                                                                         |
|    destination:  [ Where does it go? ]                                  |
|      server: https://kubernetes.default.svc (or remote EKS ARN)         |
|      namespace: payment-prod                                            |
|                                                                         |
|    syncPolicy:  [ How is it synchronized? ]                             |
|      automated:                                                         |
|        prune: true                                                      |
|        selfHeal: true                                                   |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Declarative Application Manifests

### 1. Kustomize-Based Application
Deploying an application structured with Kustomize overlays:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-api-production
  namespace: argocd
  # Finalizer enables cascading deletion of live resources when App CR is deleted!
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/ecommerce-gitops.git
    targetRevision: HEAD
    path: apps/order-api/overlays/production
    kustomize:
      namePrefix: prod-
      images:
        - "myorg/order-api:v2.4.1"
  destination:
    server: https://kubernetes.default.svc
    namespace: production-orders
```

### 2. Helm-Based Application (Using Git Repository)
Deploying an internal Helm chart stored in a Git repository with custom `values.yaml` overrides:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: billing-api-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/helm-charts.git
    targetRevision: main
    path: charts/billing-api
    helm:
      valueFiles:
        - values.yaml
        - values-production.yaml
      parameters:
        - name: "replicaCount"
          value: "5"
        - name: "ingress.enabled"
          value: "true"
  destination:
    server: https://kubernetes.default.svc
    namespace: billing
```

### 3. Helm-Based Application (Using External OCI / Helm Registry)
Deploying third-party packages directly from an OCI registry or Helm repo:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx
  namespace: argocd
spec:
  project: default
  source:
    chart: ingress-nginx
    repoURL: https://kubernetes.github.io/ingress-nginx
    targetRevision: 4.8.3
    helm:
      values: |
        controller:
          replicaCount: 3
          metrics:
            enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
```

---

## 3. Deploying to Remote / Multi-Cluster Targets

ArgoCD can deploy applications to external, remote Kubernetes clusters.

### Step 1: Register External Cluster in ArgoCD
Use the CLI to generate a ServiceAccount and kubeconfig secret in the target cluster:

```bash
# Ensure local kubectl context is pointing to target cluster (e.g., EKS secondary)
kubectl config use-context my-remote-eks-cluster

# Add cluster to ArgoCD
argocd cluster add my-remote-eks-cluster --name eks-production-us-west-2
```

### Step 2: Reference Cluster Name in Application Manifest
Instead of referencing internal API URLs, reference the registered cluster name:

```yaml
  destination:
    name: eks-production-us-west-2
    namespace: payments
```

---

## 4. Managing Deletions: Cascading vs Non-Cascading

What happens when an operator runs `kubectl delete -f application.yaml`?

- **Default (Non-Cascading)**: ArgoCD deletes the `Application` CRD from etcd, but **leaves all live workloads running on the cluster untouched**.
- **Cascading Deletion (Finalizer)**: If `finalizers: [resources-finalizer.argocd.argoproj.io]` is defined, ArgoCD **automatically deletes all live Deployments, Services, and Pods** associated with that application before removing the CRD.

---

## 5. Verification & Troubleshooting Runbook

### Managing Applications via CLI
```bash
# List all deployed applications and their health/sync state
argocd app list

# Get detailed status of a specific application
argocd app get order-api-production

# Trigger manual synchronization
argocd app sync order-api-production

# View real-time visual resource tree in terminal
argocd app get order-api-production --show-params
```

### Common Pitfalls & Solutions
1. **Application stuck in `ComparisonError`**:
   - Cause: Target directory in Git contains broken YAML syntax, or required Helm sub-chart failed to download.
   - Solution: Run `argocd app get <NAME>`. Look at the error message indicating the exact file and line number where template rendering failed.
2. **`Destination namespace does not exist` error**:
   - Cause: The target namespace has not been created on the cluster.
   - Solution: Add `syncOptions: [CreateNamespace=true]` under `spec.syncPolicy`.

---

## 6. Key Takeaways
- The `Application` CRD binds a Git Source (repo/path/tag) to a Kubernetes Destination (cluster/namespace).
- Natively supports Kustomize overlays, local Git Helm charts, and external OCI Helm registries.
- Supports multi-cluster management by targeting registered cluster names.
- Always include `resources-finalizer` when cascading cleanup of live resources is desired.
