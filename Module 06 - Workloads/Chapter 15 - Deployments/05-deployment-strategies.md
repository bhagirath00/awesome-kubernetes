# Deployment Strategies

## Introduction

Beyond the standard rolling update, Kubernetes supports several deployment strategies — each with different trade-offs between speed, risk, capacity requirements, and rollback complexity. Choosing the right strategy depends on your risk tolerance, traffic patterns, and infrastructure.

---

## Strategy 1: Recreate — Stop All, Start All

```yaml
spec:
  strategy:
    type: Recreate
```

Behavior:
1. Scale all existing Pods to 0 (downtime begins)
2. Wait until all old Pods are gone
3. Create all new Pods
4. Downtime ends when new Pods pass readiness probes

```
Before: Pod-A(v1)  Pod-B(v1)  Pod-C(v1)

→ All v1 Pods terminated (DOWNTIME)

After:  Pod-D(v2)  Pod-E(v2)  Pod-F(v2)
```

**When to use:**
- Applications that cannot run two versions simultaneously (e.g., schema lock — one process acquires an exclusive DB lock)
- Development environments where downtime is acceptable
- Applications with resource conflicts between versions

**Never use in production** for customer-facing services. The downtime can be minutes for large clusters.

---

## Strategy 2: Rolling Update — The Default

Covered in depth in `03-rolling-updates.md`. This is the standard for most production services.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

---

## Strategy 3: Blue-Green Deployment

Blue-Green is not a native Kubernetes strategy type — it is a pattern implemented using two Deployments and a Service selector switch.

```
Blue Deployment (v1.0) ← Production traffic (Service pointing here)
Green Deployment (v2.0) ← Idle, fully deployed, tested

Switch:
  Update Service selector → now points to Green
  → 100% traffic instantly moves to Green

If Green fails:
  Update Service selector → point back to Blue
  → 100% traffic instantly back to Blue (< 5 seconds)
```

### Implementation

```yaml
# Blue Deployment — current production
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server-blue
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
      slot: blue
  template:
    metadata:
      labels:
        app: api-server
        slot: blue
        version: "3.0.1"
    spec:
      containers:
      - name: api
        image: api-server:v3.0.1

---
# Green Deployment — new version, idle
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server-green
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
      slot: green
  template:
    metadata:
      labels:
        app: api-server
        slot: green
        version: "3.0.2"
    spec:
      containers:
      - name: api
        image: api-server:v3.0.2

---
# Service — points to Blue initially
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  selector:
    app: api-server
    slot: blue           # ← change to 'green' to switch traffic
  ports:
  - port: 80
    targetPort: 8080
```

**Cutover — switch 100% of traffic:**
```bash
# Verify Green is healthy first
kubectl get deployment api-server-green -n production
# READY: 3/3, AVAILABLE: 3

# Run smoke tests against Green (before cutover)
kubectl port-forward service/api-server-green-test 8080:80 -n production &
curl http://localhost:8080/healthz

# Switch traffic from Blue to Green (instant, no downtime)
kubectl patch service api-service -n production \
  -p '{"spec":{"selector":{"slot":"green"}}}'

# Monitor metrics and error rates for 15 minutes
# ...

# If Green is good → scale down Blue to save costs
kubectl scale deployment api-server-blue --replicas=0 -n production
# (keep Blue at 0 replicas for fast rollback — don't delete yet)

# If Green has issues → instant rollback (< 5 seconds)
kubectl patch service api-service -n production \
  -p '{"spec":{"selector":{"slot":"blue"}}}'
kubectl scale deployment api-server-blue --replicas=3 -n production
```

**Trade-offs:**
- ✅ Instant rollback (just update Service selector)
- ✅ New version fully tested before traffic switch
- ❌ Requires 2× capacity (Blue and Green both need resources)
- ❌ All traffic switches at once — no gradual validation with real traffic

---

## Strategy 4: Canary Deployment

Canary sends a percentage of real production traffic to the new version before full rollout.

```
Blue Deployment (v1.0):  9 replicas  → 90% of traffic
Canary Deployment (v2.0): 1 replica  → 10% of traffic

Service selects both Deployments (same label, different version)
```

### Implementation

```yaml
# Stable Deployment — 9 replicas
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: api-server    # service selects on 'app' only
  template:
    metadata:
      labels:
        app: api-server
        track: stable
    spec:
      containers:
      - name: api
        image: api-server:v3.0.1

---
# Canary Deployment — 1 replica (10% of traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-server
      track: canary
  template:
    metadata:
      labels:
        app: api-server
        track: canary
    spec:
      containers:
      - name: api
        image: api-server:v3.0.2    # new version

---
# Service selects ALL Pods with app=api-server (both stable and canary)
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api-server    # matches both stable and canary Pods
  ports:
  - port: 80
    targetPort: 8080
```

With 9 stable + 1 canary Pod, the Service load balances: ~10% of requests hit the canary.

**Progressive promotion:**
```bash
# Canary looks good after 30 minutes → gradually increase
kubectl scale deployment api-server-canary --replicas=3  -n production
kubectl scale deployment api-server-stable --replicas=7 -n production
# Now 30% canary traffic

# After another hour → 50/50
kubectl scale deployment api-server-canary --replicas=5 -n production
kubectl scale deployment api-server-stable --replicas=5 -n production

# After another hour → full canary
kubectl scale deployment api-server-canary --replicas=10 -n production
kubectl scale deployment api-server-stable --replicas=0 -n production

# Clean up — rename canary to stable (or delete stable, update canary deployment name)
```

---

## Advanced Canary with AWS ALB

The AWS Application Load Balancer supports header-based and weighted routing — more sophisticated canary than replica-count-based:

```yaml
# Ingress with weighted routing between two Services
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-canary-ingress
  annotations:
    alb.ingress.kubernetes.io/actions.weighted-routing: |
      {
        "type": "forward",
        "forwardConfig": {
          "targetGroups": [
            {"serviceName": "api-service-stable", "servicePort": 80, "weight": 90},
            {"serviceName": "api-service-canary", "servicePort": 80, "weight": 10}
          ]
        }
      }
```

This routes 10% of traffic to canary at the ALB level — independent of replica counts. You can have 1 canary Pod receive exactly 10% of requests regardless of total replica count.

---

## Choosing a Strategy

| Strategy | Downtime | Capacity Overhead | Rollback Speed | Real Traffic Validation | Complexity |
|---------|----------|------------------|---------------|------------------------|------------|
| **Recreate** | ✅ Yes | Low | Fast | No | Low |
| **Rolling** | ❌ None | 1 Pod extra | Seconds | No (all traffic) | Low |
| **Blue-Green** | ❌ None | 2× capacity | < 5 seconds | No (all-or-nothing) | Medium |
| **Canary** | ❌ None | Small % extra | Seconds | ✅ Yes (% of traffic) | High |

---

## Summary

- `Recreate` strategy stops all Pods before starting new ones — causes downtime; use only in dev or when two versions genuinely cannot coexist
- **Blue-Green**: run two complete Deployments, switch traffic instantly by changing the Service selector — 2× cost but instant rollback
- **Canary**: run the new version at small scale (1 replica = 10% of traffic), observe metrics, then gradually increase — best for validating changes with real production traffic
- AWS ALB supports weight-based routing for precise canary percentages independent of replica count
- For most production services: Rolling Update is the standard; use Blue-Green for high-stakes releases; use Canary for gradual rollout of risky changes

This completes Chapter 15 — Deployments. In Chapter 16, `01-what-is-a-replicaset.md`, you will learn about ReplicaSets — the underlying mechanism that Deployments use to maintain the desired number of Pod replicas.
