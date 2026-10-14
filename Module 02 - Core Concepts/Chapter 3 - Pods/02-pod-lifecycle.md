# Pod Lifecycle

## Introduction

Every Pod passes through a series of phases from the moment you create it to the moment it is deleted. Understanding this lifecycle is essential for debugging Pod issues, writing proper health checks, and ensuring zero-downtime deployments.

This file covers every phase a Pod goes through, what happens during graceful shutdown, and what the common failure states look like and how to fix them.

---

## Pod Phases — The High-Level View

Kubernetes tracks Pod status using a `phase` field. A Pod moves through these phases in order:

```
kubectl apply -f pod.yaml
       │
       ▼
  ┌─────────┐    Scheduler finds a node      ┌──────────────┐
  │ Pending │ ─────────────────────────────► │ ContainerCreating│
  └─────────┘                                └──────┬───────┘
                                                    │ containers start
                                                    ▼
                                            ┌───────────────┐
                                            │   Running     │ ◄── steady state
                                            └───────┬───────┘
                                                    │
                              ┌─────────────────────┤
                              │                     │
                              ▼                     ▼
                       ┌───────────┐        ┌────────────┐
                       │ Succeeded │        │  Failed    │
                       └───────────┘        └────────────┘
                       (all containers      (container
                        exited with 0)       exited non-zero)
```

| Phase | Meaning |
|-------|---------|
| `Pending` | Pod created in etcd, waiting to be scheduled or for image pull |
| `Running` | Pod bound to a node, at least one container is running |
| `Succeeded` | All containers exited with code 0 (typical for Jobs) |
| `Failed` | All containers have exited, at least one with a non-zero exit code |
| `Unknown` | Pod state cannot be determined — usually a node communication problem |

---

## Phase 1: Pending

A Pod enters `Pending` state immediately after you create it. It stays `Pending` until it is scheduled to a node and its containers have started.

### Reasons a Pod Stays Pending

**Insufficient resources on all nodes:**
```bash
kubectl describe pod my-pod -n production

# Events section shows:
# Warning  FailedScheduling  0/3 nodes are available:
#   3 Insufficient cpu.
#   preemption: 0/3 nodes are available: 3 No preemption victims found.
```

This means no node has enough free CPU to run this Pod. Fix: reduce the Pod's CPU request, add more nodes, or check if other Pods are over-consuming resources.

**Image being pulled:**
```bash
# Pod is Pending because the image is not yet downloaded
kubectl get pod my-pod

# NAME      READY   STATUS             RESTARTS   AGE
# my-pod    0/1     ContainerCreating  0          45s

kubectl describe pod my-pod
# Events:
#   Normal  Pulling  45s  kubelet  Pulling image "123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v2.0"
```

For large images (500MB+), this can take 30–120 seconds on first pull. After the image is cached on the node, subsequent Pod starts are faster.

**`ImagePullBackOff` — cannot pull the image:**
```bash
kubectl get pod my-pod
# NAME      READY   STATUS             RESTARTS   AGE
# my-pod    0/1     ImagePullBackOff   0          2m

kubectl describe pod my-pod
# Events:
#   Warning  Failed  2m  kubelet  Failed to pull image: unauthorized: authentication required
```

Common causes:
- Image name or tag is wrong
- ECR repository does not exist
- Worker node IAM role lacks `ecr:GetAuthorizationToken` permission
- Wrong registry URL

**No nodes match the Pod's node selector:**
```bash
# Events:
# Warning  FailedScheduling  0/3 nodes are available:
#   3 node(s) didn't match Pod's node affinity/selector.
```

The Pod requires nodes with specific labels but no such nodes exist in the cluster.

---

## Phase 2: Running

Once the Pod is scheduled and all containers start, it enters `Running` phase. But `Running` does not mean the application is ready to serve traffic.

### Container States

Inside the `Running` phase, each individual container has its own state:

```bash
kubectl describe pod web-pod -n production

# Containers:
#   web:
#     State:          Running
#       Started:      Mon, 15 Jan 2024 10:00:05 +0000
#     Ready:          True
#     Restart Count:  0
```

