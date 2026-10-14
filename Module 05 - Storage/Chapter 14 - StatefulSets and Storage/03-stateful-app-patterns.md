# Stateful App Patterns

## Introduction

Running stateful applications on Kubernetes requires more than just attaching a PVC. Production-grade deployments of databases and message queues need careful attention to initialization order, leader election, replication setup, backup strategies, and topology-aware scheduling. This file covers production patterns for three of the most common stateful applications on EKS: PostgreSQL, Redis, and Kafka.

---

## Pattern 1: PostgreSQL on EKS

PostgreSQL running as a StatefulSet with streaming replication:

```
postgres-0 (Primary)     → accepts reads and writes
    │ streaming replication
    ├─ postgres-1 (Replica) → reads only
    └─ postgres-2 (Replica) → reads only
```

### Services — Two Services for Read/Write Separation

```yaml
# Write Service — routes to primary only
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
  namespace: production
spec:
  selector:
    app: postgres
    role: primary        # only the primary Pod has this label (set by initialization script)
  ports:
  - port: 5432

---
# Read Service — routes to all replicas
apiVersion: v1
kind: Service
metadata:
  name: postgres-replica
  namespace: production
spec:
  selector:
    app: postgres
    role: replica
  ports:
  - port: 5432

---
# Headless Service — for StatefulSet Pod DNS
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
```

### StatefulSet with Initialization

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      # Topology spread — distribute across AZs
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: postgres

      initContainers:
      # Set up primary vs replica configuration
      - name: init-postgres
        image: postgres:15
        command:
        - /bin/sh
        - -c
        - |
          if [[ "$(hostname)" == "postgres-0" ]]; then
            echo "I am the primary"
            # Primary: initialize database if not initialized
            if [ ! -f /var/lib/postgresql/data/pgdata/PG_VERSION ]; then
              initdb -D /var/lib/postgresql/data/pgdata
              echo "host replication replicator all md5" >> /var/lib/postgresql/data/pgdata/pg_hba.conf
            fi
          else
            echo "I am a replica — syncing from postgres-0"
            PGPASSWORD=$POSTGRES_PASSWORD pg_basebackup \
              -h postgres-0.postgres-headless.production.svc.cluster.local \
              -U replicator \
              -D /var/lib/postgresql/data/pgdata \
              -P -R --wal-method=stream
          fi
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
      
      containers:
      - name: postgres
        image: postgres:15
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
          limits:
            cpu: "4"
            memory: "8Gi"
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data

  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      storageClassName: gp3
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
```

### Topology-Aware Scheduling

For a 3-replica PostgreSQL cluster, spread Pods across 3 AZs:

```yaml
topologySpreadConstraints:
- maxSkew: 1                              # max difference between AZs
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule        # prefer AZ spread, block if impossible
  labelSelector:
    matchLabels:
      app: postgres
```

With `WaitForFirstConsumer` binding mode, each Pod's EBS volume is created in the AZ where the Pod is scheduled — maintaining data locality.

---

## Pattern 2: Redis on EKS

Redis in cluster mode with 3 primary shards and 3 replicas:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: production
spec:
  serviceName: redis-headless
  replicas: 6                   # 3 primaries + 3 replicas
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command:
        - redis-server
        - /etc/redis/redis.conf
        - --cluster-enabled yes
        - --cluster-config-file /data/nodes.conf
        - --cluster-node-timeout 5000
        - --appendonly yes
        - --dir /data
        ports:
        - containerPort: 6379
          name: redis
        - containerPort: 16379
          name: cluster
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
        volumeMounts:
        - name: redis-data
          mountPath: /data
        - name: redis-config
          mountPath: /etc/redis
      
      volumes:
      - name: redis-config
        configMap:
          name: redis-config
  
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      storageClassName: gp3
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi
```

Redis Cluster initialization:
```bash
# After all 6 Pods are Running, initialize the cluster
PODS=$(kubectl get pods -n production -l app=redis \
  -o jsonpath='{range .items[*]}{.status.podIP}:6379 {end}')

kubectl exec -it redis-0 -n production -- \
  redis-cli --cluster create $PODS --cluster-replicas 1 --cluster-yes
```

---

## Pattern 3: Kafka on EKS

Kafka brokers — each needs its own storage and stable DNS:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: production
spec:
  serviceName: kafka-headless
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:7.5.0
        ports:
        - containerPort: 9092
          name: broker
        env:
        - name: KAFKA_BROKER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['statefulset.kubernetes.io/pod-name']
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "PLAINTEXT://$(POD_NAME).kafka-headless.production.svc.cluster.local:9092"
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: KAFKA_LOG_DIRS
          value: "/var/kafka-data/logs"
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: "zk-0.zk-headless.production:2181,zk-1.zk-headless.production:2181,zk-2.zk-headless.production:2181"
        resources:
          requests:
            cpu: "2"
            memory: "8Gi"
          limits:
            cpu: "4"
            memory: "16Gi"
        volumeMounts:
        - name: kafka-data
          mountPath: /var/kafka-data
  
  volumeClaimTemplates:
  - metadata:
      name: kafka-data
    spec:
      storageClassName: gp3          # or 'fast' (io2) for high-throughput
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 500Gi             # Kafka needs large storage for retention
```

---

## Production Checklist for Stateful Apps

```
Storage:
[ ] volumeClaimTemplates uses correct StorageClass (gp3 for general, io2 for high IOPS)
[ ] Storage size has 30% growth headroom beyond current data size
[ ] reclaimPolicy: Retain on production StorageClass
[ ] PVC deletion protected by RBAC (only cluster-admins can delete production PVCs)

Scheduling:
[ ] topologySpreadConstraints distributes Pods across AZs
[ ] PodDisruptionBudget ensures at least N-1 Pods stay up during node maintenance
[ ] resource requests and limits set correctly (using p95 usage data)

Reliability:
[ ] Readiness probes configured (Pods not added to Services until truly ready)
[ ] Liveness probes configured (restart crashed DB process)
[ ] preStop hook or terminationGracePeriodSeconds set for graceful shutdown

Backup:
[ ] Daily VolumeSnapshot CronJob or Velero schedule configured
[ ] Snapshots stored in S3 for disaster recovery
[ ] Recovery procedure documented and tested (restore from snapshot)

Monitoring:
[ ] Prometheus metrics from application (postgres_exporter, redis_exporter, kafka_exporter)
[ ] PVC usage alerts (alert at 80% full)
[ ] Replication lag alerts (replica WAL lag > threshold)
```

---

## Summary

- PostgreSQL on EKS: use two Services (primary for writes, replica for reads) and a Headless Service for StatefulSet DNS; init container bootstraps replicas from primary using `pg_basebackup`
- Use `topologySpreadConstraints` to distribute StatefulSet Pods across AZs — critical for HA databases
- Redis cluster mode requires 6 StatefulSet replicas (3 primaries + 3 replicas) — initialize the cluster with `redis-cli --cluster create` after all Pods are Running
- Kafka brokers advertise their own pod DNS name — stable DNS from the Headless Service is what makes broker discovery work
- Production checklist: Retain reclaim policy, RBAC protection on PVCs, PodDisruptionBudget, VolumeSnapshot backup, and replication lag monitoring

This completes Module 5 — Storage. You now understand the complete Kubernetes storage stack — from temporary emptyDir volumes through dynamic EBS provisioning, StatefulSet-managed PVCs, and production patterns for real databases.

In Module 6, `01-what-is-a-deployment.md`, you will learn about Deployments — the Kubernetes workload object for running and managing stateless application replicas with zero-downtime rolling updates.
