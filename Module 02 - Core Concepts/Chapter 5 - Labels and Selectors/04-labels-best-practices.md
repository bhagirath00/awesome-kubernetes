# Labels Best Practices

## Introduction

You have learned what labels are and how selectors work. Now the practical question is: what labels should you actually put on your resources, and how should you organize them? Bad labeling leads to confusion, broken selectors, and difficulty debugging production issues. Good labeling makes your cluster self-documenting and easy to operate.

This file covers the labeling strategy used by mature DevOps teams — what to always label, how to structure labels for monitoring and cost tracking, and common mistakes to avoid.

---

## Always Apply These Labels to Every Resource

At a minimum, every Pod (and the Deployments/StatefulSets that manage them) should have these labels:

```yaml
metadata:
  labels:
    # 1. What application is this?
    app: web-server

    # 2. Which component within the application?
    component: frontend           # frontend | backend | database | cache | worker

    # 3. Which environment?
    environment: production       # production | staging | dev | test

    # 4. Who owns this?
    team: platform                # which team is responsible

    # 5. What version is running?
    version: "2.1.0"             # application version (helps with canary rollbacks)
```

These five labels give you the ability to:
- Find all Pods for a specific application (`-l app=web-server`)
- Find all production Pods for all applications (`-l environment=production`)
- Find a specific team's resources (`-l team=payments`)
- Find all Pods of a specific version during a rollback (`-l version=2.0.0`)

---

## Use the Kubernetes Recommended Labels

The official Kubernetes recommended labels (`app.kubernetes.io/` prefix) are recognized by many tools — Helm, Argo CD, Lens, and others. Use them alongside your own labels:

```yaml
metadata:
  labels:
    # Kubernetes recommended labels
    app.kubernetes.io/name: web-server           # name of the app
    app.kubernetes.io/instance: web-server-prod  # unique instance (app + environment)
    app.kubernetes.io/version: "2.1.0"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: ecommerce-platform
    app.kubernetes.io/managed-by: helm           # helm | argocd | kubectl

    # Your own labels (can coexist)
    environment: production
    team: platform
    cost-center: "CC-1001"
```

The `app.kubernetes.io/` prefix tells tools these labels follow the standard. Using both the standard labels and your own custom labels is the recommended approach.

---

## Label Strategy for Cost Tracking

Labels are the foundation of Kubernetes cost allocation. Tools like **Kubecost** and **CloudHealth** read labels to attribute cluster costs to teams, projects, and environments.

For cost tracking to work, labels must be consistent across all resources:

```yaml
# Apply these labels consistently to ALL workloads
metadata:
  labels:
    # Cost allocation
    cost-center: "CC-2001"          # maps to your finance system
    team: payments                   # which team incurs the cost
    project: checkout-service        # which project
    environment: production          # production workloads cost more

    # Business context
    business-unit: "ecommerce"
    product: "checkout"
```

The labels map to AWS Cost Allocation Tags via the Kubernetes cluster's tags. EKS can propagate Pod labels as AWS resource tags — enabling cost breakdown by team, project, and environment directly in the AWS Cost Explorer.

```bash
# See how much resources each team's Pods are consuming
kubectl top pods --all-namespaces --show-labels | grep 'team=payments'

# With Kubecost installed, get cost by label
curl http://kubecost.kube-system.svc:9090/model/allocation?window=7d&aggregate=label:team
```

---

## Label Strategy for Monitoring

Prometheus and Grafana use labels to group metrics. When your Pods have consistent labels, you get meaningful dashboards automatically:

```yaml
# These labels become metric dimensions in Prometheus
metadata:
  labels:
    app: web-server
    component: frontend
    environment: production
    version: "2.1.0"
```

In Prometheus, Pod metrics are scraped with the Pod's labels attached as metric labels:

```
# Prometheus query: P99 latency for all production web-server instances
histogram_quantile(0.99,
  sum by (le, app, component) (
    rate(http_request_duration_seconds_bucket{
      environment="production",
      app="web-server"
    }[5m])
  )
)
```

The `environment` and `app` label values come from the Pod's Kubernetes labels — not configured separately in Prometheus.

---

## Label Strategy for Deployments — The Selector Lock-In Problem

Deployment selectors are immutable. Once you create a Deployment, you cannot change its `selector.matchLabels`. This has important implications for your label design:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  selector:
    matchLabels:
      app: web-server        # ← IMMUTABLE — cannot change this after creation
      component: frontend    # ← IMMUTABLE — cannot change this after creation
```

If you ever need to change these labels (for a refactoring or renaming), you must:
1. Delete the old Deployment (with `--cascade=orphan` to not delete the Pods)
2. Create a new Deployment with the updated selector

**Best practice:** Keep selectors minimal and stable. Use only `app` and `component` in your selector. Put all other labels (`version`, `team`, `environment`) in the Pod template but NOT in the selector:

```yaml
spec:
  selector:
    matchLabels:
      app: web-server        # minimal stable selector
      component: frontend
  template:
    metadata:
      labels:
        app: web-server      # must include selector labels
        component: frontend
        version: "2.1.0"    # extra labels — fine to change between deployments
        team: platform       # extra labels — fine to change
        environment: production
