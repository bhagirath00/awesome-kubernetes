# Pod Disruption Budgets

## What is a Pod Disruption Budget?

Protect your applications during maintenance:

```
You want to drain a node to do maintenance.
But your database replicaset only has 1 pod running!

If you drain, database goes down.

Solution: Pod Disruption Budget (PDB)
├─ Defines minimum pods that must stay running
├─ Blocks drain if it would violate budget
└─ Ensures application stays available
```

---

## Pod Disruption vs Eviction

```
Pod Disruption = Voluntary shutdown
├─ Cluster admin initiates drain
├─ Node needs maintenance
├─ Can be prevented by PDB

Pod Crash = Involuntary failure
├─ Node dies
├─ Pod runs out of memory
├─ Node reboots
├─ Cannot be prevented by PDB
```

---

## Basic PDB

### Scenario: Protect Database Pods

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: database-pdb
  namespace: databases
spec:
  minAvailable: 2        # At least 2 pods must stay running
  selector:
    matchLabels:
      app: postgres      # Applies to pods with this label
```

### How It Works

```
Deployment has 3 postgres pods:
├─ postgres-0 (healthy)
├─ postgres-1 (healthy)
└─ postgres-2 (healthy)

You drain a node. PDB says "at least 2 must run"

Pod postgres-0 is on the draining node.

Drain tries to evict postgres-0.
PDB checks: "Will 2 remain if evict?"
├─ If YES → Allow eviction
├─ If NO  → Block eviction
   └─ Drain hangs until pod can be safely evicted
```

---

## Types of PDB Policies

### Policy 1: minAvailable

Minimum pods that must stay running:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2     # At least 2 pods must run
  selector:
    matchLabels:
      app: web-app    # For pods with label app=web-app
```

If you have 4 pods, PDB allows disruption of 2 (keeps 2 running).

### Policy 2: maxUnavailable

Maximum pods that can go down:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: cache-pdb
spec:
  maxUnavailable: 1   # At most 1 pod can be unavailable
  selector:
    matchLabels:
      app: redis
```

If you have 3 redis pods, PDB allows disruption of 1 (keeps 2 running).

---

## Percentage-Based PDB

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 50%   # Keep 50% of pods running
  selector:
    matchLabels:
      app: api-server
```

If 10 pods running:
- 50% = 5 pods must stay
- Can disrupt 5 pods
- Drain allows up to 5 pods to be evicted

---

## Real-World Examples

### Example 1: Database (Strict Protection)

```yaml
# PostgreSQL should almost never have downtime
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
  namespace: databases
spec:
  minAvailable: 2       # Must have at least 2 of 3 pods
  selector:
    matchLabels:
      app: postgres
      tier: database
```

### Example 2: Web API (Moderate Protection)

```yaml
# API can tolerate losing some pods
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
  namespace: production
spec:
  maxUnavailable: 2     # At most 2 pods can go down
  selector:
    matchLabels:
      app: api
      component: server
```

### Example 3: Job Workers (Flexible)

```yaml
# Background workers are resilient
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: worker-pdb
  namespace: workers
spec:
  minAvailable: 1       # Keep at least 1 running
  selector:
    matchLabels:
      app: job-worker
```

---

## PDB with Drainin

### Scenario: Drain with PDB

