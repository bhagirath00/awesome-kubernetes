# ReplicaSet vs Deployment

## Introduction

ReplicaSets and Deployments are closely related — a Deployment manages ReplicaSets, and ReplicaSets manage Pods. Understanding exactly where one ends and the other begins helps you make the right choice, debug the right object when things go wrong, and understand why Kubernetes is designed this way.

---

## The Relationship — One Controls the Other

```
Deployment
    │  spec.selector → "manages ReplicaSets that have these labels"
    │  spec.template → "Pod template to use for new Pods"
    │
    ▼ creates and manages
ReplicaSet (v1.0) — DESIRED: 0, hash: abc123
ReplicaSet (v2.0) — DESIRED: 3, hash: def456  ← current
    │  spec.selector → "manages Pods that have these labels"
    │  spec.template → "Pod template (copied from Deployment)"
    │
    ▼ creates and manages
Pod-1  Pod-2  Pod-3
```

The Deployment controls which ReplicaSet is "current" and how many replicas it should have. The ReplicaSet controls how many Pods match its selector. Pods run the actual container.

---

## How Updates Work — The ReplicaSet Dance

When you update a Deployment's Pod template (e.g., change the image):

```bash
# Old state:
# ReplicaSet abc123 (v1.0): DESIRED=3, CURRENT=3
# ReplicaSet def456 (v2.0): does not exist yet

kubectl set image deployment/web-app web=nginx:1.26 -n production

# Deployment creates a new ReplicaSet for the new template
# New state (during rolling update):
# ReplicaSet abc123 (v1.0): DESIRED=2 → 1 → 0   (scaled down)
# ReplicaSet def456 (v2.0): DESIRED=1 → 2 → 3   (scaled up)

# Final state:
# ReplicaSet abc123 (v1.0): DESIRED=0, CURRENT=0  ← kept for rollback
# ReplicaSet def456 (v2.0): DESIRED=3, CURRENT=3  ← current
```

The Deployment controls the desired replica count of each ReplicaSet. It never deletes old ReplicaSets — it scales them to 0 and keeps them for rollback.

---

## How Rollback Works — ReplicaSet in Reverse

```bash
kubectl rollout undo deployment/web-app -n production

# Deployment detects the previous revision's ReplicaSet
# Scales abc123 back up:
# ReplicaSet abc123 (v1.0): DESIRED=3  ← scaling up
# ReplicaSet def456 (v2.0): DESIRED=0  ← scaling down

# Final state:
# ReplicaSet abc123 (v1.0): DESIRED=3, CURRENT=3  ← active again
# ReplicaSet def456 (v2.0): DESIRED=0, CURRENT=0  ← now the backup
```

A rollback is just: scale up an old ReplicaSet, scale down the current one. No new ReplicaSet is created.

---

## When Should You Use a ReplicaSet Directly?

**Almost never.** The Kubernetes documentation explicitly states that you should use Deployments instead of ReplicaSets directly.

The only scenario where using a ReplicaSet directly makes sense:

**Custom update orchestration** — when your application requires a completely custom update process that the rolling update, blue-green, or canary patterns cannot achieve. You manage two ReplicaSets manually, scaling them according to your custom logic.

```bash
# Custom update: scale new RS manually with business logic
# (extremely rare — most teams use Argo Rollouts or Flagger instead)
kubectl scale replicaset web-app-v2 --replicas=1 -n production
# ... wait, run integration tests ...
kubectl scale replicaset web-app-v2 --replicas=2 -n production
kubectl scale replicaset web-app-v1 --replicas=1 -n production
# ... wait, check metrics ...
kubectl scale replicaset web-app-v2 --replicas=3 -n production
kubectl scale replicaset web-app-v1 --replicas=0 -n production
```

In practice, tools like **Argo Rollouts** provide this capability with proper GitOps integration — you should use those tools rather than directly managing ReplicaSets.

---

## Inspecting the Deployment-ReplicaSet Link

