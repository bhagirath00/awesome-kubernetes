# Creating and Managing Namespaces

## Introduction

Creating a namespace takes one command. But setting up a namespace properly — with resource quotas, default limits, and the right labels — takes a bit more work. This file covers everything you need to create and manage namespaces effectively in a real cluster.

---

## Creating a Namespace

### With kubectl (Imperative)

```bash
# Create a namespace immediately
kubectl create namespace production
kubectl create namespace staging
kubectl create namespace dev

# Short form using alias
kubectl create ns monitoring

# Verify
kubectl get namespaces
# NAME              STATUS   AGE
# default           Active   30d
# dev               Active   5s
# kube-node-lease   Active   30d
# kube-public       Active   30d
# kube-system       Active   30d
# monitoring        Active   3s
# production        Active   8s
# staging           Active   7s
```

### With YAML (Declarative — Recommended)

Store namespace definitions in version control so namespaces are created consistently:

```yaml
# production-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    name: production             # used by NetworkPolicies to identify the namespace
    environment: production
    team: platform
  annotations:
    contact: "platform-team@mycompany.com"
    cost-center: "CC-1001"
```

```bash
kubectl apply -f production-namespace.yaml

# Output:
# namespace/production created
```

---

## Setting Default Resource Limits — LimitRange

When a developer creates a Pod without specifying resource requests or limits, the Pod has no limits — it can consume unlimited CPU and memory, potentially starving other Pods on the node.

A **LimitRange** solves this by setting default values for any Pod that does not specify its own:

```yaml
# limitrange-production.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:

  # Defaults for containers
  - type: Container
    default:                     # applied when no limits are set
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:              # applied when no requests are set
      cpu: "100m"
      memory: "128Mi"
    max:                         # hard ceiling — no container can exceed this
      cpu: "4"
      memory: "4Gi"
    min:                         # floor — containers must request at least this
      cpu: "50m"
      memory: "64Mi"

  # Defaults for Pods (sum of all containers)
  - type: Pod
    max:
      cpu: "8"
      memory: "8Gi"

  # Limits for PersistentVolumeClaims
  - type: PersistentVolumeClaim
    max:
      storage: "50Gi"
    min:
      storage: "1Gi"
```

```bash
kubectl apply -f limitrange-production.yaml

# Verify
kubectl describe limitrange default-limits -n production
```

With this LimitRange in place:
- A Pod created without resource specs gets `request: cpu=100m, memory=128Mi` and `limit: cpu=500m, memory=256Mi` automatically
- A Pod requesting more than `cpu=4` is rejected with a validation error

---

## Enforcing Namespace-Wide Quotas — ResourceQuota

A **ResourceQuota** sets the total resource budget for a namespace. No single team can consume more than their allocation:

```yaml
# resource-quota-production.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute resources
    requests.cpu: "20"            # all Pods combined can request up to 20 CPU cores
    requests.memory: 40Gi         # all Pods combined can request up to 40GB RAM
    limits.cpu: "40"
    limits.memory: 80Gi

    # Object counts
    pods: "100"                   # max 100 Pods in this namespace
    services: "20"
    services.loadbalancers: "5"   # max 5 LoadBalancer Services (AWS NLBs — they cost money)
    services.nodeports: "0"       # no NodePort Services allowed
    persistentvolumeclaims: "20"
    secrets: "50"
    configmaps: "50"

    # Storage
    requests.storage: "500Gi"     # total PVC storage across all PVCs
```

```bash
kubectl apply -f resource-quota-production.yaml

# Check current usage vs quota
kubectl describe resourcequota production-quota -n production

# Output:
# Name:                    production-quota
# Namespace:               production
# Resource                 Used    Hard
# --------                 ---     ---
# limits.cpu               4500m   40
# limits.memory            9Gi     80Gi
# pods                     18      100
# requests.cpu             2250m   20
# requests.memory          4608Mi  40Gi
# services                 5       20
# services.loadbalancers   2       5
```

When a namespace hits its quota, new resource creation fails:

```bash
kubectl apply -f new-deployment.yaml -n production
# Error from server (Forbidden): error when creating "new-deployment.yaml":
# pods "new-pod" is forbidden: exceeded quota: production-quota,
# requested: requests.cpu=500m, used: requests.cpu=20, limited: requests.cpu=20
```

