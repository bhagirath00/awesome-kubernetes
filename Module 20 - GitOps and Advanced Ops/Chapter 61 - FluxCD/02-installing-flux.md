# Installing and Bootstrapping Flux CD

Flux CD (Flux v2) is a continuous delivery tool that relies on Kubernetes Custom Resource Definitions (CRDs) and dedicated controllers to reconcile cluster state against declarative sources (Git repositories, OCI artifacts, S3 buckets). 

Unlike legacy tools that rely on external CI push runners, Flux uses a **Bootstrap** model: the Flux CLI generates its own controller manifests, commits them to your Git repository, and reconciles itself into the target Kubernetes cluster.

---

## 1. Flux Architecture & Pre-Flight Verification

Before installing Flux, verify that your Kubernetes cluster meets the minimum API requirements and controller capabilities.

```
       +-------------------------------------------------------+
       |                  Developer Workstation                |
       |  1. Generate manifests & tokens                       |
       |  2. Execute: flux bootstrap github ...                |
       +--------------------------+----------------------------+
                                  |
              +-------------------+-------------------+
              |                                       |
              v (Pushes Flux manifests)               v (Applies bootstrap CRDs)
    +-------------------+                   +--------------------+
    | Remote Git Repo   |                   | Kubernetes Cluster |
    | (e.g., GitHub)    |                   | (Namespace:        |
    |                   |                   |  flux-system)      |
    |  clusters/prod/   | <================ |                    |
    |   flux-system/    |  Pulls & syncs    |  - source-ctrl     |
    |    gotk-sync.yaml |  cluster state    |  - kustomize-ctrl  |
    |    gotk-comp.yaml |                   |  - helm-ctrl       |
    +-------------------+                   |  - notification-c. |
                                            +--------------------+
```

### Pre-requisites Check
Run the pre-installation check using the Flux CLI to validate Kubernetes server version (>= 1.25) and client connectivity:

```bash
flux check --pre
```

**Expected output:**
```
► checking prerequisites
✔ Kubernetes 1.29.2 >=1.25.0-0
✔ prerequisites checks passed
```

---

## 2. Installing the Flux CLI

The `flux` binary manages controller lifecycles, inspects reconciliation statuses, generates CRD manifests, and forces manual syncs.

### Linux / macOS (Homebrew or Script)
```bash
# macOS / Linux via Homebrew
brew install fluxcd/tap/flux

# Direct Bash install script
curl -s https://fluxcd.io/install.sh | sudo bash

# Enable shell auto-completion (Bash)
. <(flux completion bash)
```

### Windows (Winget or Chocolatey)
```powershell
# Using Winget
winget install FluxCD.Flux

# Using Chocolatey
choco install flux
```

### Verify CLI Installation
```bash
flux --version
```

---

## 3. Flux Bootstrapping

The bootstrap command is idempotent. It performs three critical operations:
1. Creates a Git repository if it doesn't already exist.
2. Commits the Flux toolkit components (`gotk-components.yaml`) and sync configurations (`gotk-sync.yaml`) to the chosen branch.
3. Applies the manifests to the cluster and sets up deploy keys or personal access tokens (PAT).

### Option A: GitHub Bootstrap
Generate a GitHub Personal Access Token (classic) with `repo` permissions (and `admin:org` if bootstrapping an organization repo).

```bash
export GITHUB_TOKEN="ghp_YourPersonalAccessTokenGoesHere"
export GITHUB_USER="your-github-username"

flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/production \
  --personal \
  --private=true
```

### Option B: GitLab Bootstrap
Generate a GitLab Personal Access Token with `api`, `read_repository`, and `write_repository` scopes:

```bash
export GITLAB_TOKEN="glpat-YourGitLabTokenGoesHere"

flux bootstrap gitlab \
  --owner=your-gitlab-group \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/production \
  --personal=false
```

### Option C: Generic Git Repository (SSH Deploy Key)
For self-hosted Git servers (Gitea, Bitbucket Server, Azure DevOps):

```bash
flux bootstrap git \
  --url=ssh://git@git.company.internal/devops/fleet-infra.git \
  --branch=main \
  --path=clusters/production \
  --private-key-file=$HOME/.ssh/id_ed25519
```

---

## 4. Production Directory Layout

A standard multi-tenant, multi-environment Flux structure isolates cluster configurations, reusable base manifests, and target environments:

```
fleet-infra/
├── clusters/
│   ├── staging/
│   │   ├── flux-system/          # Managed by Flux bootstrap
│   │   ├── infrastructure.yaml   # Points to infra overlay
│   │   └── apps.yaml             # Points to apps overlay
│   └── production/
│       ├── flux-system/
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       ├── infrastructure.yaml
│       └── apps.yaml
├── infrastructure/
│   ├── base/
│   │   ├── cert-manager/
│   │   ├── ingress-nginx/
│   │   └── monitoring/
│   └── production/
│       ├── kustomization.yaml
│       └── cert-manager-values.yaml
└── apps/
    ├── base/
    │   └── payment-service/
    └── production/
        ├── kustomization.yaml
        └── replicas-patch.yaml
```

---

## 5. Production Hardening & High Availability

By default, Flux controllers are deployed with single replicas and conservative resource limits. For mission-critical production clusters, customize controller deployments via the `flux-system` Kustomization overlay.

### High Availability Patch (`clusters/production/flux-system/kustomization.yaml`)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
patches:
  # Enable Multi-Replica HA for Source Controller
  - patch: |
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: source-controller
        namespace: flux-system
      spec:
        replicas: 2
        template:
          spec:
            containers:
              - name: manager
                resources:
                  limits:
                    cpu: 1000m
                    memory: 1Gi
                  requests:
                    cpu: 200m
                    memory: 256Mi
    target:
      kind: Deployment
      name: source-controller

  # Tune Concurrency for Kustomize Controller
  - patch: |
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: kustomize-controller
        namespace: flux-system
      spec:
        replicas: 2
        template:
          spec:
            containers:
              - name: manager
                args:
                  - --concurrent=10
                  - --requeue-dependency=15s
                resources:
                  limits:
                    cpu: 2000m
                    memory: 2Gi
                  requests:
                    cpu: 500m
                    memory: 512Mi
    target:
      kind: Deployment
      name: kustomize-controller
```

---

## 6. Verification and Troubleshooting Runbook

### Health Verification Commands

```bash
# Check overall Flux health and CRD compatibility
flux check

# List all synchronized Git repositories
flux get sources git

# List all Kustomizations and their sync status
flux get kustomizations

# View controller logs in real time
flux logs --level=info --all-namespaces
```

### Troubleshooting Common Bootstrap Failures

| Issue | Root Cause | Remediation Command |
| :--- | :--- | :--- |
| `fatal: could not read Username for 'https://github.com'` | Invalid or expired Git Token | Re-export `GITHUB_TOKEN` with valid `repo` scopes and rerun `flux bootstrap`. |
| `failed to clone repository: ssh: handshake failed` | Missing or mismatched deploy key | Run `flux create secret git flux-system --url=ssh://git@... --ssh-key-algorithm=ed25519` to regenerate the key. |
| `prerequisites checks failed: Kubernetes version < 1.25` | Unsupported K8s API primitives | Upgrade control plane or pin an older compatible Flux release. |
| `reconciliation timeout: context deadline exceeded` | Insufficient CPU/Memory on nodes | Inspect events: `kubectl describe deployment kustomize-controller -n flux-system`. |

### Forcing Immediate Reconciliation
To bypass the default sync polling interval (e.g. 10m) during emergencies:

```bash
flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source
```
