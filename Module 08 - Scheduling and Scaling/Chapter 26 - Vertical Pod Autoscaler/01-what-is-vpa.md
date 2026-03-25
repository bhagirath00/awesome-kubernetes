# What is VPA?

## Introduction

Your pod uses way more memory than it needs. It's wasting resources.

You could manually change the pod spec, but that's tedious.

**Vertical Pod Autoscaler (VPA)** automatically adjusts CPU and memory requests based on actual usage.

---

## Vertical vs. Horizontal

**Horizontal (HPA):**
```
Add more copies of the app
3 replicas → 10 replicas
```

**Vertical (VPA):**
```
Give more resources to existing pod
1 pod with 2CPU → 1 pod with 8CPU
```

---

## VPA Workflow

```
1. Metrics collected (CPU, memory usage)
   ↓
2. VPA analyzes historical usage
   ↓
3. VPA calculates recommended requests/limits
   ↓
4. VPA updates pod spec
   ↓
5. Pod is recreated with new resources
```

---

## Example

Pod is configured with:
```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi
```

But actual usage is:
```
CPU: 50m (10% of request)
Memory: 100Mi (20% of request)
```

VPA reduces it to:
```yaml
resources:
  requests:
    cpu: 100m      # Was 500m
    memory: 200Mi  # Was 512Mi
  limits:
    cpu: 200m      # Was 1000m
    memory: 400Mi  # Was 1Gi
```

Pod uses less resources → saves money.

---

## VPA Modes

### 1. Off (Recommendation Only)

```yaml
updatePolicy:
  updateMode: "Off"
```

VPA only recommends. You apply changes manually.

### 2. Initial (Set Once)

```yaml
updatePolicy:
  updateMode: "Initial"
```

VPA sets resources when pod is first created. Never changes after.

### 3. Recreate (Update and Recreate)

```yaml
updatePolicy:
  updateMode: "Recreate"
```

VPA updates and recreates pod. Old pod is deleted first.

### 4. Auto (Smart Update)

```yaml
updatePolicy:
  updateMode: "Auto"
```

VPA updates pod in-place if possible, recreates if needed.

---

## Basic Example

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: app
  
  updatePolicy:
    updateMode: "Auto"
  
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: 2000m
        memory: 2Gi
```

---

## When to Use VPA

### Good For:

- **Underprovisioned workloads** (pod needs more resources)
- **Right-sizing** (optimize resource requests)
- **Long-running services** (steady load)
- **Single-instance apps** (can't use HPA)

### Bad For:

- **Variable load** (use HPA instead)
- **Real-time apps** (pod restart causes downtime)
- **Stateful workloads** (data loss risk)
- **Cost-sensitive apps** (recreation wastes money)

---

## VPA vs. HPA

```
HPA: Scale copies
  Traffic high → More replicas
  Traffic low → Fewer replicas
  
VPA: Scale resources
  Pod needs more CPU → Increase requests
  Pod uses little memory → Decrease requests
  
Use together:
  HPA for replicas (handle concurrent load)
  VPA for pod resources (handle per-pod efficiency)
```

---

## VPA Recommendations

View what VPA recommends:

```bash
# Check VPA status
kubectl describe vpa app-vpa

Status:
  Recommendation:
    ContainerRecommendations:
    - ContainerName: app
      LowerBound:
        Cpu: 50m
        Memory: 64Mi
      Target:
        Cpu: 200m
        Memory: 256Mi
      UncappedTarget:
        Cpu: 250m
        Memory: 300Mi
      UpperBound:
        Cpu: 2000m
        Memory: 2Gi
```

---

## Installation

```bash
# Clone VPA repo
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# Run installer
./hack/vpa-up.sh
```

Installs VPA components:
- vpa-recommender (calculates recommendations)
- vpa-updater (applies updates)
- vpa-admission-controller (prevents conflicts)

---

## Common Mistakes

### Mistake 1: Using VPA with Deployments

```yaml
# WRONG — Deployment gets recreated constantly
targetRef:
  kind: Deployment
  name: app

# VPA recreates pod, which recreates Deployment, which creates new pod loop
```

Use with StatefulSets instead (designed for in-place updates).

### Mistake 2: No Minimum Allowed

```yaml
# WRONG — pods scale to 0
resourcePolicy:
  containerPolicies:
  - containerName: "*"
    # No minAllowed
```

Pod might scale to 0 and become unschedulable.

### Mistake 3: VPA + HPA Together (Conflicting)

```yaml
# WRONG — VPA and HPA fight over resources
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
# ...

apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
# ...

# VPA scales resources, HPA scales replicas
# Can conflict if not carefully coordinated
```

---

## Summary

VPA automatically adjusts pod resources:

**How it works:**
1. Monitors CPU/memory usage
2. Calculates recommendations
3. Updates pod spec
4. Recreates pod with new resources

**Modes:**
- Off: Recommendations only
- Initial: Set once at creation
- Recreate: Update and restart
- Auto: Update intelligently

**Use when:**
- Need right-sizing
- Long-running single pods
- Not suitable for HPA

**Next:** File 02 covers configuring VPA.