```bash
# You have 3 API pods
# PDB says: minAvailable=2

kubectl get pods -l app=api

# NAME            READY   STATUS    RESTARTS   AGE
# api-pod-1       1/1     Running   0          10m
# api-pod-2       1/1     Running   0          5m
# api-pod-3       1/1     Running   0          2m

# Two pods on worker-1, one on worker-2

# Drain worker-1
kubectl drain worker-1 --ignore-daemonsets

# What happens:
# Drain tries to evict api-pod-1
# PDB checks: "If I remove api-pod-1, will 2 still run?"
# ├─ Remaining: api-pod-2 (still on worker-2) + api-pod-3
# ├─ Total: 2 pods
# ├─ Required: 2 pods
# └─ Decision: ALLOW eviction ✓

# api-pod-1 is evicted, reschedules to worker-2

# Drain tries to evict api-pod-3 (also on worker-1)
# PDB checks: "If I remove api-pod-3, will 2 still run?"
# ├─ Remaining: api-pod-2 only
# ├─ Total: 1 pod
# ├─ Required: 2 pods
# └─ Decision: BLOCK eviction ❌

# Drain waits
# Eventually api-pod-1 finishes starting on worker-2
# Now there are 3 running again: api-pod-1, api-pod-2, api-pod-3

# Drain tries again to evict api-pod-3
# PDB checks: "If I remove api-pod-3, will 2 still run?"
# ├─ Remaining: api-pod-1, api-pod-2
# ├─ Total: 2 pods
# ├─ Required: 2 pods
# └─ Decision: ALLOW eviction ✓

# Result: Drain completes, always keeping 2 pods running
```

---

## Checking PDB Status

```bash
# List all PDBs
kubectl get pdb --all-namespaces

# Output:
# NAMESPACE     NAME             MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
# production    api-pdb          2               <unset>           1
# databases     postgres-pdb     2               <unset>           0
# workers       worker-pdb       1               <unset>           2

# ALLOWED DISRUPTIONS shows how many pods can be safely disrupted NOW

# Get details
kubectl describe pdb api-pdb -n production

# Output:
# Name:             api-pdb
# Namespace:        production
# Min Available:    2
# Allowed Disruptions: 1
# Current Healthy:  3
# Desired Healthy:  2
```

---

## PDB with Stateful Apps

### Scenario: Database Replica Set

```yaml
# PostgreSQL with 3 replicas
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
spec:
  minAvailable: 2        # 2 of 3 must stay up
  selector:
    matchLabels:
      app: postgres
```

Why 2 instead of 3?

```
3 replicas: Consensus system
├─ Need majority (2/3) to be healthy
├─ Losing 1 is OK (2 remain)
├─ Losing 2 would block commits
└─ PDB minAvailable=2 ensures majority
```

---

## Common Mistakes

### Mistake 1: PDB Too Strict

```yaml
# WRONG: Blocks all drains
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 10    # But only 3 pods deployed!
  selector:
    matchLabels:
      app: myapp
```

Result: `kubectl drain` fails, can't do maintenance.

### Mistake 2: PDB Too Loose

```yaml
# WRONG: Doesn't protect app
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: database-pdb
spec:
  maxUnavailable: 3   # All pods can go down!
  selector:
    matchLabels:
      app: postgres
```

Result: Database goes offline during drain.

### Mistake 3: Wrong Selector

```yaml
# WRONG: Selector matches no pods
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api-server  # But pods labeled "app=api"
      tier: backend    # And not "tier=backend"
```

Result: PDB does nothing, pods get disrupted.

---

## Best Practices

1. **For Every Important App**
   - Create a PDB
   - Set appropriate minAvailable or maxUnavailable

2. **Size PDB Appropriately**
   - Database: minAvailable = replicas - 1
   - API: minAvailable = floor(replicas / 2)
   - Workers: maxUnavailable = 50%

3. **Test PDB**
   ```bash
   # Check if drain would succeed
   kubectl drain worker-1 \
     --ignore-daemonsets \
     --dry-run=client
   ```

4. **Monitor Allowed Disruptions**
   ```bash
   # Before draining, check:
   kubectl get pdb
   
   # ALLOWED DISRUPTIONS should be > 0
   ```

---

## Summary

Pod Disruption Budgets:

**Purpose:**
- Protect apps during maintenance
- Ensure minimum availability
- Prevent unintended downtime

**Types:**
- minAvailable: keep X pods running
- maxUnavailable: allow Y pods down
- Percentage or absolute numbers

**How It Works:**
- Blocks drain if would violate budget
- Allows drain when budget permits
- Checks before each pod eviction

**Best Practice:**
- Create PDB for important apps
- Test with drain --dry-run
- Monitor ALLOWED DISRUPTIONS

**Next:** File 03 covers graceful shutdown.
