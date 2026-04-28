# What is FluxCD: The GitOps Toolkit Architecture

## 1. Architectural Concept & Deep Dive

**FluxCD** (commonly referred to as Flux v2) is a CNCF graduated set of continuous and progressive delivery solutions for Kubernetes. Originally authored by Weaveworks, Flux was the very first tool built to demonstrate the GitOps methodology.

While ArgoCD provides a monolithic control plane with an integrated Web UI and dedicated API server, **Flux is architected as the "GitOps Toolkit"**—a collection of independent, specialized, composable Kubernetes operators built natively with the Kubernetes `controller-runtime`.

In Flux, there is no proprietary API server, no custom web database, and no custom user accounts. **Everything is a native Kubernetes Custom Resource Definition (CRD)** inspected and managed directly through `kubectl`.

```
+─────────────────────────────────────────────────────────────────────────+
|                        FLUX V2 CONTROLLER TOOLKIT                       |
|                                                                         |
|                [ Source Controller (source-controller) ]                |
|                Pulls from: GitRepository, OCIRepository, HelmRepository |
|                                   │                                     |
|                                   ▼ (Emits tar.gz Artifact)             |
|         ┌─────────────────────────┴─────────────────────────┐           |
|         ▼                                                   ▼           |
|  [ Kustomize Controller ]                          [ Helm Controller ]  |
|  • Reconciles Kustomization CRDs                   • Reconciles         |
|  • Applies raw YAML & overlays                       HelmRelease CRDs   |
|         │                                                   │           |
|         └─────────────────────────┬─────────────────────────┘           |
|                                   ▼                                     |
|                       [ Target Kubernetes Cluster ]                     |
|                                   │                                     |
|         ┌─────────────────────────┴─────────────────────────┐           |
|         ▼                                                   ▼           |
|  [ Notification Controller ]                       [ Image Automation ] |
|  • Webhooks (Inbound GitHub)                       • Scans Registries   |
|  • Alerts (Outbound Slack/Teams)                   • Writes commits Git |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Core Controllers of the GitOps Toolkit

Flux decomposes GitOps into modular, independently scalable controllers:

### 1. `source-controller`
The acquisition engine:
- Connects to external artifact sources: **Git repositories** (GitHub, GitLab), **Helm repositories**, **OCI artifacts**, or **S3/GCS buckets**.
- Packages the latest commit into an in-memory gzipped tarball served locally over HTTP inside the cluster.

### 2. `kustomize-controller`
The declarative overlay engine:
- Consumes artifacts from the `source-controller`.
- Runs Kustomize builds and applies the resulting Kubernetes manifests to the cluster.
- Handles automated pruning, garbage collection, and health checks.

### 3. `helm-controller`
The package management operator:
- Reconciles declarative **`HelmRelease`** CRDs.
- Executes automated chart installations, seamless upgrades, and instant rollbacks upon failure without requiring the Helm CLI.

### 4. `notification-controller`
The two-way communications broker:
- **Inbound Receiver**: Accepts webhooks from GitHub/GitLab to trigger instant reconciliation (eliminates polling lag).
- **Outbound Provider**: Forwards status events, sync failures, and warnings to Slack, Microsoft Teams, Discord, Datadog, or Grafana.

### 5. `image-reflector-controller` & `image-automation-controller`
The automated CI feedback loop:
- Watches OCI container registries for new semver tags.
- Automatically commits and pushes updated image tags back to the Git repository.

---

## 3. High-Level CRD Workflow

A standard Flux deployment connects a `Source` to a `Reconciler`:

```yaml
# 1. THE SOURCE: Where to fetch manifests
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: fleet-infra
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/fleet-infra
  ref:
    branch: main
---
# 2. THE RECONCILER: What to apply from that source
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: production-apps
  namespace: flux-system
spec:
  interval: 10m
  targetNamespace: production
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./clusters/production
  prune: true
```

---

## 4. Key Strengths of FluxCD

1. **Kubernetes Native**: Zero external dependencies. Operates entirely via standard Kubernetes manifests, events, and RBAC.
2. **Multi-Tenancy via ServiceAccount Impersonation**: Flux reconciles resources using the specific `ServiceAccount` assigned to a tenant's Kustomization, enforcing standard Kubernetes RBAC barriers between teams.
3. **OCI-First Distribution**: Flux natively supports fetching Kubernetes manifests packaged as OCI artifacts directly from Docker Hub, GitHub Packages (GHCR), or AWS ECR.
4. **Flagger Integration**: Natively pairs with Flagger to automate progressive canary and blue-green rollouts using Istio, Linkerd, or Nginx.

---

## 5. Key Takeaways
- FluxCD is an open-source, CNCF graduated GitOps toolkit designed as composable Kubernetes-native controllers.
- Eliminates custom UI/API layers, operating strictly via Kubernetes Custom Resource Definitions.
- Splits duties among Source, Kustomize, Helm, Notification, and Image Automation controllers.
- Integrates seamlessly with OCI registries and enforces multi-tenant security via ServiceAccount impersonation.