```bash
# See which Deployment owns a ReplicaSet
kubectl get replicaset api-server-5d8f9b7c4d -n production \
  -o jsonpath='{.metadata.ownerReferences}'

# [{"apiVersion":"apps/v1","blockOwnerDeletion":true,"controller":true,"kind":"Deployment","name":"api-server","uid":"abc-123-def"}]
# ↑ ownerReferences links the ReplicaSet to its parent Deployment

# See which ReplicaSet owns a Pod
kubectl get pod api-server-5d8f9b7c4d-xyz12 -n production \
  -o jsonpath='{.metadata.ownerReferences}'
# [{"apiVersion":"apps/v1","kind":"ReplicaSet","name":"api-server-5d8f9b7c4d","uid":"def-456-ghi"}]

# Tree view: Deployment → ReplicaSet → Pods
kubectl get deployment api-server -n production -o wide
kubectl get replicasets -n production -l app=api-server
kubectl get pods -n production -l app=api-server
```

Owner references are what make cascading deletes work. When you delete a Deployment, Kubernetes follows the owner references and deletes the ReplicaSets, then the Pods.

---

## Pod Template Hash — How ReplicaSets Are Named

Kubernetes generates a unique hash from the Pod template contents and appends it to the ReplicaSet name:

```
Deployment name: api-server
Pod template hash (sha256 of spec): 5d8f9b7c4d

ReplicaSet name: api-server-5d8f9b7c4d
Pod names:       api-server-5d8f9b7c4d-<random-suffix>
```

The hash is also applied as a label to every Pod:

```bash
kubectl get pods -n production -l app=api-server --show-labels

# NAME                              READY   LABELS
# api-server-5d8f9b7c4d-abc12      1/1     app=api-server,pod-template-hash=5d8f9b7c4d
# api-server-5d8f9b7c4d-def34      1/1     app=api-server,pod-template-hash=5d8f9b7c4d
```

If the template changes (new image) → new hash → new ReplicaSet name → new Pod names.

The `pod-template-hash` label is how the Deployment controller identifies which ReplicaSet is "active" and which is historical.

---

## Debugging at the Right Level

When things go wrong, knowing which object to debug saves time:

| Problem | Check |
|---------|-------|
| Pods not being created | `kubectl describe replicaset <name>` — quota exceeded, image pull error |
| Rolling update stuck | `kubectl describe deployment <name>` — check Conditions |
| Pod crashes repeatedly | `kubectl describe pod <name>` — check Events and probe results |
| Service not routing traffic | `kubectl get endpoints` — check if Pods are Ready |
| Wrong replica count | `kubectl describe replicaset` — may be blocked by HPA |

```bash
# Quick health check: Deployment → ReplicaSet → Pods
kubectl get deployment api-server -n production     # check READY count
kubectl get rs -n production -l app=api-server      # check DESIRED vs CURRENT
kubectl get pods -n production -l app=api-server    # check Pod STATUS
kubectl describe pod <failing-pod> -n production    # check Events for root cause
```

---

## Summary

- A Deployment manages ReplicaSets; a ReplicaSet manages Pods — each layer handles a specific concern
- When a Deployment image changes, a new ReplicaSet is created; the old one is scaled to 0 but kept for rollback
- Rollback is simply scaling up an old ReplicaSet and scaling down the current one — no new ReplicaSet is created
- Never use ReplicaSets directly in modern Kubernetes — use Deployments (or StatefulSets/DaemonSets for their specific use cases)
- ReplicaSet names embed a hash of the Pod template — changing anything in the template generates a new ReplicaSet
- When debugging, check the right layer: Deployment for update issues, ReplicaSet for scheduling/creation issues, Pod for runtime issues

In the next file, `03-scaling-replicasets.md`, you will learn how to scale ReplicaSets (via Deployments) — manual scaling, HPA, and Cluster Autoscaler — the complete picture of Kubernetes scaling.
