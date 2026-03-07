# Label Selectors

## Introduction

Labels are only half of the story. The other half is **selectors** — the expressions you use to query objects by their labels. Selectors are how Kubernetes resources find and connect to each other, and how you filter resources with `kubectl`.

There are two types of selectors: **equality-based** and **set-based**. Understanding both gives you precise control over which resources are targeted by any operation.

---

## Equality-Based Selectors

Equality-based selectors use `=`, `==`, or `!=` to match labels:

```
key=value      → label with this key must have exactly this value
key==value     → same as = (both forms work)
key!=value     → label with this key must NOT have this value
```

### In kubectl

```bash
# Find all Pods with app=web-server
kubectl get pods -l app=web-server

# Find all Pods with environment=production
kubectl get pods -l environment=production

# Combine multiple conditions (AND — all must match)
kubectl get pods -l app=web-server,environment=production

# Find Pods where environment is NOT production
kubectl get pods -l environment!=production

# Find Pods across all namespaces with a specific label
kubectl get pods --all-namespaces -l app=web-server

# Find nodes not in the gpu node group
kubectl get nodes -l eks.amazonaws.com/nodegroup!=gpu-workers
```

### In YAML — Services and ReplicationControllers

Services and ReplicationControllers use equality-based selectors:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:             # equality-based only — Services do not support set-based
    app: web-server
    environment: production
  ports:
  - port: 80
    targetPort: 8080
```

The selector `app: web-server, environment: production` routes traffic to Pods that have BOTH labels set to those exact values.

---

## Set-Based Selectors

Set-based selectors are more powerful. They use `in`, `notin`, and `exists` operators:

```
key in (value1, value2)     → key must be one of these values
key notin (value1, value2)  → key must NOT be any of these values
key                          → key must exist (any value)
!key                         → key must NOT exist
```

### In kubectl

```bash
# Find Pods where environment is either production OR staging
kubectl get pods -l 'environment in (production, staging)'

# Find Pods where tier is NOT frontend
kubectl get pods -l 'tier notin (frontend, cache)'

# Find Pods that have the 'canary' label (any value)
kubectl get pods -l canary

# Find Pods that do NOT have the 'canary' label
kubectl get pods -l '!canary'

# Combine equality and set-based
kubectl get pods -l 'app=web-server,environment in (production, staging)'

# Find Nodes with specific instance types
kubectl get nodes -l 'node.kubernetes.io/instance-type in (t3.large, t3.xlarge)'
```

### In YAML — Deployments, Jobs, and NodeAffinity

Deployments, ReplicaSets, DaemonSets, and Jobs support set-based selectors via `matchLabels` and `matchExpressions`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  selector:
    matchLabels:                       # equality-based conditions
      app: web-server
    matchExpressions:                  # set-based conditions
    - key: environment
      operator: In
      values: [production, staging]
    - key: tier
      operator: NotIn
      values: [database, cache]
    - key: release                     # must have 'release' label
      operator: Exists
    - key: deprecated                  # must NOT have 'deprecated' label
      operator: DoesNotExist
```

All conditions in `matchLabels` and `matchExpressions` are ANDed together — all must match.

---

## How Deployments Use Selectors — The Critical Rule

A Deployment's `selector` must match its Pod template's labels. This is enforced and immutable — once a Deployment is created, you cannot change its selector.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-server          # ← Deployment manages Pods with this label
  template:
    metadata:
      labels:
        app: web-server        # ← Pod template MUST include selector labels
        version: "2.1.0"      # ← extra labels are fine
    spec:
      containers:
      - name: web
        image: nginx:1.25
```

> ⚠️ **Warning:** The selector labels (`app: web-server`) must be present in the Pod template labels. If they do not match, the Deployment is rejected. Extra labels in the template (like `version: "2.1.0"`) are fine — they are not required by the selector.

---

## How Services Use Selectors for Dynamic Load Balancing

The Service selector continuously watches for Pods matching its labels. As Pods are created and deleted, the Service's endpoint list updates automatically:

```bash
# Watch the Service endpoints in real time
kubectl get endpoints web-service -n production -w

# Initial state (3 Pods running):
# NAME          ENDPOINTS                                   AGE
# web-service   10.0.1.5:8080,10.0.1.6:8080,10.0.1.7:8080  5m

# Scale up to 5 replicas:
kubectl scale deployment web-deployment --replicas=5 -n production

# Endpoints update automatically:
# NAME          ENDPOINTS                                                          AGE
# web-service   10.0.1.5:8080,10.0.1.6:8080,10.0.1.7:8080,10.0.1.8:8080,10.0.1.9:8080  5m

