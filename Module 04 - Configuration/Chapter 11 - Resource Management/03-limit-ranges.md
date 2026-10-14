# LimitRanges

## Introduction

In a namespace with a ResourceQuota, every Pod must have resource requests and limits defined. If a developer applies a Pod without them, the Pod is rejected. **LimitRanges** solve this by automatically injecting default requests and limits into Pods that do not specify them.

LimitRanges also set boundaries — the minimum and maximum values any container can request or use — preventing both dangerously resource-hungry Pods and excessively under-resourced ones.

---

## What LimitRanges Do

A LimitRange applies three types of controls:

1. **Default values** — if a container has no requests or limits, inject these defaults
2. **Max values** — the ceiling a container can request; reject anything above
3. **Min values** — the floor a container must meet; reject anything below

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - type: Container         # applies to each container in every Pod
    
    defaultRequest:         # injected if container has no requests
      cpu: "100m"
      memory: "128Mi"
    
    default:                # injected if container has no limits
      cpu: "500m"
      memory: "256Mi"
    
    max:                    # container cannot request more than this
      cpu: "4"
      memory: "4Gi"
    
    min:                    # container must request at least this
      cpu: "50m"
      memory: "64Mi"
```

```bash
kubectl apply -f limitrange.yaml

# Verify
kubectl describe limitrange default-limits -n production

# Name:       default-limits
# Namespace:  production
# Type        Resource  Min    Max    Default Request  Default Limit
# ----        --------  ---    ---    ---------------  -------------
# Container   cpu       50m    4      100m             500m
# Container   memory    64Mi   4Gi    128Mi            256Mi
```

---

## How Default Injection Works

When a developer creates a Pod without resource specs:

```yaml
# Developer applies this (no resources specified):
spec:
  containers:
  - name: web
    image: nginx:1.25
    # No resources block
```

The LimitRange admission controller automatically adds:
```yaml
spec:
  containers:
  - name: web
    image: nginx:1.25
    resources:
      requests:
        cpu: "100m"        # injected from defaultRequest
        memory: "128Mi"    # injected from defaultRequest
      limits:
        cpu: "500m"        # injected from default (limits)
        memory: "256Mi"    # injected from default (limits)
```

The Pod starts with these values — the developer does not need to know about LimitRanges. They just deploy their application. But the namespace administrator knows every Pod will have at least minimal resource constraints.

---

## Rejecting Pods Outside the Min/Max Range

If a developer tries to request more than `max` or less than `min`, the Pod is rejected immediately at the API Server:

```yaml
# This Pod requests 8 CPU cores — exceeds max of 4
spec:
  containers:
  - name: worker
    resources:
      requests:
        cpu: "8"        # 8 > max of 4
      limits:
        cpu: "8"
```

```bash
kubectl apply -f worker-pod.yaml

# Error from server (Forbidden): error when creating "worker-pod.yaml":
# pods "worker" is forbidden: maximum cpu usage per Container is 4,
# but limit is 8.
```

Similarly, requesting 10Mi of memory when the minimum is 64Mi:

```bash
# Error: minimum memory usage per Container is 64Mi, but request is 10Mi
```

---

## LimitRange for Multiple Types

LimitRanges can control Container, Pod, and PersistentVolumeClaim resources separately:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: production
spec:
  limits:
  # Per container limits
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
    min:
      cpu: "50m"
      memory: "64Mi"
    maxLimitRequestRatio:
      cpu: "10"          # limit cannot be more than 10x the request
      memory: "4"        # limit cannot be more than 4x the request

  # Per Pod limits (sum of all containers in the Pod)
  - type: Pod
    max:
      cpu: "8"           # total across all containers in one Pod
      memory: "8Gi"

  # PersistentVolumeClaim size limits
  - type: PersistentVolumeClaim
    max:
      storage: "100Gi"   # maximum PVC size in this namespace
    min:
      storage: "1Gi"     # minimum PVC size
```

### `maxLimitRequestRatio` — Preventing Wild Overprovisioning

Without `maxLimitRequestRatio`, a developer could set `request: cpu=1m, limit: cpu=8`. This means the scheduler reserves almost nothing (1 millicore) but the container could burst to 8 cores — making resource planning useless.

```yaml
maxLimitRequestRatio:
  cpu: "10"     # limit.cpu / request.cpu <= 10
  memory: "4"   # limit.memory / request.memory <= 4
```

With this, if `request.cpu = 100m`, then `limit.cpu <= 1000m`. You cannot have a request of 1m and a limit of 8000m — the ratio is rejected.

---

## Interaction Between LimitRange and ResourceQuota

LimitRange and ResourceQuota work together — LimitRange must be applied before ResourceQuota if you want defaults to work:

```
Apply order:
1. LimitRange → now all Pods get default requests/limits if not specified
2. ResourceQuota → now quota tracking works (every Pod has a request value to count)

Without LimitRange + with ResourceQuota:
→ Pods without explicit resources are REJECTED
  "must specify resource limit for cpu" or similar error
```

```bash
# In a namespace with both:
kubectl get limitrange,resourcequota -n production

# NAME                    CREATED AT
# limitrange/limits       2024-01-15T10:00:00Z

# NAME                         AGE    REQUEST
# resourcequota/namespace-quota  5d    requests.cpu: 5/20, requests.memory: 4Gi/40Gi
```

Every new Pod gets default requests from LimitRange → quota accounting works correctly.

---

## LimitRange Scope — Initial Container vs All Containers

By default, LimitRange defaults apply to all containers. You can limit the scope using `scopeSelector`:

```yaml
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
    scopeSelector:
      matchExpressions:
      - scopeName: PriorityClass
        operator: In
        values: ["high-priority"]  # only apply to high-priority Pods
```

This is an advanced feature for clusters with mixed priority workloads.

---

## Viewing the Effective Defaults for a Namespace

```bash
# What defaults will be applied to new Pods in this namespace?
kubectl describe limitrange -n production

# Type        Resource   Min    Max   Default Request  Default Limit  Max Limit/Request Ratio
# ----        --------   ---    ---   ---------------  -------------  -----------------------
# Container   cpu        50m    4     100m             500m           10
# Container   memory     64Mi   4Gi   128Mi            256Mi          4
# PVC         storage    1Gi    100Gi  -                -              -

# Check what was actually injected into a Pod
kubectl get pod web-pod -n production -o yaml | grep -A 8 "resources:"
# resources:
#   requests:
#     cpu: 100m
#     memory: 128Mi
#   limits:
#     cpu: 500m
#     memory: 256Mi
```

---

## Summary

- LimitRanges apply three controls: **defaults** (injected if container has none), **max** (ceiling — rejected if exceeded), and **min** (floor — rejected if below)
- `defaultRequest` is injected when the container has no requests; `default` (limits) is injected when the container has no limits
- `maxLimitRequestRatio` prevents wild overprovisioning — the ratio of limit to request cannot exceed the specified multiplier
- LimitRanges apply to Container, Pod, and PersistentVolumeClaim separately
- LimitRange must be applied before ResourceQuota — otherwise Pods without explicit resources are rejected by the quota system
- Check effective defaults: `kubectl describe limitrange -n <namespace>`

In the next file, `04-resource-quotas.md`, you will learn about ResourceQuotas — the namespace-level budget that caps the total resources all Pods in a namespace can consume.
