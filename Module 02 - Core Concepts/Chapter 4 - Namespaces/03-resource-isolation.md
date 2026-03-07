# Resource Isolation in Namespaces

## Introduction

Namespaces provide a boundary, but the boundary means nothing without enforcement. You need to configure isolation explicitly — who can access what, how much they can consume, and which network traffic is allowed. This file covers the four layers of namespace isolation: RBAC, ResourceQuotas, LimitRanges, and NetworkPolicies — and shows how they work together to create a fully isolated environment.

---

## The Four Layers of Namespace Isolation

```
┌────────────────────────────────────────────────────────────────┐
│                       NAMESPACE: production                     │
│                                                                │
│  Layer 1: RBAC          ← Who can do what                      │
│  Layer 2: ResourceQuota ← How much total resource               │
│  Layer 3: LimitRange    ← Per-Pod and Per-Container limits      │
│  Layer 4: NetworkPolicy ← Which traffic is allowed in/out       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

Used together, these four layers give you full tenant isolation on a shared cluster.

---

## Layer 1: RBAC — Access Control

RBAC controls which users and service accounts can perform which operations on which resources within a namespace.

### Setting Up Team Access

```yaml
# rbac-production.yaml
---
# Role: developers can view Pods and read logs but cannot delete or modify
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-view
  namespace: production
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "pods/log", "deployments", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/exec"]           # allow exec into Pods for debugging
  verbs: ["create"]

