# What is ArgoCD: Architecture & Core Components

## 1. Architectural Concept & Deep Dive

**ArgoCD** is the industry-leading, CNCF graduated declarative GitOps continuous delivery controller for Kubernetes. Created by Intuit and maintained by an extensive open-source community, ArgoCD automates the deployment of desired application states specified in Git repositories directly into Kubernetes clusters.

ArgoCD continuously monitors running applications, compares their live state against the desired state stored in Git, and detects discrepancies (**Out of Sync** / **Degraded** states). It provides both an intuitive visual Web UI and an automation engine capable of auto-reconciling drift.

```
+─────────────────────────────────────────────────────────────────────────+
|                           ARGOCD ARCHITECTURE                           |
|                                                                         |
|  [ Operators (UI / CLI) ] ──► [ Inbound Webhooks (GitHub / GitLab) ]    |
|                                     │                                   |
|                                     ▼                                   |
|                         [ ArgoCD API Server ]                           |
|                         • gRPC / REST Engine                            |
|                         • RBAC & SSO (Dex / OIDC)                       |
|                                     │                                   |
|              ┌──────────────────────┼──────────────────────┐            |
|              ▼                      ▼                      ▼            |
|   [ Repository Server ]     [ Redis Cache ]     [ App Controller ]      |
|   • Clones Git repos        • Manifest cache    • Reconciliation Loop   |
|   • Renders Helm/Kustomize  • State cache       • Compares Live vs Git  |
|              │                                             │            |
|              ▼                                             ▼            |
|   [ Rendered Manifests ] ──► Ingested into ──► [ Target K8s Clusters ]  |
|                                                • In-Cluster Local       |
|                                                • Remote Clusters (EKS)  |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Core Architectural Components Breakdown

ArgoCD is composed of five standalone, scalable microservices:

### 1. `argocd-server` (API Server)
The API server exposes the Web UI and CLI endpoints:
- Authenticates users via internal tokens, GitHub OAuth, Okta, or Google Workspace via OIDC.
- Enforces fine-grained Role-Based Access Control (RBAC).
- Receives inbound Git webhook events (`/api/webhook`) to trigger instant synchronization without waiting for polling intervals.

### 2. `argocd-repo-server` (Repository Server)
The internal compiler and rendering engine:
- Clones target Git repositories locally.
- Executes templating tools: **Helm**, **Kustomize**, **Jsonnet**, or custom Config Management Plugins (CMP).
- Converts templates into pure, raw Kubernetes YAML manifests and caches the result in Redis.

### 3. `argocd-application-controller` (Reconciliation Engine)
The core Kubernetes controller:
- Continuously polls the Kubernetes API server to inspect the live state of all managed resources.
- Compares live state with the desired state provided by the `repo-server`.
- Identifies drift, calculates health status (`Healthy`, `Progressing`, `Degraded`), and invokes self-healing sync passes if configured.

### 4. `argocd-redis`
An in-memory cache that stores rendered manifests, token sessions, and cluster state to eliminate redundant Git clones and reduce load on the Kubernetes API server.

### 5. `argocd-dex-server`
An embedded OpenID Connect (OIDC) identity broker that delegates authentication to enterprise identity providers (SAML, OAuth2, LDAP).

---

## 3. High-Level Resource States in ArgoCD

Every application managed by ArgoCD is assigned two independent health and synchronization states:

```
                  +───────────────────────────────+
                  |     Sync Status (Git vs K8s)  |
                  |     • Synced                  |
                  |     • OutOfSync (Drift detected)
                  +───────────────┬───────────────+
                                  │
                                  ▼
                  +───────────────────────────────+
                  |     Health Status (Runtime)   |
                  |     • Healthy (Pods ready)    |
                  |     • Progressing (Rollout)   |
                  |     • Degraded (CrashLoop)    |
                  |     • Missing (Not deployed)  |
                  +───────────────────────────────+
```

---

## 4. Multi-Cluster Management Capability

Unlike tools that must be installed on every single cluster, **a single ArgoCD control plane can manage hundreds of external Kubernetes clusters**:

```
[ Central Management Cluster (ArgoCD Server) ]
                     │
     ┌───────────────┼───────────────┐
     ▼               ▼               ▼
[ Cluster A: EKS ] [ Cluster B: GKE ] [ Cluster C: On-Premise ]
```
ArgoCD connects to external clusters using dedicated ServiceAccount credentials and applies manifests remotely, serving as a unified pane of glass across global multi-cloud infrastructure.

---

## 5. Key Takeaways
- ArgoCD is the leading CNCF graduated continuous delivery engine for Kubernetes GitOps.
- Its architecture splits responsibilities among the API Server, Repo Server (compiler), Application Controller (reconciliation loop), and Redis cache.
- Tracks two independent metrics: Sync Status (Git alignment) and Health Status (runtime workload stability).
- Features native multi-cluster support, allowing a centralized control plane to govern worldwide Kubernetes fleets.
