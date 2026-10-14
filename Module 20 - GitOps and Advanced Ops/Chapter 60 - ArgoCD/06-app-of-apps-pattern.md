# The App-of-Apps Pattern & ApplicationSets

## 1. Architectural Concept & Deep Dive

When managing a Kubernetes cluster, an engineering organization typically deploys dozens of infrastructure addons (Ingress, Cert-Manager, Prometheus, Falco, Kyverno) and hundreds of microservices.

Manually running `kubectl apply -f app.yaml` for 100 individual applications defeats the core promise of GitOps.

The **App-of-Apps Pattern** solves this bootstrapping problem through recursion:
> **A single root "parent" ArgoCD Application is deployed. Its sole job is to watch a Git directory containing other ArgoCD `Application` manifests.**

When you deploy the root application, ArgoCD discovers all child `Application` manifests and deploys them automatically. Adding a new microservice or cluster addon to production requires nothing more than adding a new child YAML file to the Git repository!

```
+─────────────────────────────────────────────────────────────────────────+
|                        APP-OF-APPS RECURSIVE PATTERN                    |
|                                                                         |
|                [ ROOT PARENT APPLICATION: "cluster-root" ]              |
|                Points to: gitops-repo/bootstrap/                        |
|                                │                                        |
|         ┌──────────────────────┼──────────────────────┐                 |
|         ▼                      ▼                      ▼                 |
|  [ Child App: Ingress ] [ Child App: Monitoring ] [ Child App: Payments]|
|  Points to:             Points to:                Points to:            |
|  charts/ingress-nginx   charts/prometheus-stack   apps/payment-api      |
|         │                      │                      │                 |
|         ▼                      ▼                      ▼                 |
|  Deployed to cluster    Deployed to cluster    Deployed to cluster      |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Implementing the App-of-Apps Pattern

### Step 1: The Root Parent Application (`root-application.yaml`)
This is the **only manifest you ever apply manually** to a newly provisioned cluster:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-bootstrap-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/cluster-gitops.git
    targetRevision: main
    path: bootstrap/applications/ # Path containing all child Application YAMLs!
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Step 2: The Child Applications Directory (`bootstrap/applications/`)
Inside your Git repository at `bootstrap/applications/`, store your child application declarations:

#### Child 1: `ingress-nginx.yaml`
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
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

#### Child 2: `cert-manager.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
spec:
  project: default
  source:
    chart: cert-manager
    repoURL: https://charts.jetstack.io
    targetRevision: v1.13.2
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 3. The Modern Evolution: `ApplicationSet` Controller

While the App-of-Apps pattern is powerful, it still requires creating a separate YAML file for each application. 

The **`ApplicationSet`** controller (built into modern ArgoCD) introduces **Generators** that dynamically generate hundreds of `Application` CRDs based on repository directories, Git branches, or cluster lists.

```
+─────────────────────────────────────────────────────────────────────────+
|                       APPLICATIONSET GENERATOR FLOW                     |
|                                                                         |
|  Git Directory Generator scans: apps/*                                  |
|  Discovers: [ apps/auth, apps/billing, apps/frontend, apps/catalog ]    |
|                                │                                        |
|                                ▼                                        |
|  Automatically creates 4 independent ArgoCD Application CRDs in memory! |
+─────────────────────────────────────────────────────────────────────────+
```

### Example: Deploy All Applications in a Directory via Git Generator
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-workloads
  namespace: argocd
spec:
  generators:
    # 1. Scans Git repository for all subdirectories under apps/
    - git:
        repoURL: https://github.com/myorg/microservices.git
        revision: main
        directories:
          - path: apps/*
  template:
    metadata:
      # Interpolates the discovered folder name (e.g. auth, billing)
      name: "{{path.basename}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/microservices.git
        targetRevision: main
        path: "{{path}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{path.basename}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```
- If a developer commits a new folder `apps/checkout/`, the `ApplicationSet` controller immediately discovers it and provisions the new `checkout` application and namespace without anyone touching ArgoCD!

---

## 4. Multi-Cluster Matrix Generator

Deploy applications across multiple regional Kubernetes clusters (e.g. Staging, Prod-US, Prod-EU) using a **Matrix Generator**:

```yaml
spec:
  generators:
    - matrix:
        generators:
          # Generator A: List of target clusters
          - clusters:
              selector:
                matchLabels:
                  environment: production
          # Generator B: List of microservices
          - git:
              repoURL: https://github.com/myorg/apps.git
              revision: main
              directories:
                - path: services/*
```
- Multiplies 3 clusters $\times$ 10 services = **30 distinct `Application` instances generated automatically**!

---

## 5. Verification & Troubleshooting Runbook

### Health Verification Commands
```bash
# Verify the root application has synchronized
argocd app get root-bootstrap-app

# Inspect ApplicationSets and generated children
kubectl get applicationsets -n argocd
kubectl get applications -n argocd -l argocd.argoproj.io/application-set-name=cluster-workloads
```

### Common Pitfalls & Solutions
1. **Child application stuck in `OutOfSync` because parent has `prune: false`**:
   - Solution: Always configure `prune: true` and `selfHeal: true` on the parent root application so deletions in Git propagate downward.
2. **Infinite synchronization loops in ApplicationSets**:
   - Cause: Target directory contains non-Kubernetes files (e.g. README.md or images) causing generator parsing errors.
   - Solution: Use `exclude: true` in the git generator to skip documentation directories.

---

## 6. Key Takeaways
- The App-of-Apps pattern bootstraps entire clusters by nesting child `Application` CRDs under a single root Application.
- `ApplicationSet` elevates this pattern by dynamically generating applications from Git directories, branches, or cluster lists.
- A single `ApplicationSet` with a Matrix generator can manage hundreds of services across multi-cloud environments.