---
# Role: operators can deploy and manage workloads
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: operator
  namespace: production
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["pods", "deployments", "services", "configmaps", "jobs", "cronjobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

---
# Bind developer role to the dev-team group
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-view
  namespace: production
subjects:
- kind: Group
  name: dev-team                     # IAM group mapped via aws-auth ConfigMap
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-view
  apiGroup: rbac.authorization.k8s.io

---
# Bind operator role to the ops-team group
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ops-team-operator
  namespace: production
subjects:
- kind: Group
  name: ops-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: operator
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Test what a user can do
kubectl auth can-i delete pods -n production --as-group=dev-team --as=alice
# no

kubectl auth can-i get pods -n production --as-group=dev-team --as=alice
# yes

kubectl auth can-i delete pods -n production --as-group=ops-team --as=bob
# yes
```

---

## Layer 2: ResourceQuota — Namespace Budget

ResourceQuota sets the total resource ceiling for the namespace. Once the quota is reached, no new resources can be created until existing ones are deleted.

### Multi-Tier Quota Strategy

Different namespaces get different quotas based on their role:

```yaml
# Production: high quota — real traffic
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
  namespace: production
spec:
  hard:
    requests.cpu: "50"
    requests.memory: 100Gi
    limits.cpu: "100"
    limits.memory: 200Gi
    pods: "200"
    services.loadbalancers: "10"
```

```yaml
# Staging: medium quota — pre-production testing
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
  namespace: staging
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    services.loadbalancers: "2"
```

```yaml
# Dev: low quota — development work
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services.loadbalancers: "0"   # no LoadBalancers in dev — use NodePort or port-forward
```

This prevents the development team from accidentally provisioning expensive AWS load balancers in their dev environment.

### Monitoring Quota Usage

```bash
# Check quota usage in all namespaces
for ns in production staging dev; do
  echo "=== $ns ==="
  kubectl describe resourcequota -n $ns | grep -A 20 "Resource\s*Used"
done
```

---

## Layer 3: LimitRange — Default and Boundary Limits

LimitRange works at the individual Pod and container level. It sets defaults and hard boundaries within the namespace's total quota.

### Why You Need Both LimitRange and ResourceQuota

| | ResourceQuota | LimitRange |
|---|--------------|------------|
| **Scope** | Entire namespace | Individual Pod/container |
| **Controls** | Total usage | Per-container defaults and boundaries |
| **Required with Quotas?** | — | ✅ Yes — without LimitRange, Pods without resource specs cannot be created in a namespace with a quota |

When a namespace has a ResourceQuota, every Pod must specify resource requests and limits. Without a LimitRange to set defaults, Pods without explicit resource specs are rejected. LimitRange provides those defaults.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limits
  namespace: staging
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: "200m"
      memory: "256Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    max:
      cpu: "2"
      memory: "2Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
```

With this LimitRange, a developer who applies a Pod without resource specs gets `request: 200m/256Mi, limit: 500m/512Mi` automatically — and the quota tracking works correctly.

---

## Layer 4: NetworkPolicy — Traffic Isolation

By default, all Pods in a Kubernetes cluster can communicate with all other Pods, regardless of namespace. NetworkPolicies restrict this.

### Default Deny All — The Starting Point

The recommended production pattern is to start by denying all traffic, then explicitly allow what is needed:

```yaml
# 1. Deny all ingress and egress by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}         # applies to ALL Pods in the namespace
  policyTypes:
  - Ingress
  - Egress
  # No ingress or egress rules = deny everything
```

After applying this, no traffic can reach or leave any Pod in `production`. Now add explicit allows:

```yaml
# 2. Allow ingress from the load balancer (AWS ALB)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-app              # only applies to web-app Pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx   # only from ingress namespace
    ports:
    - protocol: TCP
      port: 8080
```

```yaml
# 3. Allow web-app to talk to the database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres              # applies to database Pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web-app           # only web-app Pods can reach postgres
    ports:
    - protocol: TCP
      port: 5432
```

```yaml
# 4. Allow DNS resolution (egress to CoreDNS)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
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
    - protocol: TCP
      port: 53
```

> ⚠️ **Warning:** If you apply a default-deny NetworkPolicy and forget to allow DNS (port 53), all Pods lose the ability to resolve service names. Everything breaks silently — no error message, just DNS timeouts. Always allow DNS egress when using default-deny.

---

## Complete Namespace Setup — Everything Together

Here is a complete namespace setup for a production team:

```bash
# Apply everything in the correct order

# 1. Create namespace with labels
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    name: production
    environment: production
EOF

# 2. Apply resource defaults (LimitRange first — needed before quota enforces requests)
kubectl apply -f limitrange.yaml -n production

# 3. Apply quota
kubectl apply -f resourcequota.yaml -n production

# 4. Apply RBAC
kubectl apply -f rbac.yaml -n production

# 5. Apply network policies (default-deny + specific allows)
kubectl apply -f network-policies.yaml -n production

# Verify everything is in place
kubectl get limitrange,resourcequota,networkpolicy,rolebinding -n production
```

---

## Verifying Isolation Works

Test that isolation is actually enforced:

```bash
# Test: dev team cannot delete Pods in production
kubectl delete pod web-pod-abc123 -n production --as=alice --as-group=dev-team
# Error from server (Forbidden): pods "web-pod-abc123" is forbidden:
# User "alice" cannot delete resource "pods" in API group "" in the namespace "production"

# Test: staging cannot exceed its quota
kubectl scale deployment web-app --replicas=100 -n staging
# Error from server (Forbidden): admission webhook "validate.kyverno.svc" denied:
# exceeded quota: staging-quota, requested: pods=50, used: pods=50, limited: pods=50

# Test: verify network isolation (run a test Pod in dev, try to reach production)
kubectl run -it --rm test --image=busybox -n dev -- sh
# Inside the test Pod:
wget -T 5 http://web-service.production.svc.cluster.local:8080
# Connecting to web-service.production.svc.cluster.local (10.100.1.45:8080)
# wget: download timed out
# ← NetworkPolicy blocked this traffic
```

---

## Summary

- Namespace isolation requires four layers working together: RBAC (access), ResourceQuota (total budget), LimitRange (per-Pod defaults), and NetworkPolicies (traffic control)
- RBAC uses `Role` (namespace-scoped) and `RoleBinding` to give groups different levels of access per namespace
- ResourceQuota sets the total resource ceiling — when hit, no new resources can be created; use different quotas for production, staging, and dev
- LimitRange sets defaults for Pods that do not specify resources — required when ResourceQuota is active
- NetworkPolicy default-deny-all is the recommended production baseline — then explicitly allow DNS, ingress, and inter-service traffic
- Always allow DNS egress (port 53 to kube-system) when using default-deny NetworkPolicies — without it, all Pod DNS resolution breaks

In the next file, `04-namespace-best-practices.md`, you will learn the proven organizational patterns that teams use for namespaces — naming conventions, environment strategies, and what to avoid.
