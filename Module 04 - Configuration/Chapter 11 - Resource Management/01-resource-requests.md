# Resource Requests

## Introduction

When you create a Pod, Kubernetes needs to decide which node to schedule it on. Without any guidance, the scheduler has no way to know if a node has enough capacity for the Pod. **Resource requests** are how you communicate to the scheduler what your Pod needs — a minimum guarantee of CPU and memory.

Getting resource requests right is one of the most impactful tuning decisions in a Kubernetes cluster. Under-requesting leads to noisy neighbors and OOMKills. Over-requesting wastes money by blocking resources that are never used.

---

## What Resource Requests Are

A resource request is a **guarantee**. When you set `requests.cpu: 500m`, you are telling Kubernetes:
- "This container needs at least 0.5 CPU cores to run well"
- "Please only schedule this Pod on a node that has 0.5 free CPU cores available"
- "Reserve 0.5 CPU cores on the node for this container"

The scheduler uses requests (not actual usage) to determine whether a node has capacity.

```yaml
spec:
  containers:
  - name: api
    image: api-server:v2.0
    resources:
      requests:
        cpu: "500m"     # 500 millicores = 0.5 CPU cores
        memory: "256Mi" # 256 mebibytes
```

---

## CPU Request Units

CPU is measured in **millicores** (m). 1000 millicores = 1 CPU core.

```yaml
resources:
  requests:
    cpu: "1"        # 1 full CPU core
    cpu: "1000m"    # same as "1" — 1000 millicores
    cpu: "500m"     # 0.5 CPU core (half a core)
    cpu: "250m"     # 0.25 CPU core (quarter core)
    cpu: "100m"     # 0.1 CPU core (tenth of a core)
    cpu: "50m"      # minimum practical value
```

CPU requests are a **soft limit**. If the node has spare CPU capacity, your container can use more than it requested (it is not capped at the request amount — that is what limits are for). But under contention, each container is guaranteed its requested share.

---

## Memory Request Units

Memory uses binary prefixes:

```yaml
resources:
  requests:
    memory: "64Mi"   # 64 mebibytes = 64 × 1,048,576 bytes
    memory: "256Mi"  # 256 mebibytes
    memory: "512Mi"  # 512 mebibytes
    memory: "1Gi"    # 1 gibibyte = 1,073,741,824 bytes
    memory: "2Gi"    # 2 gibibytes
    memory: "8Gi"    # 8 gibibytes (large JVM or data processing)
```

```
Mi = mebibytes (2^20 bytes = 1,048,576 bytes)
M  = megabytes (10^6 bytes = 1,000,000 bytes)
Gi = gibibytes (2^30 bytes)
G  = gigabytes (10^9 bytes)
```

Always use `Mi` and `Gi` — never `M` and `G` — for precision.

---

## How the Scheduler Uses Requests

The scheduler sums up all resource requests for all Pods on a node and compares against the node's allocatable capacity:

```bash
# See a node's allocatable resources
kubectl describe node ip-10-0-1-100.ec2.internal

# Capacity:
#   cpu:     2
#   memory:  7902Mi
# Allocatable:
#   cpu:     1930m    ← slightly less than capacity (system reserved)
#   memory:  7268Mi   ← slightly less than capacity (kubelet reserved)

# Non-terminated Pods allocated resources:
#   Namespace   Name          CPU Requests   Memory Requests
#   production  web-pod-abc   500m (25%)     256Mi (3%)
#   production  api-pod-def   500m (25%)     512Mi (7%)
#   kube-system aws-node      25m (1%)       64Mi (0%)
#
# Total:        1025m (53%)    832Mi (11%)
#
# Available:    905m (47%)     6436Mi (89%)
```

A new Pod with `requests.cpu: 1000m` cannot be scheduled on this node — only `905m` is available. The Pod goes `Pending` until a node with enough free CPU exists.

---

## Quality of Service (QoS) Classes

Kubernetes assigns a QoS class to every Pod based on how its resource requests and limits are configured. This class determines which Pods are evicted first when a node runs out of resources:

```
Guaranteed → BestEffort → Burstable
(safest)                  (evicted last to first under pressure)

When node runs out of memory, eviction order:
1. BestEffort Pods (no requests or limits set) — evicted first
2. Burstable Pods (requests set but different from limits) — evicted second
3. Guaranteed Pods (requests equal limits) — evicted last
```

### Guaranteed QoS

Pod has requests AND limits, and requests equal limits for both CPU and memory:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"    # same as request
    memory: "512Mi" # same as request
# QoS Class: Guaranteed
```

Guaranteed Pods are never evicted unless they exceed their limits. Use for production-critical Pods.

### Burstable QoS

Pod has requests lower than limits (or has requests but no limits):

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1000m"    # limit higher than request
    memory: "1Gi"
# QoS Class: Burstable
```

Burstable Pods can use extra capacity when available. They are evicted when the node is under pressure and memory usage exceeds their request.

### BestEffort QoS

Pod has no requests or limits defined:

```yaml
resources: {}    # no requests, no limits
# QoS Class: BestEffort
```

BestEffort Pods use whatever is available on the node but are the first to be evicted when resources are tight. **Never use BestEffort in production.**

```bash
# Check a Pod's QoS class
kubectl get pod web-pod -n production -o jsonpath='{.status.qosClass}'
# Guaranteed
```

---

## Setting Requests Based on Actual Usage

Do not guess resource requests. Measure actual usage and set requests slightly above the p95 usage:

```bash
# View current CPU and memory usage for Pods
kubectl top pods -n production

# NAME              CPU(cores)   MEMORY(bytes)
# api-pod-abc123    150m         180Mi
# web-pod-def456    80m          120Mi
# worker-ghi789     600m         450Mi

# Set requests at ~120% of typical usage:
# api-pod: cpu=200m, memory=220Mi
# web-pod: cpu=100m, memory=150Mi
# worker: cpu=750m, memory=550Mi

# For accurate baselines, use Prometheus queries:
# 95th percentile CPU usage over 7 days:
# quantile_over_time(0.95, rate(container_cpu_usage_seconds_total{container="api"}[5m])[7d:5m])

# 95th percentile memory usage over 7 days:
# quantile_over_time(0.95, container_memory_working_set_bytes{container="api"}[7d])
```

Tools like **Goldilocks** or **Vertical Pod Autoscaler (VPA) in recommendation mode** automate this analysis:

```bash
# Install Goldilocks to get request recommendations
helm install goldilocks fairwinds-stable/goldilocks --namespace goldilocks --create-namespace

# Label namespaces for analysis
kubectl label namespace production goldilocks.fairwinds.com/enabled=true

# Goldilocks reads VPA recommendations and shows them in a dashboard
# Run port-forward to access the dashboard:
kubectl port-forward -n goldilocks svc/goldilocks-dashboard 8080:80
# Then open http://localhost:8080 — shows recommended requests/limits per container
```

---

## Summary

- Resource requests are a scheduling guarantee — the scheduler places Pods only on nodes with enough free requested capacity
- CPU in millicores: `1000m = 1 core`, `500m = 0.5 core`. Memory in mebibytes: `256Mi`, `1Gi`
- QoS class determines eviction order — Guaranteed (requests=limits) is safest; BestEffort (no resources) is evicted first
- Base requests on measured p95 usage from `kubectl top pods` or Prometheus; use Goldilocks/VPA recommendation mode to automate this
- Over-requested clusters waste money; under-requested clusters cause noisy neighbors and OOMKills — measure before you set

In the next file, `02-resource-limits.md`, you will learn about limits — the hard ceiling on resource consumption — and the difference between CPU throttling and memory OOMKill behavior.