Individual container states:

| Container State | Meaning |
|----------------|---------|
| `Waiting` | Container is waiting (pulling image, init containers running) |
| `Running` | Container process is executing |
| `Terminated` | Container process has exited (check exit code) |

### The Ready Condition and Readiness Probes

The `READY` column in `kubectl get pods` (e.g., `1/1`) shows how many containers are **ready** — not just running. A container is only `Ready` if its readiness probe passes.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10   # start checking after 10 seconds
  periodSeconds: 5          # check every 5 seconds
  failureThreshold: 3       # mark not ready after 3 failures
  successThreshold: 1       # mark ready after 1 success
```

Until the readiness probe passes:
- The container shows `0/1` (not ready)
- The Pod is NOT added to Service endpoints
- No traffic is routed to this Pod

This is the mechanism that prevents Kubernetes from sending traffic to a Pod that is still starting up (connecting to database, loading cache, warming up).

### The Liveness Probe

The liveness probe checks if the container is still healthy after it started:

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30   # wait 30s before starting liveness checks
  periodSeconds: 10         # check every 10 seconds
  failureThreshold: 3       # restart container after 3 consecutive failures
```

If the liveness probe fails 3 times in a row, the kubelet **restarts the container** (not the Pod — just the container inside it). The restart count increments, and you see `CrashLoopBackOff` if it keeps failing.

> ⚠️ **Warning:** Do not make your liveness probe too aggressive. If you set `periodSeconds: 1` and `failureThreshold: 1`, a single slow request will restart your container. Use `failureThreshold: 3` minimum.

### CrashLoopBackOff — The Most Common Pod Problem

```bash
kubectl get pods
# NAME      READY   STATUS             RESTARTS   AGE
# web-pod   0/1     CrashLoopBackOff   5          10m
```

`CrashLoopBackOff` means the container started, crashed (exit code non-zero), restarted, crashed again — repeatedly. Kubernetes adds an exponential backoff delay between restarts (10s → 20s → 40s → ... → max 5 minutes) to prevent thrashing.

**How to diagnose:**
```bash
# Read the logs from the crashed container (current run)
kubectl logs web-pod -n production

# Read logs from the PREVIOUS run (after container restarted)
kubectl logs web-pod -n production --previous

# Get more details
kubectl describe pod web-pod -n production

# Look at:
# - "Last State" section (previous container exit code)
# - "Events" section (what happened)
```

**Common causes:**
- Application crashes on startup (configuration error, missing environment variable)
- Application cannot connect to a dependency (database not ready)
- Out of memory (OOMKilled) — container exceeds its memory limit
- Liveness probe fails too quickly

---

## Phase 3: Termination (Graceful Shutdown)

When you delete a Pod — or Kubernetes deletes it during a rolling update, scale-down, or node eviction — the Pod goes through a graceful shutdown sequence. This is critical for zero-downtime deployments.

### The Termination Sequence

```
kubectl delete pod web-pod
         │
         ▼
Step 1: Pod enters "Terminating" state
        API Server sets deletionTimestamp on the Pod
         │
         ▼
Step 2: Pod removed from Service endpoints IMMEDIATELY
        kube-proxy updates iptables — no new traffic goes to this Pod
         │
         ▼
Step 3: Kubernetes sends SIGTERM to PID 1 in each container
        This signals the application to shut down gracefully
         │
         ▼
Step 4: terminationGracePeriodSeconds countdown begins (default: 30 seconds)
        Application should:
        - Stop accepting new requests
        - Finish processing in-flight requests
        - Close database connections
        - Flush logs
         │
         ▼
Step 5a: Application exits cleanly within 30 seconds
         Container terminates with exit code 0
         │
Step 5b: Application does NOT exit within 30 seconds
         Kubernetes sends SIGKILL — immediate forced termination
         Container killed, exit code 137
```

### Why terminationGracePeriodSeconds Matters

```yaml
spec:
  terminationGracePeriodSeconds: 30   # default is 30 seconds
  containers:
  - name: web
    image: nginx:1.25
```

