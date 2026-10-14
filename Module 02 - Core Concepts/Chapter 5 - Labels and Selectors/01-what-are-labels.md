# What Are Labels

## Introduction

Labels are one of the most fundamental concepts in Kubernetes. They are the mechanism that connects resources together — a Service finds its Pods through labels, a Deployment knows which ReplicaSet it manages through labels, and a NodeSelector places Pods on the right nodes through labels.

Without labels, Kubernetes resources would be isolated objects with no way to group, select, or relate them to each other.

---

## What Labels Are

A label is a key-value pair attached to a Kubernetes object. You can attach any number of labels to any Kubernetes resource — Pods, Deployments, Services, Nodes, Namespaces, and more.

```yaml
metadata:
  labels:
    app: web-server
    version: "2.1.0"
    environment: production
    team: backend
    tier: frontend
```

Labels are:
- **Arbitrary** — you define the keys and values. Kubernetes does not care what they mean.
- **Flexible** — you can attach as many as you need (reasonable limit: 64 labels per object)
- **Queryable** — you can filter and select objects using label selectors
- **Not unique** — many resources can have the same label

This last point is what makes labels powerful. If 50 Pods all have `app: web-server`, a Service with selector `app: web-server` routes traffic to all 50 of them automatically.

---

## Label Syntax Rules

Label keys and values must follow specific rules:

### Key Rules

```
[optional-prefix/]name

Optional prefix:
- Must be a DNS subdomain (e.g., kubernetes.io, mycompany.com)
- Max 253 characters
- Labels without prefix are user-defined

Name:
- Required
- Max 63 characters
- Must start and end with alphanumeric character (a-z, 0-9)
- Can contain: letters, digits, hyphens (-), underscores (_), periods (.)
```

### Value Rules

```
- Max 63 characters
- Can be empty ""
- Must start and end with alphanumeric character
- Can contain: letters, digits, hyphens, underscores, periods
```

### Valid Label Examples

```yaml
labels:
  # Simple user labels
  app: web-server
  version: "2.1.0"
  environment: production
  tier: frontend
  team: payments

  # Labels with prefix (used by tools and controllers)
  app.kubernetes.io/name: web-server
  app.kubernetes.io/version: "2.1.0"
  app.kubernetes.io/component: frontend
  app.kubernetes.io/part-of: ecommerce-platform
  app.kubernetes.io/managed-by: helm

  # AWS-specific labels
  eks.amazonaws.com/nodegroup: production-workers
  topology.kubernetes.io/zone: us-east-1a
  topology.kubernetes.io/region: us-east-1
```

### Invalid Label Examples

```yaml
labels:
  # These will be rejected:
  My Label: web          # spaces not allowed in key
  app: my web server     # spaces not allowed in value
  very-long-key-name-that-exceeds-sixty-three-characters-limit: value  # key too long
  1app: web              # key cannot start with digit (unless prefixed)
```

---

## The Kubernetes Recommended Label Set

The Kubernetes project recommends a standard set of labels for all resources. Following this standard makes your cluster easier to understand and integrates better with tools like Helm and Argo CD:

```yaml
metadata:
  labels:
    app.kubernetes.io/name: web-server           # name of the application
    app.kubernetes.io/instance: web-server-prod  # unique instance name
    app.kubernetes.io/version: "2.1.0"           # application version
    app.kubernetes.io/component: frontend        # component within the architecture
    app.kubernetes.io/part-of: ecommerce         # the larger application this belongs to
    app.kubernetes.io/managed-by: helm           # the tool managing the resource
```

Using the `app.kubernetes.io/` prefix indicates these labels follow the standard. You can use them alongside your own custom labels.

---

## Adding Labels to Existing Resources

You can add, update, or remove labels from running resources without restarting them:

```bash
# Add a label to an existing Pod
kubectl label pod web-pod-abc123 release=stable

# Update an existing label (use --overwrite)
kubectl label pod web-pod-abc123 version=2.2.0 --overwrite

# Remove a label (add minus sign after the key)
kubectl label pod web-pod-abc123 release-

# Add a label to all Pods with a specific label
kubectl label pods -l app=web-server canary=true

# Add a label to a node (common for node selection)
kubectl label node ip-10-0-1-100.ec2.internal gpu=true
kubectl label node ip-10-0-1-100.ec2.internal nodegroup=gpu-workers
```

---

## Viewing Labels

