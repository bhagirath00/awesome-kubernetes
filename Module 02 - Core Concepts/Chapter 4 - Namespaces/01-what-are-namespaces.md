# What Are Namespaces

## Introduction

A Kubernetes cluster is shared infrastructure. Multiple teams, multiple applications, and multiple environments often run on the same cluster. Without any organization, everything would be mixed together — development workloads next to production workloads, one team's services colliding with another's.

**Namespaces** solve this problem. A namespace is a virtual cluster within your physical cluster — a way to divide one Kubernetes cluster into multiple isolated environments. Every resource in Kubernetes (except a few cluster-wide ones) belongs to exactly one namespace.

---

## The Core Idea

Think of namespaces like folders on your computer. You organize files into folders so that your work documents do not mix with your personal photos. Namespaces organize Kubernetes resources so that your development environment does not mix with production, and Team A's services do not conflict with Team B's.

```
┌─────────────────────────────────────────────────────────────────┐
│                     KUBERNETES CLUSTER                          │
│                                                                 │
│  ┌───────────────────┐  ┌──────────────────┐  ┌─────────────┐  │
│  │   Namespace:      │  │  Namespace:      │  │  Namespace: │  │
│  │   production      │  │  staging         │  │  dev        │  │
│  │                   │  │                  │  │             │  │
│  │  api-deployment   │  │  api-deployment  │  │  api-deploy │  │
│  │  db-service       │  │  db-service      │  │  db-service │  │
│  │  web-deployment   │  │  web-deployment  │  │  web-deploy │  │
│  │                   │  │                  │  │             │  │
│  │  (real traffic)   │  │  (testing)       │  │  (feature)  │  │
│  └───────────────────┘  └──────────────────┘  └─────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                   kube-system namespace                   │  │
│  │   coredns   kube-proxy   aws-node   metrics-server        │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

Each namespace is fully independent — resources in `production` do not see or conflict with resources in `staging` or `dev`.

---

## What Namespaces Provide

### Name Uniqueness Per Namespace

Resource names must be unique within a namespace, but can repeat across namespaces:

```bash
# This is valid — same name in different namespaces
kubectl get deployment web-app -n production   # production's web-app
kubectl get deployment web-app -n staging      # staging's web-app (different resource)
kubectl get deployment web-app -n dev          # dev's web-app (different resource)
```

This means each team can use sensible names for their resources without worrying about collisions with other teams.

### Scoped Access Control (RBAC)

RBAC (Role-Based Access Control) is namespace-scoped. You can give a user full access to the `dev` namespace but read-only access to `staging` and no access to `production`:

```yaml
# Give the dev team full access to the dev namespace only
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-admin
  namespace: dev                  # scoped to dev namespace only
subjects:
- kind: Group
  name: dev-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin                     # full admin permissions
  apiGroup: rbac.authorization.k8s.io
```

This RoleBinding only applies within the `dev` namespace. The `dev-team` group cannot access `staging` or `production`.

### Resource Quotas Per Namespace

You can limit how much CPU, memory, and how many objects each namespace can use:

```yaml
# staging namespace gets limited resources
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    requests.cpu: "4"             # total CPU requests across all Pods
    requests.memory: 8Gi          # total memory requests
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "50"                    # max 50 Pods
    services: "10"                # max 10 Services
    persistentvolumeclaims: "10"  # max 10 PVCs
```

This ensures the staging team cannot accidentally consume all cluster resources and impact production.

### Network Isolation (with NetworkPolicies)

By default, all Pods in a cluster can communicate with all other Pods, regardless of namespace. NetworkPolicies let you restrict this:

```yaml
# Block all ingress traffic to the production namespace from other namespaces
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: production
spec:
  podSelector: {}                  # applies to all Pods in production
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production         # only allow traffic from within production
```

---

## The Default Namespaces

When you install Kubernetes, four namespaces exist automatically:

```bash
kubectl get namespaces

