# ArgoCD Sync Policies, Waves, and Resource Hooks

## 1. Architectural Concept & Deep Dive

The synchronization engine in ArgoCD governs how and when desired states from Git are applied to the live cluster. 

Beyond simply applying manifests, enterprise deployments require deterministic lifecycle coordination:
- Database schema migrations must complete *before* new application pods launch.
- ConfigMaps and CRDs must be created *before* the controllers that depend on them.
- Resources deleted from Git must be pruned safely without taking down dependencies.
- Manual changes (`kubectl edit`) made directly in production must be automatically reverted (**Self-Healing**).

```
+─────────────────────────────────────────────────────────────────────────+
|                 ARGOCD SYNC PHASES & SYNC WAVES TIMELINE                |
|                                                                         |
|  [ PHASE 1: PreSync ]                                                   |
|  Wave -2: Execute DB Backup Job                                         |
|  Wave -1: Execute Flyway / Liquibase DB Schema Migration Job            |
|         │                                                               |
|         ▼ (Waits for Job to complete successfully!)                     |
|  [ PHASE 2: Sync ]                                                      |
|  Wave 0:  Deploy ConfigMaps, Secrets, RBAC                              |
|  Wave 1:  Deploy Deployments, StatefulSets, Services                    |
|  Wave 2:  Deploy Ingress, Monitoring ServiceMonitors                    |
|         │                                                               |
|         ▼ (Waits for all pods to report Healthy status!)                |
|  [ PHASE 3: PostSync ]                                                  |
|  Wave 3:  Execute Slack Notification / Smoke Test Probe Job             |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Automated Sync Policies: Prune & Self-Heal

By default, ArgoCD operates in manual mode. In production GitOps, configure **Automated Sync**:

```yaml
spec:
  syncPolicy:
    automated:
      prune: true      # Delete resources from K8s if removed from Git
      selfHeal: true   # Revert out-of-band manual changes made on the cluster
      allowEmpty: false # Prevent accidental deletion of all resources if Git dir is blank
```

### 1. The Prune Directive (`prune: true`)
When an engineer deletes `ingress.yaml` from Git:
- If `prune: false`, the Ingress continues running on the cluster indefinitely (**orphan resource**).
- If `prune: true`, ArgoCD deletes the Ingress from the cluster automatically.

### 2. The Self-Heal Directive (`selfHeal: true`)
If an engineer runs `kubectl scale deployment payment --replicas=10` directly in the terminal:
- ArgoCD detects drift between Git (`replicas: 3`) and the live cluster (`replicas: 10`).
- With `selfHeal: true`, ArgoCD automatically scales the deployment back to `3` within seconds, enforcing Git as the absolute single source of truth.

---

## 3. Essential Production Sync Options (`syncOptions`)

ArgoCD provides advanced synchronization flags configured under `syncPolicy.syncOptions`:

```yaml
spec:
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
      - PruneLast=true
      - ServerSideApply=true
```

| Sync Option | Mechanism & Purpose | Production Benefit |
| :--- | :--- | :--- |
| **`CreateNamespace=true`** | Automatically provisions target namespace if missing. | Eliminates manual namespace creation steps. |
| **`ApplyOutOfSyncOnly=true`**| Applies only modified resources rather than applying every YAML file in the repository. | Drastically speeds up sync passes on large apps (> 100 resources) and reduces API server rate limits. |
| **`PruneLast=true`** | Deletes obsolete resources at the very end of the sync pass, after new resources are healthy. | **Zero Downtime**. Prevents dropping traffic while new pods are still spinning up. |
| **`ServerSideApply=true`** | Uses Kubernetes Server-Side Apply instead of client-side `kubectl apply`. | **Mandatory** for massive manifests (> 264KB, e.g. Prometheus CRDs) exceeding annotation size limits. |

---

## 4. Sync Waves: Controlled Multi-Tier Orchestration

**Sync Waves** order the creation of resources using the annotation `argocd.argoproj.io/sync-wave`:
- Waves can be negative or positive integers (`... -2, -1, 0, 1, 2 ...`).
- ArgoCD applies wave `N`, **waits until all resources in wave `N` report Healthy**, and only then proceeds to wave `N+1`.

### Example: Ordered Infrastructure Rollout
```yaml
# 1. ConfigMap applied first (Wave 0)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  annotations:
    argocd.argoproj.io/sync-wave: "0"
---
# 2. Deployment applied second, guaranteed config exists (Wave 1)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  annotations:
    argocd.argoproj.io/sync-wave: "1"
---
# 3. Ingress applied last, once pods are verified healthy (Wave 2)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    argocd.argoproj.io/sync-wave: "2"
```

---

## 5. Resource Hooks: PreSync, PostSync, and SyncFail

Resource Hooks execute ephemeral jobs at specific points in the synchronization lifecycle using the annotation `argocd.argoproj.io/hook`:

| Hook Annotation | Execution Trigger | Production Use Case |
| :--- | :--- | :--- |
| `PreSync` | Before any manifests in the application are applied. | Database schema migrations, preliminary backup snapshot. |
| `Sync` | Executed synchronously alongside standard manifests. | Complex custom orchestration tasks. |
| `PostSync` | After all manifests are applied and report Healthy. | Slack deployment announcements, integration smoke tests. |
| `SyncFail` | Triggered exclusively if the sync pass fails. | PagerDuty alerting, rollbacks, automated ticket filing. |

### Example: Database Migration PreSync Hook
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-schema-migration
  annotations:
    # 1. Execute during PreSync
    argocd.argoproj.io/hook: PreSync
    # 2. Automatically delete the Job pod once migration succeeds
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: mycompany/db-migrations:v1.4
          command: ["./migrate", "--env=prod"]
```

---

## 6. Verification & Troubleshooting Runbook

### Monitoring Sync Operations
```bash
# View active sync progress and wave transitions in real time
argocd app sync order-api --watch

# Terminate an ongoing or hanging sync operation
argocd app terminate-op order-api

# Inspect failures in PreSync or PostSync hooks
kubectl get jobs -n production -l argocd.argoproj.io/hook
```

### Common Pitfalls & Solutions
1. **Sync stuck indefinitely in `PreSync` phase**:
   - Cause: Migration Job crashed or pod is stuck in `ImagePullBackOff`.
   - Solution: ArgoCD will not proceed to wave 0 until PreSync jobs exit `0`. Inspect Job logs: `kubectl logs job/db-schema-migration -n production`.
2. **CrashLoopBackOff caused by missing prune**:
   - Cause: Old service was renamed in Git, but old deployment still tries to connect to the previous service name.
   - Solution: Enable `syncPolicy.automated.prune: true` and configure `PruneLast=true`.

---

## 7. Key Takeaways
- Always configure `automated.prune: true` and `automated.selfHeal: true` to enforce true GitOps compliance.
- Use `ServerSideApply=true` for large CRDs to bypass Kubernetes 264KB annotation limits.
- Sync Waves (`sync-wave`) enforce strict ordering of manifests (e.g. ConfigMaps -> Deployments -> Ingresses).
- PreSync hooks automate database schema migrations before new workload code is exposed to traffic.
