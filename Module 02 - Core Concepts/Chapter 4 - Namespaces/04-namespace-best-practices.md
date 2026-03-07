# Namespace Best Practices

## Introduction

Namespaces are simple to create but easy to misuse. Bad namespace organization leads to security gaps, difficult access control, and cluttered clusters that are hard to understand and audit. Good namespace organization gives you clarity, security, and scalability as your team and cluster grow.

This file covers proven patterns that real DevOps teams use for organizing namespaces in production Kubernetes clusters on AWS.

---

## Best Practice 1: Never Use the Default Namespace for Production Workloads

The `default` namespace has no resource quotas, no access controls, and no network policies by default. Anything deployed there has no guardrails.

```bash
# What NOT to do:
kubectl apply -f my-critical-production-app.yaml
# If no namespace is specified, it lands in "default"

# Always specify a namespace:
kubectl apply -f my-critical-production-app.yaml -n production
```

Treat the `default` namespace like a scratch pad — only for quick tests you intend to delete immediately. Never run production, staging, or even long-lived development workloads in `default`.

---

## Best Practice 2: Use One Namespace Per Environment

The simplest and most common pattern separates environments into namespaces:

```
Cluster: my-company-eks

├── production       ← live traffic, strictest controls
├── staging          ← pre-production testing
├── dev              ← development and feature branches
├── monitoring       ← Prometheus, Grafana, Alertmanager
├── logging          ← Fluentd, Elasticsearch, Kibana
├── ingress-nginx    ← Ingress controller
└── kube-system      ← Kubernetes system components
```

This pattern works well for small to medium teams (1–20 engineers) with a small number of applications (1–10 services).

### Naming Convention

Use a consistent, clear naming convention:

```bash
# Good: clear, lowercase, hyphens
production
staging
dev
monitoring
logging
ingress-nginx

# Bad: unclear or inconsistent
prod          # ambiguous short form
Staging       # capitalization is inconsistent
dev_environment   # underscores (Kubernetes names use hyphens)
ns1           # meaningless
```

---

## Best Practice 3: Use Per-Team or Per-Service Namespaces for Large Organizations

When you have many teams and many services, environment-only namespaces get crowded. A better pattern at scale:

```
Cluster: my-company-eks

├── platform-prod         ← Platform team, production
├── platform-staging      ← Platform team, staging
├── payments-prod         ← Payments team, production
├── payments-staging      ← Payments team, staging
├── catalog-prod          ← Catalog team, production
├── catalog-staging       ← Catalog team, staging
├── shared-services-prod  ← Shared infrastructure (message queues, etc.)
└── infrastructure        ← Cluster-wide infra (monitoring, logging, ingress)
```

**Pattern:** `{team}-{environment}`

Each team owns their namespaces completely. The `platform` team controls access to `platform-prod` and `platform-staging`. The `payments` team controls `payments-prod` and `payments-staging`. Teams cannot accidentally affect each other.

---

## Best Practice 4: Label Every Namespace Consistently

Labels on namespaces are used by NetworkPolicies, cost tracking tools, and monitoring systems:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments-production
  labels:
    # Standard Kubernetes labels
    kubernetes.io/metadata.name: payments-production   # auto-set in newer K8s versions

    # Environment and team (for RBAC and NetworkPolicy selection)
    environment: production
    team: payments

    # Cost allocation labels (synced to AWS cost tags via tools like Kubecost)
    cost-center: "CC-2001"
    project: "payments-platform"

    # Compliance labels
    data-classification: "pci"      # PCI-scoped namespace
    compliance: "required"

  annotations:
    contact-email: "payments-team@mycompany.com"
    runbook: "https://wiki.mycompany.com/namespaces/payments"
    created-by: "terraform"
    created-date: "2024-01-15"
```

NetworkPolicies use `namespaceSelector.matchLabels` to target namespaces. If your namespaces have consistent labels, writing NetworkPolicies is straightforward.

---

## Best Practice 5: Manage Namespaces as Code

Never create namespaces manually with `kubectl create namespace`. Always define them in version-controlled YAML or Terraform:

### Option A: YAML in Git

```
infra-repo/
└── kubernetes/
    └── namespaces/
        ├── production.yaml
        ├── staging.yaml
        ├── dev.yaml
        ├── monitoring.yaml
        └── kustomization.yaml
```

```yaml
# kustomization.yaml — apply all namespaces at once
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- production.yaml
- staging.yaml
- dev.yaml
- monitoring.yaml
```

```bash
# Apply all namespaces
kubectl apply -k kubernetes/namespaces/
```

### Option B: Terraform

```hcl
# namespaces.tf — create namespaces with Terraform
resource "kubernetes_namespace" "production" {
  metadata {
    name = "production"
    labels = {
      environment     = "production"
      team            = "platform"
      cost-center     = "CC-1001"
    }
    annotations = {
      "contact" = "platform-team@mycompany.com"
    }
  }
}