# NAME              STATUS   AGE
# default           Active   30d
# kube-node-lease   Active   30d
# kube-public       Active   30d
# kube-system       Active   30d
```

### `default`

The default namespace for resources when you do not specify one. When you run `kubectl apply -f pod.yaml` without `-n <namespace>`, the resource goes into `default`.

> ⚠️ **Warning:** Avoid using the `default` namespace in production. It has no resource quotas and no access controls by default — everything can run there unchecked. Create dedicated namespaces for your workloads.

### `kube-system`

All Kubernetes system components run here: CoreDNS, kube-proxy, the metrics server, the AWS node agent, Cluster Autoscaler, and more.

```bash
kubectl get pods -n kube-system

# NAME                       READY   STATUS    RESTARTS   AGE
# aws-node-4xkqp             1/1     Running   0          30d
# coredns-6d4b75cb6d-abc12   1/1     Running   0          30d
# coredns-6d4b75cb6d-def34   1/1     Running   0          30d
# kube-proxy-7bmpv           1/1     Running   0          30d
# metrics-server-abc123      1/1     Running   0          30d
```

> ⚠️ **Warning:** Do not deploy your application workloads into `kube-system`. If your application consumes too many resources, it can starve the system components and destabilize the cluster.

### `kube-public`

A namespace readable by all users (including unauthenticated). Used for cluster information that should be publicly available. Rarely used by application teams.

### `kube-node-lease`

Contains node heartbeat objects (Lease objects). These are how the Node Controller tracks which nodes are alive. Never modify resources in this namespace.

---

## Namespace-Scoped vs Cluster-Scoped Resources

Not all Kubernetes resources belong to a namespace. Some resources are cluster-wide:

**Namespace-scoped resources** (belong to a namespace):

| Resource | Kind |
|----------|------|
| Pods | `Pod` |
| Deployments | `Deployment` |
| Services | `Service` |
| ConfigMaps | `ConfigMap` |
| Secrets | `Secret` |
| Roles | `Role` |
| RoleBindings | `RoleBinding` |
| PersistentVolumeClaims | `PersistentVolumeClaim` |
| ServiceAccounts | `ServiceAccount` |

**Cluster-scoped resources** (no namespace):

| Resource | Kind |
|----------|------|
| Nodes | `Node` |
| PersistentVolumes | `PersistentVolume` |
| StorageClasses | `StorageClass` |
| ClusterRoles | `ClusterRole` |
| ClusterRoleBindings | `ClusterRoleBinding` |
| Namespaces | `Namespace` |
| IngressClasses | `IngressClass` |

```bash
# Check if a resource type is namespaced or cluster-scoped
kubectl api-resources --namespaced=true    # namespace-scoped
kubectl api-resources --namespaced=false   # cluster-scoped
```

---

## DNS and Cross-Namespace Communication

Within a namespace, services communicate by short name:

```bash
# From any Pod in the 'production' namespace:
curl http://web-service        # reaches web-service in production namespace
curl http://web-service:8080   # same, with port
```

To communicate across namespaces, use the full DNS name:

```
<service-name>.<namespace>.svc.cluster.local
```

```bash
# From any Pod, reaching a service in a different namespace:
curl http://web-service.production.svc.cluster.local
curl http://database.production.svc.cluster.local:5432

# From dev namespace reaching staging:
curl http://api.staging.svc.cluster.local:8080
```

This full DNS name always works, even within the same namespace.

---

## Summary

- Namespaces are virtual clusters within a physical Kubernetes cluster — they divide resources into isolated groups
- Resources in different namespaces can have the same name without conflict
- Namespaces enable scoped RBAC (give access to one namespace without affecting others), resource quotas (limit CPU/memory per namespace), and network isolation (with NetworkPolicies)
- Four default namespaces: `default` (avoid in production), `kube-system` (system components), `kube-public`, `kube-node-lease`
- Most resources are namespace-scoped (Pods, Deployments, Services). Some are cluster-scoped (Nodes, PersistentVolumes, ClusterRoles).
- Cross-namespace DNS: `<service>.<namespace>.svc.cluster.local`

In the next file, `02-creating-namespaces.md`, you will create and manage namespaces with `kubectl` and YAML, configure default limits for new namespaces with LimitRanges, and set up ResourceQuotas to protect the cluster.