```bash
# Show labels for all Pods
kubectl get pods --show-labels

# Output:
# NAME              READY   STATUS    RESTARTS   AGE   LABELS
# web-pod-abc123    1/1     Running   0          5m    app=web-server,env=production,version=2.1.0
# web-pod-def456    1/1     Running   0          5m    app=web-server,env=production,version=2.1.0
# api-pod-xyz789    1/1     Running   0          3m    app=api-server,env=production,version=1.5.0

# Filter by label
kubectl get pods -l app=web-server

# Output:
# NAME              READY   STATUS    RESTARTS   AGE
# web-pod-abc123    1/1     Running   0          5m
# web-pod-def456    1/1     Running   0          5m

# Show specific labels as columns
kubectl get pods -L app,version,environment

# Output:
# NAME              READY   STATUS    RESTARTS   AGE   APP          VERSION   ENVIRONMENT
# web-pod-abc123    1/1     Running   0          5m    web-server   2.1.0     production
# api-pod-xyz789    1/1     Running   0          3m    api-server   1.5.0     production
```

---

## How Labels Connect Resources

The most important use of labels is resource selection. Here is how each major resource type uses labels:

### Service → Pod (Traffic Routing)

A Service uses its `selector` to find which Pods to route traffic to. Any Pod with matching labels receives traffic:

```yaml
# Service routes to all Pods with app: web-server
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web-server        # finds Pods with this label
  ports:
  - port: 80
    targetPort: 8080
```

```yaml
# These Pods receive traffic from the Service
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: web-server        # matches Service selector
spec:
  containers:
  - name: web
    image: nginx:1.25
    ports:
    - containerPort: 8080
```

If you add another Pod with `app: web-server`, it automatically starts receiving traffic — no Service update needed.

### Deployment → ReplicaSet → Pod

A Deployment uses `selector.matchLabels` to manage its ReplicaSets and Pods. The Pod template must include the same labels:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-server      # the Deployment manages Pods with this label
  template:
    metadata:
      labels:
        app: web-server    # Pods created get this label — must match selector
        version: "2.1.0"
    spec:
      containers:
      - name: web
        image: nginx:1.25
```

The `selector.matchLabels` must match the labels in `template.metadata.labels`. If they do not match, the Deployment is rejected.

### HPA → Deployment

A HorizontalPodAutoscaler references its target by name, but the HPA uses the Deployment's Pod template labels to track Pods:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deployment   # targets this deployment
  minReplicas: 2
  maxReplicas: 20
```

---

## Labels on Nodes — Node Selection

Labels are also applied to nodes, and Pods can use `nodeSelector` to choose specific nodes:

```bash
# Label a node group for GPU workloads
kubectl label node ip-10-0-1-200.ec2.internal accelerator=gpu

# Label nodes by availability zone (EKS does this automatically)
# topology.kubernetes.io/zone: us-east-1a  ← auto-applied by EKS
```

```yaml
# Pod that requires a GPU node
spec:
  nodeSelector:
    accelerator: gpu           # only schedule on nodes with this label
```

EKS automatically applies many useful labels to nodes:

```bash
kubectl get node ip-10-0-1-100.ec2.internal --show-labels

# Key auto-applied labels:
# kubernetes.io/arch=amd64
# kubernetes.io/os=linux
# topology.kubernetes.io/zone=us-east-1a
# topology.kubernetes.io/region=us-east-1
# node.kubernetes.io/instance-type=t3.medium
# eks.amazonaws.com/nodegroup=production-workers
# eks.amazonaws.com/capacityType=ON_DEMAND
```

---

## Summary

- Labels are key-value pairs attached to Kubernetes objects — arbitrary, flexible, and queryable
- Keys can have an optional prefix (e.g., `app.kubernetes.io/name`) — the Kubernetes recommended labels use the `app.kubernetes.io/` prefix
- Labels are how resources connect: Service selectors find Pods, Deployment selectors manage ReplicaSets, NodeSelector places Pods on nodes
- Use `kubectl label` to add, update, or remove labels from running resources
- Use `kubectl get pods --show-labels` to see all labels, or `-L key1,key2` to show specific labels as columns
- EKS automatically applies labels to nodes for zone, region, instance type, and node group — use these in node selectors

In the next file, `02-label-selectors.md`, you will learn how to write powerful selector queries — equality-based and set-based selectors — to precisely target resources across your cluster.
