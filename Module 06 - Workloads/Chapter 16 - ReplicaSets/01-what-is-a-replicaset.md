# What is a ReplicaSet

## Introduction

A **ReplicaSet** is the Kubernetes object that ensures a specified number of identical Pod replicas are running at all times. It continuously watches the cluster — if a Pod crashes or a node fails, the ReplicaSet creates a replacement Pod. If there are too many Pods (perhaps from manual creation), it deletes the excess.

In practice, you almost never create a ReplicaSet directly. You create a **Deployment**, which creates and manages ReplicaSets on your behalf. However, understanding ReplicaSets is essential — they are the foundation of how Kubernetes maintains Pod count, and you will see them constantly while debugging Deployments.

---

## What a ReplicaSet Does

```
Desired state:  3 Pods with label app=web-server
Current state:  2 Pods (one crashed)

ReplicaSet controller:
  → Detects: current (2) < desired (3)
  → Creates 1 new Pod
  → Monitors until new Pod is Running
  → Now: current (3) = desired (3)

Node failure takes out 2 Pods:
  → Detects: current (1) < desired (3)
  → Creates 2 new Pods on available nodes
```

The ReplicaSet controller is a reconciliation loop — it continuously compares desired state to actual state and acts to close the gap.

---

## ReplicaSet YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-server-rs
  namespace: production
  labels:
    app: web-server
spec:
  replicas: 3            # desired Pod count
  
  selector:
    matchLabels:
      app: web-server    # ReplicaSet "owns" Pods with this label
  
  template:              # Pod template — used to create new Pods
    metadata:
      labels:
        app: web-server  # MUST match selector.matchLabels
    spec:
      containers:
      - name: web
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```

```bash
kubectl apply -f replicaset.yaml

kubectl get replicaset web-server-rs -n production

# NAME             DESIRED   CURRENT   READY   AGE
# web-server-rs    3         3         3       30s

kubectl get pods -n production -l app=web-server
# NAME                      READY   STATUS
# web-server-rs-abc12       1/1     Running
# web-server-rs-def34       1/1     Running
# web-server-rs-ghi56       1/1     Running
```

---

## How a ReplicaSet Identifies Its Pods — the Selector

A ReplicaSet uses `selector.matchLabels` to find its Pods. Any Pod in the namespace with matching labels is "owned" by this ReplicaSet — regardless of how it was created.

This has an important implication:

```bash
# Manually create a Pod with matching labels
kubectl run extra-pod --image=nginx:1.25 -n production --labels="app=web-server"

# The ReplicaSet detects it now has 4 Pods (3 desired + 1 extra)
# It immediately deletes one Pod to maintain count = 3
# It may delete your manually created Pod!
kubectl get pods -n production -l app=web-server
# Still shows 3 Pods — one of the four was deleted
```

Never manually create Pods with labels that match an existing ReplicaSet's selector — the ReplicaSet will delete them to maintain the desired count.

---

## ReplicaSet vs Deployment — The Key Difference

| | ReplicaSet | Deployment |
|---|-----------|-----------|
| **Manages** | Pods directly | ReplicaSets (which manage Pods) |
| **Update support** | No — can't rolling update | Yes — rolling update, rollback |
| **Rollback** | No history | Yes — previous ReplicaSets kept |
| **Image change** | Doesn't replace Pods | Creates new ReplicaSet, phases out old |
| **Use directly?** | Rarely | Yes — standard workload object |

If you change the image in a ReplicaSet's `template`, existing Pods are NOT replaced — the new image is only used when new Pods are created (e.g., after a Pod crash). This is why Deployments exist — they manage image updates by creating new ReplicaSets.

```bash
# Change image in ReplicaSet — existing Pods keep the old image
kubectl set image replicaset web-server-rs web=nginx:1.26 -n production

kubectl get pods -n production -l app=web-server \
  -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'
# nginx:1.25   ← old image — Pod was NOT replaced
# nginx:1.25
# nginx:1.25

# Only a new Pod would use nginx:1.26
kubectl delete pod web-server-rs-abc12 -n production
# Pod recreated by ReplicaSet
kubectl get pods -n production -l app=web-server \
  -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'
# nginx:1.25   ← unchanged
# nginx:1.25   ← unchanged
# nginx:1.26   ← the new Pod uses the updated template image
```

This is the exact reason you use Deployments: they handle image upgrades by creating new ReplicaSets rather than just modifying the template.

---

## Viewing ReplicaSets Created by a Deployment

```bash
# A Deployment creates ReplicaSets with generated names
kubectl get replicasets -n production -l app=api-server

# NAME                         DESIRED   CURRENT   READY   AGE
# api-server-5d8f9b7c4d        3         3         3       2h   ← current
# api-server-7f6b4c8e9a        0         0         0       5h   ← old (kept for rollback)
# api-server-2a3b5c6d7e        0         0         0       2d   ← older

# The hash in the name corresponds to a hash of the Pod template spec
# Different image → different hash → new ReplicaSet

# Which revision does each ReplicaSet correspond to?
kubectl get replicaset api-server-5d8f9b7c4d -n production \
  -o jsonpath='{.metadata.annotations}'
# {"deployment.kubernetes.io/revision": "5"}   ← this is revision 5
```

---

## Orphaned ReplicaSets — After Deployment Deletion

```bash
# If you delete a Deployment, all its ReplicaSets and Pods are deleted too (default)
kubectl delete deployment api-server -n production
# Deployment, ReplicaSets, and Pods all deleted

# To keep ReplicaSets and Pods (orphan them):
kubectl delete deployment api-server -n production --cascade=orphan
# Deployment deleted, but ReplicaSets and Pods stay running
# The orphaned ReplicaSet becomes self-managing (no parent Deployment)
```

Orphaned ReplicaSets are useful when you want to delete a Deployment definition from Git but keep the application running during a migration.

---

## Summary

- A ReplicaSet is a reconciliation loop that keeps a specified number of Pod replicas running — it creates replacements if Pods crash and deletes excess if too many exist
- ReplicaSets identify their Pods by label selector — any Pod in the namespace matching the selector is considered owned by the ReplicaSet
- Never manually create Pods with labels matching a ReplicaSet's selector — the ReplicaSet will delete them to maintain the desired count
- Changing a ReplicaSet's `template.image` does NOT replace existing Pods — only new Pods (after crashes) use the new image; this is why Deployments exist
- Deployments create multiple ReplicaSets over time — old ones are kept at 0 replicas for rollback history

In the next file, `02-replicaset-vs-deployment.md`, you will learn when to use ReplicaSets directly and how the Deployment-ReplicaSet relationship plays out during updates, rollbacks, and scaling.
