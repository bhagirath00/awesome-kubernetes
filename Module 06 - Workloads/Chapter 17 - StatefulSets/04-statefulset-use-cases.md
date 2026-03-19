# StatefulSet Use Cases

## Introduction

StatefulSets solve a specific class of problem: running distributed, stateful systems on Kubernetes. Understanding the real-world use cases — why each application needs stable identity, persistent storage, and ordered operations — helps you recognize when a StatefulSet is the right tool.

---

## Use Case 1: Relational Databases (PostgreSQL, MySQL)

PostgreSQL with streaming replication is the canonical StatefulSet use case:

```
postgres-0 → PRIMARY (read + write)
postgres-1 → REPLICA (read only)
postgres-2 → REPLICA (read only)
```

Why StatefulSet:
- **Per-Pod data**: Each node has its own data directory (primary and replicas are different)
- **Stable DNS**: Replication configuration uses `postgres-0.postgres-headless` as the primary address
- **Ordered startup**: Pod 0 (primary) must start and initialize before replicas connect

```bash
# Primary accepts writes
kubectl exec postgres-0 -n production -- \
  psql -U postgres -c "INSERT INTO orders VALUES (1, 'test');"

# Replica returns data synced from primary
kubectl exec postgres-1 -n production -- \
  psql -U postgres -c "SELECT * FROM orders;"
# (1, "test")  ← replica has the data

# Direct DNS-based connection to primary
psql -h postgres-0.postgres-headless.production.svc.cluster.local -U postgres
```

---

## Use Case 2: Message Queues (Kafka)

Kafka brokers require stable identity because:
- Kafka uses broker ID to identify nodes in the cluster
- Producers and consumers connect to specific broker hostnames for partition leadership
- Each broker stores its own partition replicas

```yaml
spec:
  serviceName: kafka-headless
  replicas: 3
  template:
    spec:
      containers:
      - name: kafka
        env:
        - name: KAFKA_BROKER_ID_COMMAND
          value: "hostname | rev | cut -d- -f1 | rev"  # ordinal from hostname
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "PLAINTEXT://$(POD_NAME).kafka-headless.production:9092"
        # ↑ Each broker advertises its own stable DNS hostname to clients
```

When `kafka-1` restarts:
- New Pod has the same hostname: `kafka-1`
- Same DNS: `kafka-1.kafka-headless.production.svc.cluster.local`
- Same data (PVC reattaches): partition replicas are intact
- Kafka controller recognizes broker-1 is back — resumes partition assignment

---

## Use Case 3: Distributed Consensus (ZooKeeper, etcd)

ZooKeeper requires an odd number of nodes (3 or 5) and each node needs a unique, stable ID for quorum:

```yaml
spec:
  serviceName: zk-headless
  replicas: 3
  template:
    spec:
      initContainers:
      - name: init-zookeeper
        image: busybox
        command:
        - /bin/sh
        - -c
        - |
          # Set ZooKeeper myid from Pod ordinal
          ORDINAL=$(hostname | rev | cut -d- -f1 | rev)
          echo $((ORDINAL + 1)) > /var/lib/zookeeper/data/myid
          # zk-0 → myid=1, zk-1 → myid=2, zk-2 → myid=3
        volumeMounts:
        - name: data
          mountPath: /var/lib/zookeeper/data
      
      containers:
      - name: zookeeper
        image: zookeeper:3.8
        env:
        - name: ZOO_SERVERS
          value: "server.1=zk-0.zk-headless.production:2888:3888 server.2=zk-1.zk-headless.production:2888:3888 server.3=zk-2.zk-headless.production:2888:3888"
```

When any ZooKeeper node restarts, it comes back with the same `myid` and the same ZooKeeper state (from the PVC). The quorum recognizes it and allows it to rejoin without reconfiguration.

---

## Use Case 4: Search and Analytics (Elasticsearch)

Elasticsearch nodes manage shards — pieces of the index distributed across nodes. Each node needs stable identity for shard allocation:

```yaml
spec:
  serviceName: elasticsearch-headless
  replicas: 3
  template:
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
        env:
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name   # "elasticsearch-0", "elasticsearch-1"
        - name: cluster.initial_master_nodes
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: discovery.seed_hosts
          value: "elasticsearch-headless"   # resolves to all Pod IPs via DNS
        resources:
          requests:
            cpu: "2"
            memory: "8Gi"
          limits:
            cpu: "4"
            memory: "16Gi"
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      storageClassName: gp3
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 500Gi   # each node needs significant storage for index data
```

---

## Use Case 5: Distributed Caching (Redis Cluster)

Redis Cluster uses consistent hashing to distribute keys across 3 primary shards, each with a replica:

```yaml
spec:
  serviceName: redis-cluster-headless
  replicas: 6    # 3 primaries + 3 replicas
  template:
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
        - --cluster-announce-hostname
        - $(POD_NAME).redis-cluster-headless.production.svc.cluster.local
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        volumeMounts:
        - name: redis-data
          mountPath: /data
  
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

`cluster-announce-hostname` makes Redis advertise its stable Pod DNS name — clients can reconnect after a Pod restart using the same hostname.

---

## StatefulSet Maintenance Operations

```bash
# Drain a StatefulSet node for maintenance (like node drain but for the app)
# Step 1: Scale down from the top (highest ordinal)
kubectl scale statefulset elasticsearch --replicas=2 -n production
# elasticsearch-2 is removed — Elasticsearch moves its shards to remaining nodes

# Verify cluster health (no unassigned shards)
kubectl exec elasticsearch-0 -n production -- \
  curl -s localhost:9200/_cluster/health | python3 -m json.tool | grep status
# "status": "green"   ← all shards assigned to remaining nodes

# Scale back up
kubectl scale statefulset elasticsearch --replicas=3 -n production
# elasticsearch-2 rejoins — cluster rebalances shards

# Force leader election to a specific Pod (e.g., after maintenance)
kubectl exec kafka-0 -n production -- \
  kafka-leader-election.sh --bootstrap-server localhost:9092 \
  --topic my-topic --partition 0 --election-type PREFERRED
```

---

## Summary

- PostgreSQL: per-Pod data directories + stable DNS for replication — primary always Pod 0, replicas connect by hostname
- Kafka: stable broker ID (from ordinal) + advertised hostname (from Pod DNS) — brokers advertise their stable hostname to clients
- ZooKeeper: `myid` file from Pod ordinal + stable quorum member addresses — quorum membership survives Pod restarts
- Elasticsearch: `node.name` from Pod metadata + discovery via Headless Service DNS — each node manages its own shard data
- Redis Cluster: `cluster-announce-hostname` uses stable Pod DNS — clients can reconnect after Pod restart using the same hostname
- Maintenance: scale down from the highest ordinal (replicas first, primary last) to allow data migration before node removal

This completes Chapter 17 — StatefulSets. In Chapter 18, `01-what-is-a-daemonset.md`, you will learn about DaemonSets — the workload that runs exactly one Pod on every node in the cluster, used for system-level agents like log collectors and monitoring tools.
