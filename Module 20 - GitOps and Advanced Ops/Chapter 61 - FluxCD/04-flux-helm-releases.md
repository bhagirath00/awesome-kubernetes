# Managing Helm Releases with Flux CD

Helm is the de-facto package manager for Kubernetes. However, manually running `helm install` or wrapping Helm in CI pipelines causes configuration drift, missing state locks, and lack of rollback traceability.

Flux solves this via dedicated GitOps controllers:
1. **Source Controller**: Watches and packages charts from HTTP repositories, OCI registries, or Git repositories.
2. **Helm Controller**: Manages the Helm lifecycle (install, upgrade, test, rollback, and uninstall) directly against the Kubernetes API.

---

## 1. Flux Helm Controller Architecture

```
   [Helm Repo / OCI Registry]              [Git Configuration Repo]
      (e.g., bitnami / ghcr)             (HelmRelease & HelmRepository YAMLs)
                 |                                      |
                 v                                      v
  +--------------------------------------------------------------------------+
  |                             Source Controller                            |
  |  - Tracks HelmRepository / GitRepository                                 |
  |  - Fetches & verifies chart archives (.tgz)                              |
  |  - Creates HelmChart Custom Resources                                    |
  +-------------------------------------+------------------------------------+
                                        | Artifact Ready
                                        v
  +--------------------------------------------------------------------------+
  |                              Helm Controller                             |
  |  - Evaluates HelmRelease spec                                            |
  |  - Merges values (spec.values, ConfigMaps, Secrets)                      |
  |  - Performs atomic helm install / upgrade                                |
  |  - Executes Helm tests & triggers automatic rollbacks on failure         |
  +-------------------------------------+------------------------------------+
                                        | Reconciles
                                        v
  +--------------------------------------------------------------------------+
  |                        Kubernetes Cluster Resources                      |
  |             Deployments, Services, ConfigMaps, Ingresses                 |
  +--------------------------------------------------------------------------+
```

---

## 2. Declaring Chart Sources: `HelmRepository`

Before deploying a chart, declare its upstream location. Flux supports traditional HTTP/S chart repositories and modern OCI registries.

### Standard HTTP Repository
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: ingress-nginx
  namespace: flux-system
spec:
  interval: 2h
  url: https://kubernetes.github.io/ingress-nginx
  timeout: 3m
```

### OCI Registry Repository (e.g., GitHub Packages, AWS ECR, Harbor)
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: bitnami-oci
  namespace: flux-system
spec:
  interval: 1h
  type: oci
  url: oci://registry-1.docker.io/bitnamicharts
  # Optional: Authentication Secret for private OCI registries
  # secretRef:
  #   name: oci-registry-credentials
```

---

## 3. The `HelmRelease` Custom Resource

The `HelmRelease` resource defines the desired chart, semantic version range, target deployment namespace, inline values overrides, and external config references.

### Production Example: Ingress NGINX with Automated Rollback

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: ingress-system
spec:
  interval: 15m
  chart:
    spec:
      chart: ingress-nginx
      version: "4.10.x"       # Supports SemVer ranges (e.g., >=4.10.0 <4.11.0)
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
        namespace: flux-system
      interval: 1h

  # Target namespace inside the cluster
  targetNamespace: ingress-system
  install:
    createNamespace: true
    remediation:
      retries: 3
  
  # Upgrades & Failure Remediation
  upgrade:
    remediation:
      retries: 3
      remediateLastFailure: true
    cleanupOnFail: true

  # Rollback configuration
  rollback:
    timeout: 5m
    recreate: true
    disableHooks: false

  # Run Helm tests after install/upgrade
  test:
    enable: true
    ignoreFailures: false

  # Values Configuration (Merged hierarchically)
  values:
    controller:
      replicaCount: 3
      minAvailable: 2
      resources:
        requests:
          cpu: 250m
          memory: 512Mi
        limits:
          cpu: 1000m
          memory: 1Gi
      metrics:
        enabled: true
        serviceMonitor:
          enabled: true
      service:
        annotations:
          service.beta.kubernetes.io/aws-load-balancer-type: "external"
          service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "instance"
          service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
```

---

## 4. Merging Dynamic Values from Secrets & ConfigMaps

Hardcoding sensitive parameters (e.g., database passwords or API keys) into the `values` block is dangerous. Flux allows loading values dynamically from external ConfigMaps and Secrets using `valuesFrom`:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: redis-cache
  namespace: database
spec:
  interval: 10m
  chart:
    spec:
      chart: redis
      version: "18.x"
      sourceRef:
        kind: HelmRepository
        name: bitnami-oci
        namespace: flux-system
  values:
    architecture: replication
    sentinel:
      enabled: true
  valuesFrom:
    # 1. Base configuration from ConfigMap
    - kind: ConfigMap
      name: redis-common-config
      valuesKey: base-values.yaml
    
    # 2. Encrypted credentials from SOPS-decrypted Secret
    - kind: Secret
      name: redis-passwords
      valuesKey: sensitive-values.yaml
      optional: false
```

*Values precedence:* Values are evaluated in order; items declared lower in the `valuesFrom` array or inside `spec.values` override earlier keys.

---

## 5. Automated Patching with Post-Renderers

Flux allows using native Kustomize as a Helm **post-renderer**. This lets you patch third-party Helm charts without maintaining a local fork:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cert-manager
  namespace: cert-manager
spec:
  interval: 30m
  chart:
    spec:
      chart: cert-manager
      version: "v1.14.4"
      sourceRef:
        kind: HelmRepository
        name: jetstack
        namespace: flux-system
  postRenderers:
    - kustomize:
        patches:
          # Inject nodeSelector or security context into upstream Deployment
          - patch: |
              apiVersion: apps/v1
              kind: Deployment
              metadata:
                name: cert-manager
              spec:
                template:
                  spec:
                    nodeSelector:
                      topology.kubernetes.io/zone: us-east-1a
            target:
              kind: Deployment
              name: cert-manager
```

---

## 6. Verification and Troubleshooting Runbook

### Health & Status Commands

```bash
# List all Helm repositories and check sync state
flux get sources helm

# List all Helm releases, revisions, and status
flux get helmreleases -A

# View detailed release history and conditions
flux export helmrelease ingress-nginx -n ingress-system

# Force immediate reconciliation of a HelmRelease
flux reconcile helmrelease ingress-nginx -n ingress-system --with-source
```

### Resolving Stuck Helm Releases

When a release fails hooks or gets stuck in `pending-install` or `pending-upgrade`, Helm creates a persistent state lock in its Secret backend.

```bash
# 1. Identify the stuck Helm release secrets
kubectl get secrets -n ingress-system -l owner=helm

# 2. Suspend Flux reconciliation to prevent race conditions
flux suspend helmrelease ingress-nginx -n ingress-system

# 3. Use the Helm CLI or Flux to reset the state
helm rollback ingress-nginx 1 -n ingress-system
# OR delete the dangling pending secret:
# kubectl delete secret sh.helm.release.v1.ingress-nginx.v2 -n ingress-system

# 4. Resume Flux reconciliation
flux resume helmrelease ingress-nginx -n ingress-system
flux reconcile helmrelease ingress-nginx -n ingress-system
```
