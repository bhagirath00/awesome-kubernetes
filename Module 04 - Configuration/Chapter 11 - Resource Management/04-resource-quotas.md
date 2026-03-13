# Resource Quotas

## Introduction

A single team or workload can accidentally consume all cluster resources — monopolizing CPU and memory, creating too many Pods, provisioning too many load balancers. **ResourceQuota** prevents this by setting a hard budget for each namespace. Once a namespace hits its quota, no new resources can be created until existing ones are deleted.

ResourceQuotas operate at the namespace level — they are the financial budget for a tenant in a shared cluster.

---

## How ResourceQuota Works

When you apply a ResourceQuota to a namespace, the API Server enforces it on every create or update request:

```
Developer: kubectl apply -f new-deployment.yaml (which would create 5 more Pods)
         │
         ▼
API Server: does this namespace have a ResourceQuota?
         │ Yes
         ▼
Quota Admission Controller: check current usage + new request vs quota
         │
         ├─ Within quota → allow
         └─ Exceeds quota → reject with 403 Forbidden
```

---

## Creating a ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute resources (sum of requests across all Pods in the namespace)
    requests.cpu: "20"          # total CPU requests cannot exceed 20 cores
    requests.memory: 40Gi       # total memory requests cannot exceed 40 GiB

    # Compute limits (sum of limits across all Pods)
    limits.cpu: "40"
    limits.memory: 80Gi

    # Object counts
    pods: "100"                 # maximum 100 Pods
    services: "20"              # maximum 20 Services
    services.loadbalancers: "5" # maximum 5 LoadBalancer Services (AWS NLBs — cost control)
    services.nodeports: "0"     # no NodePort Services allowed
    replicationcontrollers: "20"
    configmaps: "50"
    secrets: "50"
    persistentvolumeclaims: "20"

    # Storage
    requests.storage: "500Gi"   # total PVC storage across all PVCs
    gold.storageclass.storage.k8s.io/requests.storage: "100Gi"  # limit by StorageClass
    standard.storageclass.storage.k8s.io/persistentvolumeclaims: "10"
```

```bash
kubectl apply -f resourcequota.yaml

# Check current quota usage
kubectl describe resourcequota production-quota -n production

# Name:                    production-quota
# Namespace:               production
# Resource                 Used       Hard
# --------                 ----       ----
# limits.cpu               12000m     40
# limits.memory            24Gi       80Gi
# pods                     48         100
# requests.cpu             6000m      20
# requests.memory          12Gi       40Gi
# services                 8          20
# services.loadbalancers   3          5
```

---

## What Happens When Quota Is Exceeded

```bash
# Scale up a Deployment — but quota will be exceeded
kubectl scale deployment api-server --replicas=20 -n production

# The ReplicaSet Controller tries to create Pods until it hits the quota:
kubectl get events -n production | grep "exceeded quota"

# Error creating: pods "api-server-xyz" is forbidden:
# exceeded quota: production-quota,
# requested: pods=1,
# used: pods=100,
# limited: pods=100

# The Deployment shows 15/20 — got 15 Pods before hitting the 100 Pod limit
kubectl get deployment api-server -n production
# NAME         READY   UP-TO-DATE   AVAILABLE   DESIRED
# api-server   15/20   15           15          20
```

The Deployment does not fail completely — it creates as many Pods as the quota allows, then stops.

---

## Quota Scopes — Applying Quota Differently by Priority

You can apply different quota rules to different Priority Classes. This allows high-priority Pods (production traffic) to have their own budget separate from low-priority Pods (batch jobs):

```yaml
# Quota for high-priority Pods only
apiVersion: v1
kind: ResourceQuota
metadata:
  name: high-priority-quota
  namespace: production
spec:
  hard:
    pods: "50"
    requests.cpu: "10"
    requests.memory: 20Gi
  scopeSelector:
    matchExpressions:
    - scopeName: PriorityClass
      operator: In
      values: ["high-priority"]

---
# Separate quota for best-effort (batch jobs)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: besteffort-quota
  namespace: production
