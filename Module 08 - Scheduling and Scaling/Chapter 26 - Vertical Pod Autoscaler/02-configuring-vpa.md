# Configuring VPA

## Basic Configuration

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: StatefulSet
    name: database
  
  updatePolicy:
    updateMode: "Auto"
  
  resourcePolicy:
    containerPolicies:
    - containerName: "db"
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 4000m
        memory: 4Gi
      controlledResources: ["cpu", "memory"]
      controlledValues: RequestsAndLimits
```

---

## VPA for Right-Sizing

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: web-app
  
  updatePolicy:
    updateMode: "Off"  # Only recommend, don't apply
  
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: 1000m
        memory: 1Gi
```

Check recommendations before applying.

---

## VPA with Safe Bounds

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: critical-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: StatefulSet
    name: app
  
  updatePolicy:
    updateMode: "Recreate"  # Explicit recreation
  
  resourcePolicy:
    containerPolicies:
    - containerName: "app"
      minAllowed:
        cpu: 500m          # Never go below
        memory: 512Mi
      maxAllowed:
        cpu: 4000m         # Never go above
        memory: 4Gi
      controlledResources:
      - cpu
      - memory
```

Bounds prevent over-scaling.

---

## Multi-Container Pod

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-with-sidecar-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: app
  
  updatePolicy:
    updateMode: "Auto"
  
  resourcePolicy:
    containerPolicies:
    # Main app container
    - containerName: "app"
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2000m
        memory: 2Gi
    
    # Sidecar container (different limits)
    - containerName: "logging-sidecar"
      minAllowed:
        cpu: 10m
        memory: 32Mi
      maxAllowed:
        cpu: 200m
        memory: 256Mi
```

Each container can have different policies.

---

## Only Control CPU

```yaml
resourcePolicy:
  containerPolicies:
  - containerName: "*"
    minAllowed:
      cpu: 50m
      memory: 64Mi
    maxAllowed:
      cpu: 2000m
      memory: 2Gi
    controlledResources:
    - cpu        # Only control CPU
    # Memory not listed = not controlled
```

VPA adjusts CPU, leaves memory alone.

---

## Real-World Example: Database

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: postgres-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: StatefulSet
    name: postgres
  
  updatePolicy:
    updateMode: "Off"  # Manual review first
  
  resourcePolicy:
    containerPolicies:
    - containerName: "postgres"
      minAllowed:
        cpu: 1000m          # Database needs baseline
        memory: 1Gi
      maxAllowed:
        cpu: 8000m          # But cap expensive
        memory: 8Gi
      controlledResources: ["cpu", "memory"]
      controlledValues: RequestsAndLimits
```

Database uses VPA for right-sizing without constant recreation.

---

## Monitoring VPA

```bash
# Check VPA status
kubectl describe vpa app-vpa
Name: app-vpa
Status:
  Recommendation:
    ContainerRecommendations:
    - ContainerName: app
      Target:
        Cpu: 250m
        Memory: 300Mi
      UpperBound:
        Cpu: 400m
        Memory: 500Mi
      LowerBound:
        Cpu: 100m
        Memory: 100Mi

# View events
kubectl get events --sort-by='.lastTimestamp' | grep vpa

# Check pod history
kubectl get pods --watch  # See pod recreations
```

---

## VPA Troubleshooting

### VPA Not Applying Recommendations

```bash
# Check VPA controller logs
kubectl logs -n kube-system deployment/vpa-updater

# Check if pod has resource requests
kubectl get pods -o yaml | grep -A5 resources

# Without requests, VPA can't calculate percentage
```

### Pod Keeps Restarting

```yaml
# Check update mode
updatePolicy:
  updateMode: "Off"  # Use Off to prevent restarts

# Or check for conflicts (VPA + HPA)
# VPA recreates, HPA sees it as new pod
```

---

## VPA vs Manual Sizing

```
Without VPA:
  1. Monitor pod usage over weeks
  2. Manually calculate appropriate requests
  3. Update pod spec
  4. Deploy
  5. Monitor again
  6. Repeat

With VPA (Recommend mode):
  1. Deploy VPA with updateMode: Off
  2. Wait 24 hours for baseline
  3. Check recommendations
  4. Manually apply when confident
```

---

## Summary

VPA configuration:

**Modes:**
- Off: Recommendations only
- Initial: Set once at creation
- Recreate: Full recreation
- Auto: Smart updates

**Bounds:**
- minAllowed: Never scale below
- maxAllowed: Never scale above

**Control:**
- controlledResources: What to adjust
- controlledValues: Requests, limits, or both

**Next:** File 03 compares VPA vs. HPA.