The developer sees a clear error and knows they need to either free up resources or request a quota increase.

---

## Working With Namespaces in kubectl

### Always Specifying the Namespace

```bash
# Specify namespace with every command
kubectl get pods -n production
kubectl get services -n staging
kubectl apply -f deployment.yaml -n dev

# Short form (-n is short for --namespace)
kubectl get pods -n prod
```

### Setting a Default Namespace

If you work in the same namespace repeatedly, set it as your default context to avoid typing `-n` every time:

```bash
# Set default namespace for the current context
kubectl config set-context --current --namespace=production

# Now kubectl get pods goes to production without -n
kubectl get pods

# Check which namespace is your current default
kubectl config view --minify | grep namespace:
# namespace: production

# See all context settings
kubectl config get-contexts
```

### Getting Resources Across All Namespaces

```bash
# List resources in all namespaces at once
kubectl get pods --all-namespaces
kubectl get pods -A           # short form

# Output:
# NAMESPACE      NAME                          READY   STATUS    RESTARTS
# kube-system    coredns-6d4b75-abc12          1/1     Running   0
# kube-system    kube-proxy-7bmpv              1/1     Running   0
# production     api-server-abc123             1/1     Running   0
# staging        api-server-def456             1/1     Running   0

# Useful for finding all Pods with a specific label across namespaces
kubectl get pods -A -l app=api-server
```

---

## Deleting a Namespace

Deleting a namespace deletes everything inside it — all Pods, Deployments, Services, Secrets, ConfigMaps, and PVCs:

```bash
# Delete a namespace (WARNING: deletes all resources inside)
kubectl delete namespace dev

# Output:
# namespace "dev" deleted
```

> ⚠️ **Warning:** Deleting a namespace is irreversible and immediate. All resources inside are permanently deleted. PersistentVolume data retention depends on the StorageClass `reclaimPolicy` — with `Delete` policy, the underlying EBS volume is also deleted.

If a namespace is stuck in `Terminating` state for a long time, it usually means a resource inside has a finalizer blocking deletion:

```bash
# Check what is stuck
kubectl get all -n dev

# Find resources with finalizers
kubectl get pods -n dev -o json | jq '.items[] | select(.metadata.finalizers != null) | .metadata.name'

# Force remove a stuck namespace (use with caution)
kubectl get namespace dev -o json | \
  jq '.spec.finalizers = []' | \
  kubectl replace --raw /api/v1/namespaces/dev/finalize -f -
```

---

## Namespace Setup Checklist for Production

When creating a production namespace, apply these in order:

```bash
# 1. Create the namespace
kubectl apply -f namespace.yaml

# 2. Apply resource defaults
kubectl apply -f limitrange.yaml -n production

# 3. Apply resource quota
kubectl apply -f resourcequota.yaml -n production

# 4. Create service accounts for the namespace
kubectl apply -f service-accounts.yaml -n production

# 5. Apply RBAC — who can do what in this namespace
kubectl apply -f rbac.yaml -n production

# 6. Apply network policies — control traffic in and out
kubectl apply -f network-policies.yaml -n production

# Verify the namespace is ready
kubectl describe namespace production
kubectl describe limitrange -n production
kubectl describe resourcequota -n production
```

---

## Summary

- Create namespaces with `kubectl create namespace` or declarative YAML (`kubectl apply`)
- Always add labels to namespaces — they are used by NetworkPolicies to select namespaces
- **LimitRange** sets default resource requests and limits for Pods in a namespace — prevents unbounded resource consumption from Pods without explicit resource specs
- **ResourceQuota** caps total resource usage for the entire namespace — prevents one team from consuming the whole cluster
- Set your default namespace with `kubectl config set-context --current --namespace=<ns>` to avoid typing `-n` constantly
- Use `kubectl get pods -A` to see resources across all namespaces at once
- Deleting a namespace deletes everything inside — use with caution

In the next file, `03-resource-isolation.md`, you will go deeper into how namespaces provide resource isolation — combining ResourceQuotas, LimitRanges, NetworkPolicies, and RBAC for a fully isolated team environment.