```

The `version` label in the template changes with every deployment but is not in the selector — so it does not cause any conflict.

---

## Avoid These Labeling Mistakes

### Mistake 1: Putting Resource Names in Labels

```yaml
# BAD — specific to one instance, useless for grouping
labels:
  pod-name: web-pod-abc123
  hostname: ip-10-0-1-100.ec2.internal
```

These are already available through `metadata.name` and `spec.nodeName`. Do not duplicate them as labels — they add noise without value.

### Mistake 2: Using Labels for Data That Belongs in Annotations

```yaml
# BAD — git commit hash is metadata, not for selection
labels:
  git-commit: "a1b2c3d4e5f6"

# CORRECT — git commit belongs in annotations
annotations:
  git-commit: "a1b2c3d4e5f6"
```

If you will never filter resources by it (`kubectl get pods -l git-commit=abc`), it belongs in annotations, not labels.

### Mistake 3: Inconsistent Label Names

```yaml
# Team A uses:
labels:
  application: web-server
  env: prod

# Team B uses:
labels:
  app: web-server
  environment: production

# Team C uses:
labels:
  service: web-server
  deploy-env: production
```

When three teams use different label keys for the same concept, you cannot write a single Prometheus query, ResourceQuota, or NetworkPolicy that covers all of them. Standardize on one set of label names across the entire organization.

### Mistake 4: Changing Selector Labels Without Deletion

```bash
# This fails silently or causes errors:
kubectl apply -f deployment-with-new-selector.yaml

# Error: Deployment.apps "web-deployment" is invalid:
# spec.selector: Invalid value: ... field is immutable
```

Selector labels cannot be changed on existing Deployments. Plan your selector labels carefully at design time.

### Mistake 5: No Version Label

Without a version label, you cannot easily identify which Pods are running an old version during a rolling update:

```bash
# With version labels, you can see the rollout progress:
kubectl get pods -l app=web-server --show-labels

# NAME                       READY   LABELS
# web-dep-abc-xyz    1/1     app=web-server,version=2.0.0   ← old
# web-dep-abc-123    1/1     app=web-server,version=2.0.0   ← old
# web-dep-def-456    1/1     app=web-server,version=2.1.0   ← new
```

---

## Complete Label Template — Use This as Your Standard

Copy this template for every Deployment you create:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <app-name>
  namespace: <environment>
  labels:
    app.kubernetes.io/name: <app-name>
    app.kubernetes.io/instance: <app-name>-<environment>
    app.kubernetes.io/version: "<version>"
    app.kubernetes.io/component: <frontend|backend|worker|database>
    app.kubernetes.io/part-of: <platform-name>
    app.kubernetes.io/managed-by: <helm|argocd|kubectl>
    team: <team-name>
    cost-center: "<CC-XXXX>"
    environment: <production|staging|dev>
spec:
  selector:
    matchLabels:
      app: <app-name>           # keep selector minimal and stable
      component: <component>
  template:
    metadata:
      labels:
        app: <app-name>
        component: <component>
        version: "<version>"
        team: <team-name>
        environment: <production|staging|dev>
        app.kubernetes.io/name: <app-name>
        app.kubernetes.io/version: "<version>"
    spec:
      containers:
      - name: <app-name>
        image: ...
```

---

## Summary

- Every resource should have at minimum: `app`, `component`, `environment`, `team`, and `version` labels
- Use the Kubernetes recommended labels (`app.kubernetes.io/` prefix) alongside your custom labels
- Label every resource consistently for cost tracking — Kubecost and CloudHealth use labels to attribute costs to teams and projects
- Prometheus inherits Pod labels as metric dimensions — consistent labels enable powerful cross-service queries
- Keep Deployment selectors minimal and stable (`app` + `component`) — selectors are immutable after creation
- Put frequently-changing values (`version`, `git-commit`) in the Pod template labels but NOT in the selector
- Never put selection-irrelevant data (git commit, hostnames, timestamps) in labels — use annotations for that
- Standardize label keys across the entire organization — inconsistent naming breaks monitoring, policies, and cost tracking

This completes Chapter 5 — Labels and Selectors, and with it the full Module 2 — Core Concepts.

You now have a complete understanding of the three foundational pillars of Kubernetes: **Pods** (the workload unit), **Namespaces** (the organizational boundary), and **Labels and Selectors** (the connection mechanism). Everything else in Kubernetes builds on these three concepts.

In Module 3 — Networking, you will learn how Kubernetes handles network traffic: Services in depth, Ingress and HTTP routing, DNS resolution, and network policies for security.