Set this high enough for your application to finish its in-flight work. For a web server handling long-running requests, 60 seconds might be appropriate. For a quick job, 10 seconds is fine.

Your application must handle `SIGTERM` correctly:

```python
# Python example: handle SIGTERM for graceful shutdown
import signal
import sys
import time

def graceful_shutdown(signum, frame):
    print("SIGTERM received — finishing in-flight requests...")
    # Set a flag to stop accepting new connections
    # Wait for in-flight requests to complete
    time.sleep(5)
    print("Shutdown complete")
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
```

### The Race Condition During Shutdown

There is a known timing issue: the Pod is removed from Service endpoints (Step 2) and SIGTERM is sent (Step 3) at almost the same time. But kube-proxy on each node takes a few seconds to update its iptables rules.

During this brief window, a small amount of traffic might still reach the terminating Pod. The fix is to add a `preStop` hook that introduces a short sleep:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sleep", "5"]   # wait 5 seconds before SIGTERM
```

This 5-second delay gives kube-proxy time to update its rules on all nodes before the Pod starts shutting down. This is a common production pattern for zero-downtime deployments.

---

## Pod Conditions

In addition to the phase, Pods have conditions that give more detail about their state:

```bash
kubectl describe pod web-pod | grep -A 10 "Conditions:"

# Conditions:
#   Type              Status
#   Initialized       True    ← init containers completed
#   Ready             True    ← all containers are ready
#   ContainersReady   True    ← all containers are running
#   PodScheduled      True    ← Pod assigned to a node
```

| Condition | Meaning |
|-----------|---------|
| `PodScheduled` | Pod has been assigned to a node |
| `Initialized` | All init containers have completed |
| `ContainersReady` | All containers are running and healthy |
| `Ready` | Pod is ready to serve traffic (all containers Ready) |

When `Ready` is `False`, the Pod is NOT receiving traffic from any Service.

---

## Restart Policies

The `restartPolicy` field controls what happens when a container exits:

```yaml
spec:
  restartPolicy: Always   # default — restart container on any exit
```

| Policy | Behavior | Use Case |
|--------|----------|---------|
| `Always` | Restart on any exit (success or failure) | Web servers, APIs — always running |
| `OnFailure` | Restart only on non-zero exit code | Batch jobs — retry on failure, stop on success |
| `Never` | Never restart | One-shot tasks — run once and be done |

For Pods managed by a Deployment, `restartPolicy: Always` is always used (and is the only option).
For Jobs, use `OnFailure` or `Never`.

---

## Debugging Pod Lifecycle Issues — Reference

```bash
# Pod stuck in Pending
kubectl describe pod <pod-name> -n <namespace>
# Look at Events section — it tells you exactly why scheduling failed

# Pod in CrashLoopBackOff
kubectl logs <pod-name> -n <namespace> --previous
# Read the logs from the last crash

# Pod stuck in Terminating
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force
# Force delete a stuck terminating Pod (use carefully)

# Watch Pod status change in real time
kubectl get pods -n <namespace> --watch

# Get all Pod events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

---

## Summary

- Pods move through five phases: Pending → Running → Succeeded or Failed (or Unknown)
- `Pending` means the Pod is waiting to be scheduled or for its image to be pulled
- `Running` means containers are started but does not guarantee traffic readiness — the readiness probe controls that
- `CrashLoopBackOff` means the container keeps crashing — check logs with `--previous` flag
- Graceful termination: Kubernetes sends SIGTERM, waits `terminationGracePeriodSeconds` (default 30s), then SIGKILL
- Add a `preStop: sleep 5` hook to prevent brief traffic to terminating Pods (race condition fix)
- `restartPolicy: Always` for long-running services; `OnFailure` or `Never` for Jobs
- Readiness probes prevent traffic from reaching Pods that are not yet ready; liveness probes restart unhealthy containers

In the next file, `03-multi-container-pods.md`, you will learn the sidecar, init container, and ambassador patterns — when and why you put multiple containers in a single Pod.
