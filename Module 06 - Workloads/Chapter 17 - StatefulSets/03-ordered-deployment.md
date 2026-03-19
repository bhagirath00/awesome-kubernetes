# Ordered Deployment

## Introduction

Ordered deployment is one of the most important features that separates StatefulSets from Deployments. When a StatefulSet starts, scales, or updates, it follows a strict order — each Pod must be Running and Ready before the next one starts. When scaling down or deleting, Pods are terminated in reverse order.

This ordering is what makes clustered databases and distributed systems reliable on Kubernetes.

---

## OrderedReady — The Default Policy

With `podManagementPolicy: OrderedReady` (the default):

**Scale up (0 → 3):**
```
kubectl scale statefulset kafka --replicas=3

1. Create kafka-0
   → Wait for kafka-0 to be Running AND Ready (readiness probe passes)
   → kafka-0 is Ready ✅

2. Create kafka-1
   → Wait for kafka-1 to be Ready
   → kafka-1 is Ready ✅

3. Create kafka-2
   → Wait for kafka-2 to be Ready
   → kafka-2 is Ready ✅

Done — all 3 Pods Running
```

**Scale down (3 → 1):**
```
kubectl scale statefulset kafka --replicas=1

1. Terminate kafka-2
   → Wait for kafka-2 to be fully terminated (Pod object deleted)
   → kafka-2 gone ✅

2. Terminate kafka-1
   → Wait for kafka-1 to be fully terminated
   → kafka-1 gone ✅

Done — only kafka-0 Running
```

Why reverse order for scale-down? Because in most clustered systems, the highest-ordinal node is the most recently added replica — and typically the "least important" in the cluster membership. Removing from the top ensures the primary/leader (index 0) is the last to go.

---

## Initialization with Init Containers

For clustered applications, Pod 0 often needs to initialize as a primary, while Pods 1+ initialize as replicas connecting to Pod 0. Init containers handle this:

```yaml
spec:
  template:
    spec:
      initContainers:
      # Determine this Pod's role and configure accordingly
      - name: init-role
        image: busybox:1.35
        command:
        - /bin/sh
        - -c
        - |
          # Read the Pod ordinal from the hostname
          ORDINAL=$(hostname | rev | cut -d- -f1 | rev)
          
          if [ "$ORDINAL" -eq "0" ]; then
            echo "I am Pod 0 — initializing as PRIMARY"
            touch /etc/app/is-primary
            echo "primary" > /etc/app/role
          else
            echo "I am Pod $ORDINAL — waiting for PRIMARY to be ready"
            # Wait until Pod 0 is responsive
            until nc -z kafka-0.kafka-headless.production.svc.cluster.local 9092; do
              echo "Waiting for kafka-0..."
              sleep 2
            done
            echo "replica" > /etc/app/role
            echo "PRIMARY is ready — starting as REPLICA"
          fi
        volumeMounts:
        - name: init-config
          mountPath: /etc/app
      
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:7.5.0
        env:
        # Use the Pod ordinal as the broker ID
        - name: KAFKA_BROKER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name    # "kafka-0", "kafka-1", etc.
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: data
          mountPath: /var/kafka-data
        - name: init-config
          mountPath: /etc/app
      
      volumes:
      - name: init-config
        emptyDir: {}
```

Because of `OrderedReady`, when `kafka-1`'s init container runs, `kafka-0` is guaranteed to already be Running and Ready. The init container's `nc -z` check will succeed quickly.

---

## Readiness Probes — The Gate for Ordered Startup

Readiness probes are the gate that controls when the next Pod in the sequence starts. Until a Pod passes its readiness probe, the StatefulSet controller waits:

```yaml
containers:
- name: postgres
  readinessProbe:
    exec:
      command:
      - /bin/sh
      - -c
      - |
        pg_isready -U postgres && \
        psql -U postgres -c "SELECT 1;" > /dev/null 2>&1
    initialDelaySeconds: 10
    periodSeconds: 5
    failureThreshold: 6      # wait up to 30 seconds before marking Not Ready
    successThreshold: 1
```

