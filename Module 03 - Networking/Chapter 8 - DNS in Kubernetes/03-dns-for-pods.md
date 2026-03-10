# DNS for Pods

## Introduction

Kubernetes creates DNS records not only for Services but also for individual Pods. Pod DNS is less commonly used than Service DNS, but it is essential for StatefulSets (where each Pod has a stable, addressable identity) and can be valuable for debugging specific Pod instances.

This file covers Pod DNS records, how to customize Pod DNS configuration, and practical applications of Pod-level DNS.

---

## Pod DNS Record Format

By default, when a Pod has a hostname and is part of a StatefulSet with a Headless Service, it gets a DNS A record in the format:

```
<pod-name>.<service-name>.<namespace>.svc.<cluster-domain>

Example:
postgres-0.postgres-headless.production.svc.cluster.local → 10.0.1.45
postgres-1.postgres-headless.production.svc.cluster.local → 10.0.1.46
postgres-2.postgres-headless.production.svc.cluster.local → 10.0.2.67
```

For regular Pods (not StatefulSets), Kubernetes creates DNS records based on the Pod's IP address:

```
<pod-ip-with-dashes>.<namespace>.pod.cluster.local

Example:
10-0-1-45.production.pod.cluster.local → 10.0.1.45
```

The IP dashes format is rarely used directly but is important to understand — it is the format used when you see `pod.cluster.local` DNS records.

---

## Pod Hostname and Subdomain

You can give a Pod a hostname and connect it to a Service using the `hostname` and `subdomain` fields. This creates a fully qualified Pod DNS record:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  namespace: production
spec:
  hostname: web-0         # the Pod's hostname
  subdomain: web-headless # must match a Headless Service name
  containers:
  - name: web
    image: nginx:1.25
```

```yaml
# The matching Headless Service
apiVersion: v1
kind: Service
metadata:
  name: web-headless      # matches spec.subdomain in the Pod
  namespace: production
spec:
  clusterIP: None
  selector:
    app: web
  ports:
  - port: 80
```

Result — the Pod gets this DNS record:
```
web-0.web-headless.production.svc.cluster.local → (Pod IP)
```

This is exactly the mechanism that StatefulSets use internally. When you create a StatefulSet with `serviceName: web-headless`, Kubernetes automatically sets `hostname` to the Pod's ordinal name (`web-0`, `web-1`) and `subdomain` to the `serviceName`.

---

## StatefulSet Pod DNS — Deep Dive

Here is the complete DNS setup for a Kafka StatefulSet:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
  namespace: production
spec:
  clusterIP: None
  selector:
    app: kafka
  ports:
  - name: broker
    port: 9092
  - name: controller
    port: 9093
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: production
spec:
  serviceName: kafka-headless     # sets subdomain on each Pod automatically
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
        env:
        # Kafka uses Pod DNS names for broker configuration
        - name: KAFKA_BROKER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name    # "kafka-0", "kafka-1", "kafka-2"
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "PLAINTEXT://$(HOSTNAME).kafka-headless.production.svc.cluster.local:9092"
        # $(HOSTNAME) is automatically set to "kafka-0", "kafka-1", etc.
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: >
            zk-0.zookeeper-headless.production.svc.cluster.local:2181,
            zk-1.zookeeper-headless.production.svc.cluster.local:2181,
            zk-2.zookeeper-headless.production.svc.cluster.local:2181
```

Kafka brokers advertise their own DNS name. Other brokers and clients use that DNS name to connect. Since the DNS name is stable (tied to the Pod ordinal via StatefulSet), cluster membership is stable even as Pods restart.

```bash
# Verify Pod DNS records after StatefulSet creation
kubectl run dns-test --image=busybox --rm -it --restart=Never -- sh

# Inside test Pod:
nslookup kafka-0.kafka-headless.production.svc.cluster.local
# Address: 10.0.1.45

nslookup kafka-1.kafka-headless.production.svc.cluster.local
# Address: 10.0.1.46

nslookup kafka-2.kafka-headless.production.svc.cluster.local
# Address: 10.0.2.67

# What is my own hostname?
hostname
# kafka-0   (inside the kafka-0 Pod)
```

---

## Pod DNS Policy — `dnsPolicy`

The `dnsPolicy` field controls how a Pod's DNS is configured:

```yaml
spec:
  dnsPolicy: ClusterFirst    # default — use cluster DNS (CoreDNS)
```