spec:
  hard:
    pods: "20"
  scopes:
  - BestEffort          # applies only to BestEffort QoS Pods
```

Available scopes:
- `Terminating` — Pods with `activeDeadlineSeconds` set (Jobs)
- `NotTerminating` — Pods without deadline (Deployments, StatefulSets)
- `BestEffort` — Pods with no requests/limits
- `NotBestEffort` — Pods with requests/limits (Guaranteed or Burstable)
- `PriorityClass` — Pods with a specific PriorityClass name

---

## Multi-Tier Quota Strategy — Different Namespaces Get Different Budgets

The most common pattern: different namespaces for different environments, each with an appropriate budget:

```yaml
# Production — generous budget for real traffic
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
  namespace: production
spec:
  hard:
    requests.cpu: "40"
    requests.memory: 80Gi
    limits.cpu: "80"
    limits.memory: 160Gi
    pods: "200"
    services.loadbalancers: "10"

# Staging — medium budget for pre-production testing
---
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

# Dev — small budget, no load balancers (use port-forward)
---
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
    services.loadbalancers: "0"    # no AWS NLBs in dev — too expensive
```

---

## Monitoring Quota Usage

```bash
# Check all namespaces quota usage at once
kubectl get resourcequota --all-namespaces

# Get quota utilization percentage for a namespace
kubectl get resourcequota -n production -o json | \
  jq -r '.items[] | .status | to_entries[] | 
  "\(.key): used=\(.value.used // "0") hard=\(.value.hard) (\((.value.used // "0" | tonumber) / (.value.hard | tonumber) * 100 | floor)%)"'

# Example output:
# limits.cpu: used=12 hard=40 (30%)
# limits.memory: used=24Gi hard=80Gi (30%)
# pods: used=48 hard=100 (48%)
```

Set up a Prometheus alert when a namespace reaches 80% of its quota:

```yaml
# Prometheus alert rule
groups:
- name: kubernetes.quotas
  rules:
  - alert: NamespaceQuotaCPUHigh
    expr: |
      kube_resourcequota{resource="requests.cpu", type="used"}
      / kube_resourcequota{resource="requests.cpu", type="hard"} > 0.80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Namespace {{ $labels.namespace }} is using over 80% of its CPU quota"
      description: "CPU quota utilization is {{ $value | humanizePercentage }}"
```

---

## ResourceQuota and LimitRange Together

The complete namespace resource governance setup combines both:

```bash
# Apply in this order for a new namespace:

# 1. LimitRange first — ensures new Pods get default resources
kubectl apply -f limitrange.yaml -n production

# 2. ResourceQuota second — now quota tracking works because every Pod has resources
kubectl apply -f resourcequota.yaml -n production

# Verify both are active
kubectl get limitrange,resourcequota -n production
# NAME                       CREATED AT
# limitrange/default-limits  2024-01-15T10:00:00Z
#
# NAME                                AGE   REQUEST              LIMIT
# resourcequota/production-quota      5d    pods: 48/100         services: 8/20
```

---

## Summary

- ResourceQuota sets a hard budget for a namespace — when hit, no new resources can be created until existing ones are deleted
- Quotas track compute resources (`requests.cpu`, `requests.memory`, `limits.cpu`, `limits.memory`), object counts (`pods`, `services`, `secrets`), and storage (`requests.storage`)
- `services.loadbalancers: "0"` in dev namespaces prevents accidental AWS NLB provisioning (each NLB costs ~$16/month)
- Use quota scopes to apply different budgets to different QoS classes or PriorityClasses within one namespace
- Apply LimitRange before ResourceQuota — without LimitRange, Pods without explicit resources are rejected by the quota system
- Monitor quota utilization with Prometheus and alert at 80% — gives teams time to clean up or request a quota increase before hitting the limit

This completes Chapter 11 — Resource Management, and with it the entire **Module 4 — Configuration**.

You have now completed both Module 3 (Networking) and Module 4 (Configuration). The curriculum now covers a complete end-to-end Kubernetes knowledge base — from cluster architecture and core concepts through networking and configuration management.
