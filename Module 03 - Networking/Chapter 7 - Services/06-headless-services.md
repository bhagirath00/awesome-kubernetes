# Headless Services

## Introduction

A Headless Service is a Service with `clusterIP: None`. Unlike a regular Service, it has no virtual IP and kube-proxy creates no iptables rules for it. Instead of routing traffic through a VIP, DNS queries for a Headless Service return the individual IP addresses of all matching Pods.

Headless Services are the essential companion for **StatefulSets** — databases, Kafka clusters, ZooKeeper, Elasticsearch. They are also used whenever your application needs to discover and connect to individual Pod instances directly, rather than through a load balancer.

---

## How Headless Services Differ from Regular Services

```
Regular Service (ClusterIP):
  DNS query: web-service.production.svc.cluster.local
  DNS response: 172.20.45.123  (single ClusterIP — load balanced)

Headless Service (clusterIP: None):
  DNS query: postgres-headless.production.svc.cluster.local
  DNS response: 10.0.1.45, 10.0.1.46, 10.0.2.67  (all Pod IPs directly)
```

With a regular Service, the client gets one stable IP and traffic is distributed by kube-proxy. With a Headless Service, the client gets all Pod IPs and decides itself which one to connect to.

---

## Creating a Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
spec:
  clusterIP: None        # this is what makes it headless
  selector:
    app: postgres
  ports:
  - name: postgres
    port: 5432
    targetPort: 5432
```

```bash
kubectl apply -f headless-service.yaml

kubectl get service postgres-headless -n production

# NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
# postgres-headless    ClusterIP   None         <none>        5432/TCP   5m
# Notice: CLUSTER-IP is "None"

# DNS returns all Pod IPs:
kubectl run dns-test --image=busybox --rm -it --restart=Never -- \
  nslookup postgres-headless.production.svc.cluster.local

# Output:
# Server:    172.20.0.10
# postgres-headless.production.svc.cluster.local  has address 10.0.1.45
# postgres-headless.production.svc.cluster.local  has address 10.0.1.46
# postgres-headless.production.svc.cluster.local  has address 10.0.2.67
```

---

## Headless Services With StatefulSets — The Critical Use Case

StatefulSets give each Pod a **stable, predictable name**: `postgres-0`, `postgres-1`, `postgres-2`. Headless Services make each individual Pod reachable by its own stable DNS name:

```
<pod-name>.<headless-service-name>.<namespace>.svc.cluster.local

postgres-0.postgres-headless.production.svc.cluster.local → 10.0.1.45
postgres-1.postgres-headless.production.svc.cluster.local → 10.0.1.46
postgres-2.postgres-headless.production.svc.cluster.local → 10.0.2.67
```

This per-Pod DNS is what makes StatefulSets work for clustered databases. Each database node needs a permanent identity to participate in cluster membership (leader election, replication group, etc.).

```yaml
# Complete StatefulSet + Headless Service setup
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
  - name: postgres
    port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless    # must match the Headless Service name
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name    # injects "postgres-0", "postgres-1", etc.
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:             # each Pod gets its own PVC
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 20Gi
```

With this setup:
- `postgres-0` always has a stable DNS name: `postgres-0.postgres-headless.production`
- `postgres-1` always has: `postgres-1.postgres-headless.production`
- Each Pod has its own dedicated PersistentVolume
- If `postgres-0` is deleted and recreated, it gets the same name, same DNS, and same PVC

---

## Per-Pod DNS Records

CoreDNS creates per-Pod A records for each Pod in a Headless Service:

```bash
# Each individual Pod has its own DNS name
kubectl run dns-test --image=busybox --rm -it --restart=Never -- \
  nslookup postgres-0.postgres-headless.production.svc.cluster.local

# Output:
# Name: postgres-0.postgres-headless.production.svc.cluster.local
# Address: 10.0.1.45    ← only postgres-0's IP

nslookup postgres-1.postgres-headless.production.svc.cluster.local
# Address: 10.0.1.46    ← only postgres-1's IP
```

This is how a new cluster member joins a Kafka or ZooKeeper cluster — it resolves `kafka-0.kafka-headless`, `kafka-1.kafka-headless`, and `kafka-2.kafka-headless` to get the current broker IPs for initial cluster contact.

---

## Headless Service Without a Selector

You can create a Headless Service with no selector. In this case, no Endpoints are created automatically — you define them manually:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-kafka
  namespace: production
spec:
  clusterIP: None
  # No selector — manual endpoints
  ports:
  - name: kafka
    port: 9092
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-kafka    # matches Service name
  namespace: production
subsets:
- addresses:
  - ip: 10.100.1.10       # external Kafka broker 1
    hostname: kafka-0     # enables kafka-0.external-kafka DNS
  - ip: 10.100.1.11
    hostname: kafka-1
  - ip: 10.100.1.12
    hostname: kafka-2
  ports:
  - port: 9092
```

This gives you the per-host DNS naming (`kafka-0.external-kafka.production`) for an externally managed Kafka cluster, with no changes required on the Kafka side.

---

## Headless Services for Service Discovery

Some applications implement their own client-side load balancing — gRPC, for example, performs connection-level load balancing. A regular Kubernetes Service would route all gRPC connections to a single Pod (because a single TCP connection carries all gRPC calls).

With a Headless Service, the gRPC client gets all Pod IPs and manages its own connection pool:

```python
# Python gRPC client using Headless Service for client-side LB
import grpc
from grpc import experimental

# Headless Service DNS returns all Pod IPs
# gRPC resolves all IPs and creates connections to each
channel = grpc.insecure_channel(
    'dns:///api-grpc-headless.production.svc.cluster.local:9090',
    options=[
        ('grpc.lb_policy_name', 'round_robin'),  # client-side round-robin
    ]
)
```

This distributes gRPC calls across all backend Pods — something a regular ClusterIP Service cannot do because it operates at the connection level.

---

## Summary

- Headless Services (`clusterIP: None`) have no virtual IP — DNS returns individual Pod IPs instead of a VIP
- StatefulSets require a Headless Service (`serviceName` field) to enable stable per-Pod DNS: `<pod-name>.<service>.<namespace>.svc.cluster.local`
- Per-Pod DNS records are how clustered databases (PostgreSQL replication, Kafka brokers, ZooKeeper nodes) discover each other and maintain stable identities
- Headless Services without selectors accept manually defined Endpoints — useful for pointing to external clusters with per-host naming
- gRPC and other connection-level load-balanced protocols benefit from Headless Services — the client resolves all Pod IPs and manages its own load balancing

This completes Chapter 7 — Services. In Chapter 8, `01-kube-dns-overview.md`, you will learn how CoreDNS works in Kubernetes, how DNS resolution flows through the cluster, and how to troubleshoot DNS failures.