This probe checks:
1. Is the PostgreSQL process running? (`pg_isready`)
2. Can we actually execute a query? (`SELECT 1`)

Both must pass before the Pod is considered Ready — and before the next Pod in the StatefulSet starts.

---

## Termination Grace Period — Clean Shutdown

When a StatefulSet Pod is terminated, it receives `SIGTERM`. The Pod has `terminationGracePeriodSeconds` to complete in-flight operations before it is forcefully killed:

```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 120    # 2 minutes for PostgreSQL to flush and close
      
      containers:
      - name: postgres
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                # Stop accepting new connections
                psql -U postgres -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle';"
                # Execute checkpoint (flush dirty pages to disk)
                psql -U postgres -c "CHECKPOINT;"
                echo "PostgreSQL graceful shutdown complete"
```

For databases, `terminationGracePeriodSeconds: 120` gives the process time to:
1. Reject new connections
2. Wait for active transactions to complete
3. Flush WAL to disk
4. Write a clean shutdown marker

Without this, a forced kill leaves the database in an inconsistent state requiring crash recovery on the next start.

---

## Pausing a StatefulSet Rollout

During an update, you may want to pause after the first Pod updates to validate it before continuing:

```bash
# Start the update
kubectl set image statefulset/postgres postgres=postgres:16 -n production

# Watch updates in progress
kubectl get pods -n production -l app=postgres -w

# postgres-2 updated (v16 running) — pause before postgres-1 updates
kubectl patch statefulset postgres -n production \
  -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":1}}}}'
# partition: 1 means "only update Pods with ordinal >= 1"
# postgres-2 (ordinal 2) = already updated
# postgres-1 (ordinal 1) = will now also update
# postgres-0 (ordinal 0) = paused at partition boundary

# After postgres-1 updates and is stable, continue to primary
kubectl patch statefulset postgres -n production \
  -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
# Now postgres-0 (primary) updates
```

---

## Handling a Stuck StatefulSet

If a Pod is stuck not becoming Ready, the entire StatefulSet update halts:

```bash
# Check which Pod is stuck
kubectl get pods -n production -l app=kafka
# NAME      READY   STATUS             RESTARTS
# kafka-0   1/1     Running            0    ← ok
# kafka-1   0/1     Running            0    ← stuck Not Ready — StatefulSet waits here
# kafka-2   1/1     Running            0    ← not yet updated

# Check why kafka-1 is not Ready
kubectl describe pod kafka-1 -n production | grep -A 10 "Readiness"
# Readiness probe failed: dial tcp 127.0.0.1:9092: connection refused
# ← Kafka process is not accepting connections yet

kubectl logs kafka-1 -n production | tail -20
# ERROR: cannot connect to ZooKeeper — leader election in progress

# Wait for it to resolve naturally (ZooKeeper election completing)
# OR: if the Pod is broken, manually delete it — StatefulSet recreates it
kubectl delete pod kafka-1 -n production
```

---

## Summary

- `OrderedReady` policy (default): Pods start in order 0 → 1 → 2, each waiting for the previous to be Ready; terminate in reverse 2 → 1 → 0
- Init containers enable role detection — Pod 0 initializes as primary; Pods 1+ wait for Pod 0 then join as replicas
- Readiness probes are the gate: the next Pod only starts after the current Pod's readiness probe succeeds
- Set generous `terminationGracePeriodSeconds` for databases (120s+) and use `preStop` hooks to trigger graceful shutdown procedures
- Use `partition` to pause StatefulSet updates mid-way for validation — set `partition: N` to pause at ordinal N
- If a Pod is stuck Not Ready and blocking the rollout, delete the stuck Pod — the StatefulSet recreates it, which may resolve the issue

In the next file, `04-statefulset-use-cases.md`, you will see the complete use case gallery for StatefulSets — when to use them, what applications benefit most, and real production examples.
