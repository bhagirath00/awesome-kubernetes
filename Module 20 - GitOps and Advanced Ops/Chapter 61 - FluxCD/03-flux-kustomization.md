# Flux Kustomization Controller & Declarative Overlays

In the Flux CD ecosystem, **`Kustomization`** refers to both the native Kubernetes Kustomize tool (`kustomize.config.k8s.io`) and the custom resource defined by Flux (`kustomize.toolkit.fluxcd.io`). The Flux Kustomize Controller evaluates these manifests, builds the directory tree, performs variable substitutions, decrypts encrypted secrets, and continuously reconciles the resulting state into the cluster.

---

## 1. Kustomize Tool vs. Flux Kustomization CRD

Understanding the boundary between the two concepts prevents configuration drift and pipeline errors:

| Feature | `kustomize.config.k8s.io` (Native) | `kustomize.toolkit.fluxcd.io` (Flux CRD) |
| :--- | :--- | :--- |
| **Execution Context** | Client-side via `kubectl kustomize` | In-cluster reconciliation loop via Controller |
| **Source of Truth** | Local directory path | `GitRepository`, `OCIRepository`, or `Bucket` CR |
| **Pruning / Garbage Collection** | None (manual `kubectl delete`) | Automatic deletion of orphaned resources |
| **Secret Decryption** | Requires client plugins/wrapper scripts | Native in-controller SOPS engine (AWS KMS, Age, Vault) |
| **Dependency Graphs** | Flat execution | Ordered pipelines using `dependsOn` |

```
  +-------------------------------------------------------------------------+
  |                           Git Repository                                |
  |   apps/production/kustomization.yaml (Native Kustomize Overlay)         |
  +------------------------------------+------------------------------------+
                                       |
                                       v
  +------------------------------------+------------------------------------+
  |              Flux Kustomization Resource (CRD)                          |
  |  - sourceRef: GitRepository/fleet-infra                                 |
  |  - path: ./apps/production                                              |
  |  - prune: true                                                          |
  |  - dependsOn: [infrastructure]                                          |
  |  - decryption: provider: sops                                           |
  +------------------------------------+------------------------------------+
                                       |
                   Reconciles every 10m / Webhook Trigger
                                       v
  +-------------------------------------------------------------------------+
  |                        Target Kubernetes Cluster                        |
  |  Applied: Deployments, Services, ConfigMaps, Ingresses                  |
  +-------------------------------------------------------------------------+
```

---

## 2. Declarative Kustomization Specification

A production-grade Flux Kustomization must explicitly configure validation, garbage collection (`prune: true`), timeouts, and target namespaces.

### Production Example: `clusters/production/apps.yaml`

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps-production
  namespace: flux-system
spec:
  # Target Source Reference
  sourceRef:
    kind: GitRepository
    name: fleet-infra
    namespace: flux-system

  # Relative path inside the Git repository
  path: ./apps/production

  # Reconciliation interval and drift detection
  interval: 10m0s
  retryInterval: 1m0s
  timeout: 5m0s

  # Automatically remove resources deleted from Git
  prune: true

  # Force server-side apply for conflict management and CRD validation
  wait: true
  force: false

  # Dependency Sequencing: Wait until infrastructure finishes reconciling
  dependsOn:
    - name: infra-production

  # Target namespace override (optional)
  targetNamespace: payment-system

  # Health checking: wait for workloads to be ready before marking sync successful
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: payment-api
      namespace: payment-system
    - apiVersion: v1
      kind: Service
      name: payment-api
      namespace: payment-system
```

---

## 3. Dependency Graphing with `dependsOn`

A common issue in GitOps is race conditions—such as deploying a Custom Resource before its Operator's CRD has finished installing, or deploying a pod before its Namespace exists.

Flux provides deterministic ordering through `dependsOn`:

```yaml
# 1. Namespaces and Global RBAC
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: platform-prerequisites
  namespace: flux-system
spec:
  interval: 15m
  prune: true
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./platform/prerequisites
---
# 2. Operators & Custom Resource Definitions
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cert-manager-operator
  namespace: flux-system
spec:
  interval: 15m
  prune: true
  dependsOn:
    - name: platform-prerequisites
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./platform/cert-manager
---
# 3. Workloads consuming CRDs (Certificates, Ingresses)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: customer-portal
  namespace: flux-system
spec:
  interval: 10m
  prune: true
  dependsOn:
    - name: cert-manager-operator
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./apps/customer-portal
```

---

## 4. In-Cluster Secret Decryption via Mozilla SOPS

Storing raw Kubernetes secrets in Git breaches zero-trust compliance. Flux natively integrates with **Mozilla SOPS** to decrypt encrypted secrets on the fly without storing decrypted plaintexts to disk.

### Step 1: Encrypting the Secret with SOPS (Workstation)
Using **Age** encryption (or AWS KMS / GCP KMS):

```bash
# Generate age key pair
age-keygen -o age.agekey
# Public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p

# Create K8s secret containing private key in flux-system
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=age.agekey

# Encrypt local secret manifest
sops --age=age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p \
  --encrypt \
  --encrypted-regex '^(data|stringData)$' \
  database-credentials.yaml > database-credentials.enc.yaml
```

### Step 2: Flux Kustomization Decryption Configuration

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: payment-secrets
  namespace: flux-system
spec:
  interval: 10m
  prune: true
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./apps/production/secrets

  # Native SOPS Engine Configuration
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

When Flux builds the manifests from Git, it decrypts `database-credentials.enc.yaml` in memory and applies the plain Secret directly to the Kubernetes API server over TLS.

---

## 5. Variable Substitution (Post-Build)

Flux supports dynamic environment variable injection at runtime, allowing you to reuse the exact same base YAML across dev, staging, and production environments without generating duplicate boilerplate:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: payment-service
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./apps/base/payment
  prune: true
  postBuild:
    substitute:
      ENVIRONMENT: "production"
      CLUSTER_DOMAIN: "prod.internal.company.com"
      LOG_LEVEL: "warn"
    substituteFrom:
      - kind: ConfigMap
        name: cluster-environment-vars
```

Manifests can reference variables using standard bash syntax: `${ENVIRONMENT}` or `${CLUSTER_DOMAIN}`.

---

## 6. Inspection, Drift Detection, and Runbook

### Inspecting Kustomizations

```bash
# Check status of all Kustomizations
flux get kustomizations

# Trace detail, conditions, and commit SHAs
flux export kustomization apps-production

# Force immediate reconcile ignoring interval
flux reconcile kustomization apps-production --with-source
```

### Common Failure Modes & Resolution

| Error Condition | Root Cause | Remediation |
| :--- | :--- | :--- |
| `dependency 'infra' is not ready` | Parent Kustomization is degraded or stalled | Check parent conditions: `flux get kustomization infra` and fix underlying failures. |
| `decryption failed: no matching keys found` | Target secretRef missing or age key mismatch | Ensure `sops-age` secret exists in `flux-system` and contains the matching private key. |
| `prune failed: cannot delete namespace` | Protected resources or stuck finalizers | Remove finalizers or check `prune: false` for persistent namespaces. |
| `kustomize build failed: accumulated resources should not be empty` | Missing `kustomization.yaml` or bad relative paths | Validate folder structure using `kustomize build ./path` locally before pushing. |