resource "kubernetes_namespace" "staging" {
  metadata {
    name = "staging"
    labels = {
      environment = "staging"
      team        = "platform"
    }
  }
}
```

Using Terraform means namespace configuration is reviewed in pull requests, has audit history, and can be applied idempotently by CI/CD.

---

## Best Practice 6: Apply a Standard Baseline to Every Namespace

Every namespace (except `kube-system`) should have these four things applied automatically when created:

```bash
# Namespace setup script — called for every new namespace
setup_namespace() {
  NS=$1
  TEAM=$2

  # 1. Create namespace
  kubectl apply -f manifests/namespaces/${NS}.yaml

  # 2. Apply default limits (LimitRange)
  kubectl apply -f manifests/limits/default-limitrange.yaml -n $NS

  # 3. Apply RBAC for the team
  sed "s/TEAM_PLACEHOLDER/$TEAM/g" manifests/rbac/team-rbac-template.yaml | \
    kubectl apply -n $NS -f -

  # 4. Apply default network policies
  kubectl apply -f manifests/network-policies/default-deny.yaml -n $NS
  kubectl apply -f manifests/network-policies/allow-dns.yaml -n $NS

  echo "Namespace $NS ready for team $TEAM"
}

# Usage:
setup_namespace "payments-staging" "payments-team"
```

Tools like **Argo CD ApplicationSets** and **Helm** can automate this pattern entirely — any new namespace detected automatically gets the baseline configuration applied.

---

## Best Practice 7: Audit Namespace Contents Regularly

Over time, namespaces accumulate unused resources — old ConfigMaps, abandoned Deployments, stale Secrets. Regular auditing keeps clusters clean:

```bash
# Find namespaces with no running Pods (potentially unused)
kubectl get namespaces -o json | jq -r '.items[].metadata.name' | while read ns; do
  pod_count=$(kubectl get pods -n $ns --no-headers 2>/dev/null | wc -l)
  echo "$ns: $pod_count pods"
done

# Find old completed Jobs (safe to delete)
kubectl get jobs --all-namespaces | grep "0/1"

# Find resources in dev namespace older than 30 days
kubectl get pods -n dev --sort-by=.metadata.creationTimestamp

# Check which namespaces are close to their quota limit
kubectl get resourcequota --all-namespaces -o json | \
  jq -r '.items[] | "\(.metadata.namespace) CPU: \(.status.used["requests.cpu"])/\(.status.hard["requests.cpu"])"'
```

---

## Common Namespace Anti-Patterns to Avoid

| Anti-Pattern | Problem | Better Approach |
|-------------|---------|----------------|
| Everything in `default` | No isolation, no quotas | Create dedicated namespaces per environment |
| One namespace per microservice | Namespace sprawl — too many namespaces to manage | One namespace per team or environment |
| No resource quotas | One team can consume the whole cluster | Apply ResourceQuota to every namespace |
| No LimitRange | Pods without limits bypass ResourceQuota | Apply LimitRange to every namespace with a quota |
| Manually created namespaces | No audit trail, inconsistent setup | Manage namespaces as code in Git |
| No labels | NetworkPolicies cannot select namespaces | Label every namespace consistently from creation |
| Naming with uppercase or underscores | Kubernetes names must be lowercase and DNS-safe | Use lowercase with hyphens only |

---

## Reference: Complete Namespace Setup Template

```yaml
# Complete namespace template — copy and customize per team/environment
---
apiVersion: v1
kind: Namespace
metadata:
  name: NAMESPACE_NAME
  labels:
    environment: ENV         # production | staging | dev
    team: TEAM_NAME
    cost-center: "CC-XXXX"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: NAMESPACE_NAME
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "500m"
      memory: "256Mi"
    max:
      cpu: "4"
      memory: "4Gi"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: NAMESPACE_NAME
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    services.loadbalancers: "2"
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: NAMESPACE_NAME
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: NAMESPACE_NAME
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

---

## Summary

- Never deploy production workloads to the `default` namespace — create dedicated namespaces with quotas and access controls
- Simple teams: one namespace per environment (`production`, `staging`, `dev`). Larger organizations: `{team}-{environment}` pattern
- Use consistent lowercase, hyphen-separated naming — `payments-production` not `Payments_Prod`
- Label every namespace with environment, team, and cost-center — these labels are used by NetworkPolicies, monitoring, and cost tools
- Manage namespaces as code (YAML in Git or Terraform) — never create them manually
- Every namespace should have a baseline of: LimitRange + ResourceQuota + RBAC + NetworkPolicies (default-deny + allow-DNS)
- Avoid namespace sprawl — one namespace per microservice creates an unmanageable number of namespaces

This completes Chapter 4 — Namespaces. You now know what namespaces are, how to create and manage them, how to implement multi-layer isolation, and the best practices that keep clusters organized as they grow.

In Chapter 5, `01-what-are-labels.md`, you will learn about labels and selectors — the mechanism that connects Deployments to Pods, Services to Pods, and allows you to organize and query any resource in your cluster.