| Policy | Behavior |
|--------|---------|
| `ClusterFirst` | Use CoreDNS for cluster DNS; forward external queries to upstream. **Default.** |
| `ClusterFirstWithHostNet` | Same as ClusterFirst but for Pods using `hostNetwork: true` |
| `Default` | Use the **node's** DNS configuration (not CoreDNS) — does not resolve Service names |
| `None` | Ignore all DNS defaults. Must provide custom `dnsConfig` |

The `Default` policy is confusingly named — it does NOT use cluster DNS. It uses the node's `/etc/resolv.conf`, which means the Pod cannot resolve Kubernetes Service names.

Use `Default` only for Pods that must resolve names like the node itself (e.g., some monitoring agents that need to reach AWS endpoints without going through CoreDNS).

---

## Custom DNS Configuration — `dnsConfig`

You can customize a Pod's DNS settings with the `dnsConfig` field:

```yaml
spec:
  dnsPolicy: None        # required when using dnsConfig fully custom
  dnsConfig:
    nameservers:
    - 172.20.0.10        # CoreDNS
    - 8.8.8.8            # Google DNS as fallback
    searches:
    - production.svc.cluster.local
    - svc.cluster.local
    - cluster.local
    - mycompany.internal  # add a custom internal domain search
    options:
    - name: ndots
      value: "5"
    - name: timeout
      value: "2"          # DNS query timeout in seconds
    - name: attempts
      value: "3"          # number of DNS query attempts before failure
```

### Adding a Custom Search Domain

A common use case: adding your corporate domain to the search list so Pods can resolve internal services by short name:

```yaml
spec:
  dnsPolicy: ClusterFirst
  dnsConfig:
    searches:
    - mycompany.internal  # appended to the default search domains
    options:
    - name: ndots
      value: "5"
```

With this, a Pod querying `internal-tool` will also try `internal-tool.mycompany.internal` — resolving internal corporate hostnames without the full domain.

### Reducing ndots for External-Heavy Workloads

If a Pod makes many external HTTP calls, the default `ndots:5` causes unnecessary DNS lookups. Reduce it for such Pods:

```yaml
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "2"    # only search domains for names with fewer than 2 dots
```

With `ndots:2`, `api.stripe.com` (2 dots) is tried as absolute first — only 1 DNS lookup instead of 5.

---

## Pod DNS for Debugging

Use Pod DNS records to connect to a specific instance of a service for debugging:

```bash
# Connect directly to kafka-1 (bypassing load balancing)
kubectl exec -it debug-pod -n production -- \
  kafka-topics.sh --bootstrap-server kafka-1.kafka-headless.production:9092 --list

# Check replication for a specific broker
kubectl exec -it debug-pod -n production -- \
  kafka-topics.sh \
    --bootstrap-server kafka-1.kafka-headless.production:9092 \
    --describe --topic my-topic

# Connect directly to postgres-0 (the primary) vs postgres-1 (a replica)
kubectl exec -it debug-pod -n production -- \
  psql -h postgres-0.postgres-headless.production -U admin -d mydb -c "SELECT pg_is_in_recovery();"
# false → postgres-0 is the primary

kubectl exec -it debug-pod -n production -- \
  psql -h postgres-1.postgres-headless.production -U admin -d mydb -c "SELECT pg_is_in_recovery();"
# true → postgres-1 is a replica
```

---

## Summary

- Regular Pods get an IP-based DNS record: `<ip-with-dashes>.<namespace>.pod.cluster.local`
- Pods in a StatefulSet with a Headless Service get stable per-Pod DNS: `<pod-name>.<service>.<namespace>.svc.cluster.local`
- StatefulSet pod DNS is the mechanism that enables Kafka, ZooKeeper, Cassandra, and PostgreSQL clusters to maintain stable member identities
- `dnsPolicy: ClusterFirst` is the default — use CoreDNS for cluster names, forward external queries upstream
- `dnsPolicy: Default` uses the node's DNS — Pods with this policy cannot resolve Kubernetes Service names
- `dnsConfig` customizes DNS: add corporate search domains, reduce `ndots` for external-heavy Pods, or add fallback nameservers
- Use per-Pod DNS to connect to specific StatefulSet instances during debugging without going through a load balancer

In the next file, `04-custom-dns.md`, you will learn how to customize CoreDNS behavior — adding stub zones for private domains, configuring rewrites, and setting up DNS for hybrid cloud environments where the cluster needs to reach on-premise services.
