# DaemonSet Rolling Updates

## Introduction

Updating a DaemonSet is more constrained than updating a Deployment. Because each node can only run one DaemonSet Pod, you cannot use `maxSurge` — there is no room for an extra Pod alongside the existing one. Updates are always in-place: terminate the old Pod on a node, then start the new Pod on the same node.

Kubernetes supports two DaemonSet update strategies: `RollingUpdate` (gradual, node by node) and `OnDelete` (manual, update only when you explicitly delete Pods).

---

## Update Strategy 1: `RollingUpdate`

The default strategy — Kubernetes updates Pods one node at a time (controlled by `maxUnavailable`):

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1     # update 1 node at a time (can be number or percentage)
      # maxUnavailable: 25% → update 25% of nodes at a time
```

```bash
# Update the DaemonSet image
kubectl set image daemonset/fluent-bit \
  fluent-bit=amazon/aws-for-fluent-bit:2.31.12 \
  -n kube-system

# Watch the rolling update
kubectl rollout status daemonset/fluent-bit -n kube-system

# Waiting for daemon set "fluent-bit" rollout to finish: 0 out of 3 new pods have been updated...
# Waiting for daemon set "fluent-bit" rollout to finish: 1 out of 3 new pods have been updated...
# Waiting for daemon set "fluent-bit" rollout to finish: 2 out of 3 new pods have been updated...
# daemon set "fluent-bit" successfully rolled out

# Watch individual Pod updates
kubectl get pods -n kube-system -l app=fluent-bit -w

# NAME               READY   STATUS              NODE
# fluent-bit-abc12   1/1     Running             ip-10-0-1-100    ← old
# fluent-bit-def34   1/1     Running             ip-10-0-2-100    ← old
# fluent-bit-ghi56   1/1     Running             ip-10-0-3-100    ← old
# fluent-bit-abc12   0/1     Terminating         ip-10-0-1-100    ← updating node 1
# fluent-bit-jkl78   0/1     ContainerCreating   ip-10-0-1-100    ← new Pod on same node
# fluent-bit-jkl78   1/1     Running             ip-10-0-1-100    ← new Pod ready
# fluent-bit-def34   0/1     Terminating         ip-10-0-2-100    ← updating node 2
# ...
```

---

## Update Strategy 2: `OnDelete`

With `OnDelete`, Pods are only updated when you manually delete them. Kubernetes creates a new Pod with the updated template after deletion:

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

```bash
# Update the DaemonSet template
kubectl set image daemonset/falco falco=falcosecurity/falco-no-driver:v0.37.0 -n falco-system

# With OnDelete strategy, NOTHING HAPPENS automatically
# Verify no automatic update:
kubectl get pods -n falco-system -l app=falco
# All Pods still show old image

# Manually trigger update on specific nodes (your choice of timing)
kubectl delete pod falco-abc12 -n falco-system   # node 1 updates
# Wait and verify node 1 is healthy
kubectl delete pod falco-def34 -n falco-system   # node 2 updates
kubectl delete pod falco-ghi56 -n falco-system   # node 3 updates
```

**When to use `OnDelete`:**
- Security agents or network plugins where downtime even for 1 node is extremely risky
- Cluster-critical DaemonSets where you want human approval before each node updates
- During major upgrades when you want to test on one node before proceeding

---

## `maxUnavailable` — Controlling Update Speed

```bash
# Update 1 node at a time (safest — default for critical DaemonSets)
maxUnavailable: 1

# Update 25% of nodes at a time (faster — good for log collectors)
maxUnavailable: 25%

# Update all nodes simultaneously (fastest — risky for network plugins)
maxUnavailable: 100%
# OR
maxUnavailable: 100   # if you have 3 nodes, 100 > 3 → all update at once
```

For VPC CNI (`aws-node`), use `maxUnavailable: 1` — updating CNI tears down Pod networking on a node briefly. One node at a time minimizes impact.

For log collectors (Fluent Bit), `maxUnavailable: 25%` is acceptable — losing log collection on 25% of nodes briefly does not cause a user-facing impact.

---

## Verifying a DaemonSet Update

```bash
# Check overall DaemonSet status
kubectl get daemonset fluent-bit -n kube-system

# NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
# fluent-bit    3         3         3       3            3
# UP-TO-DATE: 3 → all 3 Pods running the new version

# Verify image version on all DaemonSet Pods
kubectl get pods -n kube-system -l app=fluent-bit \
  -o jsonpath='{range .items[*]}{.spec.nodeName}: {.spec.containers[0].image}{"\n"}{end}'
# ip-10-0-1-100.ec2.internal: amazon/aws-for-fluent-bit:2.31.12
# ip-10-0-2-100.ec2.internal: amazon/aws-for-fluent-bit:2.31.12
# ip-10-0-3-100.ec2.internal: amazon/aws-for-fluent-bit:2.31.12

# Check rollout history
kubectl rollout history daemonset/fluent-bit -n kube-system

# REVISION  CHANGE-CAUSE
# 1         Initial installation
# 2         Upgrade to 2.31.12

# View a specific revision
kubectl rollout history daemonset/fluent-bit -n kube-system --revision=2
```

---

## Rolling Back a DaemonSet

```bash
# Roll back to the previous revision
kubectl rollout undo daemonset/fluent-bit -n kube-system

# Roll back to a specific revision
kubectl rollout undo daemonset/fluent-bit -n kube-system --to-revision=1

# Watch rollback
kubectl rollout status daemonset/fluent-bit -n kube-system
# daemon set "fluent-bit" successfully rolled out
```

---

## Handling New Nodes During a Rollout

When Cluster Autoscaler adds a new node during a DaemonSet update, the new node gets the latest DaemonSet template (the current `spec.template`) — not the version that is currently being rolled out. This is correct behavior:

```
DaemonSet update in progress (nodes 1 and 2 have new version, node 3 still on old)
Cluster Autoscaler adds node 4 (due to load spike)
    ↓
DaemonSet controller schedules a Pod on node 4
Pod on node 4 uses the CURRENT template (new version)
```

This means new nodes always get the target version, even during a partial rollout.

---

## DaemonSet Update Best Practices

```bash
# 1. Always check rollout health after update
kubectl rollout status daemonset/node-exporter -n monitoring --timeout=300s
# Non-zero exit code → DaemonSet failed to update

# 2. For network-critical DaemonSets (aws-node, kube-proxy), update during low-traffic periods
# (brief network interruption on each node during CNI Pod restart)

# 3. Monitor node health during CNI update
kubectl get nodes -w
# Nodes should stay Ready during update (brief NotReady is possible during kube-proxy restart)

# 4. After update, verify DaemonSet-managed resources are working
# For fluent-bit: check CloudWatch for continued log flow
# For node-exporter: check Prometheus targets page for all nodes
# For aws-node: check new Pod IP allocations

# 5. If a node's DaemonSet Pod is stuck:
kubectl delete pod fluent-bit-stuck-xyz -n kube-system
# DaemonSet controller recreates the Pod immediately
```

---

## Summary

- DaemonSets support two update strategies: `RollingUpdate` (automatic, node by node) and `OnDelete` (manual — Pods updated only when deleted)
- `maxUnavailable` in RollingUpdate controls how many nodes update simultaneously — use `1` for network plugins, `25%` for log collectors
- `OnDelete` is appropriate for cluster-critical DaemonSets where human approval is needed before each node updates
- New nodes added during a rollout always receive the latest template version — they are never left behind
- DaemonSet rollback works the same as Deployment rollback: `kubectl rollout undo daemonset/<name>`
- After updating critical DaemonSets (CNI, kube-proxy), monitor node status with `kubectl get nodes -w` and verify the application continues working

This completes Chapter 18 — DaemonSets. In Chapter 19, `01-what-is-a-job.md`, you will learn about Jobs — the Kubernetes workload for running tasks that execute once and complete.