# Scale back down:
kubectl scale deployment web-deployment --replicas=2 -n production

# Endpoints shrink:
# NAME          ENDPOINTS                AGE
# web-service   10.0.1.5:8080,10.0.1.6:8080  5m
```

You never manually update a Service's endpoint list — the label selector handles this continuously.

---

## Canary Deployments With Label Selectors

Label selectors enable a canary deployment pattern — sending a small percentage of traffic to a new version while most traffic goes to the stable version:

```yaml
# Stable Deployment (v1.0.0) — 9 replicas
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: web-server
      track: stable
  template:
    metadata:
      labels:
        app: web-server      # ← matches the Service selector
        track: stable
        version: "1.0.0"
    spec:
      containers:
      - name: web
        image: web-app:1.0.0
---
# Canary Deployment (v2.0.0) — 1 replica
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-server
      track: canary
  template:
    metadata:
      labels:
        app: web-server      # ← matches the Service selector too
        track: canary
        version: "2.0.0"
    spec:
      containers:
      - name: web
        image: web-app:2.0.0
---
# Service selects BOTH deployments — routes 90% to stable, 10% to canary
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web-server          # ← matches ALL web Pods regardless of track
  ports:
  - port: 80
    targetPort: 8080
```

With 9 stable Pods and 1 canary Pod, traffic is split approximately 90%/10% by the Service's round-robin load balancing. This is the simplest canary pattern — for more precise traffic splitting, use Ingress with weights or a service mesh.

---

## Node Selectors and Node Affinity

Label selectors also target nodes — controlling where Pods are scheduled:

### Simple NodeSelector

```yaml
spec:
  nodeSelector:
    eks.amazonaws.com/nodegroup: gpu-workers   # must run on gpu-workers node group
    topology.kubernetes.io/zone: us-east-1a   # AND in us-east-1a only
```

### Advanced NodeAffinity (Set-Based)

NodeAffinity uses the full set-based selector syntax:

```yaml
spec:
  affinity:
    nodeAffinity:
      # Hard requirement (Pod will not schedule if not met)
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node.kubernetes.io/instance-type
            operator: In
            values:
            - t3.large
            - t3.xlarge
            - m5.large           # any of these instance types
          - key: eks.amazonaws.com/capacityType
            operator: In
            values:
            - ON_DEMAND          # only on-demand, not spot instances

      # Soft preference (scheduler tries but does not require)
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100              # highest weight = strongest preference
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - us-east-1a         # prefer us-east-1a but will use others if needed
```

---

## Selector-Based Operations With kubectl

```bash
# Delete all Pods matching a label (dangerous — use carefully)
kubectl delete pods -l app=web-server -n production

# Scale all Deployments with a specific label
kubectl scale deployment -l team=backend --replicas=3 -n production

# Get all resources of any type with a label
kubectl get all -l app=web-server -n production

# Output:
# NAME                                READY   STATUS    RESTARTS   AGE
# pod/web-deployment-abc123-xyz       1/1     Running   0          5m
# pod/web-deployment-abc123-def       1/1     Running   0          5m
#
# NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)
# service/web-svc   ClusterIP   172.20.45.123  <none>        80/TCP
#
# NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/web-deployment   2/2     2            2           5m

# Apply network policy to specific Pods using label selector
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-db-access
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database     # only apply to database Pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend  # only backend Pods can connect
EOF
```

---

## Summary

- **Equality-based selectors** use `=`, `==`, `!=` — match Pods where a label has or does not have a specific value
- **Set-based selectors** use `in`, `notin`, `exists`, `!` — match Pods where a label is one of a set of values, or where a label exists or does not exist
- Services and ReplicationControllers support equality-based selectors only; Deployments, ReplicaSets, and Jobs support both via `matchLabels` and `matchExpressions`
- A Deployment's `selector.matchLabels` must match its Pod template labels — this constraint is immutable once the Deployment is created
- The canary deployment pattern uses label selectors to split traffic: stable Pods and canary Pods both have `app: web-server`, so the Service routes to all of them proportionally
- Node affinity uses set-based selectors to control scheduling placement — `In` for required node types, `preferred` for soft preferences
- `kubectl get all -l <selector>` shows all resource types matching a label at once

In the next file, `03-annotations.md`, you will learn about annotations — similar to labels but designed for metadata storage rather than selection, and how they are used by tools like Helm, Argo CD, and AWS Load Balancer Controller.
