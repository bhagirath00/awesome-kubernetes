# Multi-Container Pods

## Introduction

The vast majority of Pods you create will have exactly one container. But Kubernetes supports running multiple containers in a single Pod, and there are specific, well-defined patterns where this is not only useful but the right approach.

This file covers the three main multi-container Pod patterns: **Sidecar**, **Ambassador**, and **Adapter**. You will also learn about **Init Containers**, which are a fourth pattern that runs setup containers before the main container starts.

---

## Why Multi-Container Pods Exist

Two containers in a Pod share:
- **The same network** — they communicate via `localhost`, not via Service DNS
- **The same volumes** — they can read and write the same files
- **The same lifecycle** — they start and stop together

This co-location is the key. When two processes must share files or communicate with zero latency (no network hop), putting them in the same Pod is the right design.

When two processes are independent and do not need to share files or communicate via localhost, they should be in **separate Pods** — this allows them to scale independently.

---

## Pattern 1: Sidecar

The **sidecar** pattern adds a helper container that enhances or extends the main container's functionality. The sidecar runs alongside the main container and does something the main application does not do itself.

### Example: Log Shipping Sidecar

Your application writes logs to a file. A sidecar container tails that file and ships logs to a centralized logging system (like CloudWatch Logs or Elasticsearch):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logging
  namespace: production
spec:
  containers:

  # Main container — your application
  - name: web-app
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/web-app:v1.0
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app         # app writes logs here

  # Sidecar — ships logs to CloudWatch
  - name: log-shipper
    image: amazon/aws-for-fluent-bit:latest
    env:
    - name: AWS_REGION
      value: "us-east-1"
    - name: LOG_GROUP_NAME
      value: "/production/web-app"
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app         # reads logs from same directory

  volumes:
  - name: log-volume
    emptyDir: {}                      # shared between both containers
```

Both containers mount the same `log-volume`. The web app writes to `/var/log/app/app.log`. The log-shipper reads from the same path and sends to CloudWatch. They communicate through the shared filesystem — no network calls needed.

### Example: Envoy Proxy Sidecar (Service Mesh)

Tools like Istio automatically inject an Envoy proxy sidecar into every Pod. Envoy intercepts all network traffic in and out of the Pod and handles:
- mTLS (mutual TLS encryption between services)
- Distributed tracing
- Retry logic
- Circuit breaking

```yaml
# Istio injects this automatically — shown here for understanding
spec:
  containers:
  - name: web-app          # your actual container
    image: web-app:v1.0
    ports:
    - containerPort: 8080

  - name: istio-proxy      # injected sidecar — intercepts all traffic
    image: istio/proxyv2:1.18.0
    ports:
    - containerPort: 15090  # Envoy admin port
    - containerPort: 15021  # health check port
```

The sidecar pattern is also used in Kubernetes for:
- **Secrets rotation** — a sidecar fetches updated secrets from AWS Secrets Manager and writes them to a shared volume
- **Config reloading** — a sidecar watches for ConfigMap changes and signals the main app to reload
- **Metric scraping** — a Prometheus exporter sidecar exposes metrics from an application that does not natively support Prometheus

---

## Pattern 2: Ambassador

The **ambassador** pattern uses a helper container as a proxy for communicating with external services. The main container always talks to `localhost`, and the ambassador translates and routes the traffic to the correct external endpoint.

### Example: Database Proxy Ambassador

Your application connects to a database. In production, you have a primary for writes and a read replica for reads. An ambassador (like `cloud-sql-proxy` or `rds-proxy`) handles the connection pooling and routing:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-with-db-proxy
  namespace: production
spec:
  containers:

  # Main container — always connects to localhost:5432
  - name: api-app
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/api:v1.0
    env:
    - name: DB_HOST
      value: "localhost"              # talks to the ambassador, not directly to RDS
    - name: DB_PORT
      value: "5432"

  # Ambassador — proxies connections to RDS
  - name: rds-proxy
    image: rds-proxy-sidecar:v1.0
    env:
    - name: RDS_ENDPOINT
      value: "mydb.abc123.us-east-1.rds.amazonaws.com"
    - name: RDS_PORT
      value: "5432"
    ports:
    - containerPort: 5432             # listens on localhost:5432
```

The main container does not need to know the RDS hostname or handle connection pooling — the ambassador handles it. This simplifies the main application and makes it easy to change the database endpoint without updating the app.

### Example: Redis Cluster Ambassador

When your application needs to talk to a Redis cluster (multiple shards), an ambassador like `twemproxy` or `redis-cluster-proxy` handles the cluster protocol. The main app just connects to `localhost:6379`:

```yaml
containers:
- name: app
  env:
  - name: REDIS_HOST
    value: "localhost"
  - name: REDIS_PORT
    value: "6379"

- name: redis-proxy              # ambassador handles cluster topology
  image: redis-cluster-proxy:v1
  env:
  - name: REDIS_CLUSTER_NODES
    value: "redis-0.redis:6379,redis-1.redis:6379,redis-2.redis:6379"
  ports:
  - containerPort: 6379
```

---

## Pattern 3: Adapter

The **adapter** pattern standardizes the output of the main container. The main container produces data in its own format, and the adapter transforms it into a standard format that other systems (like Prometheus or a centralized logging platform) expect.

### Example: Prometheus Metrics Adapter

An old application exposes its metrics in a non-Prometheus format. An adapter sidecar reads the app's metrics endpoint and exposes them in Prometheus format:

```yaml
spec:
  containers:

  # Main container — exposes metrics in a custom format
  - name: legacy-app
    image: legacy-app:v1.0
    ports:
    - containerPort: 8080          # app's HTTP port
    - containerPort: 9000          # custom metrics port (non-Prometheus format)

  # Adapter — converts metrics to Prometheus format
  - name: metrics-adapter
    image: prometheus-adapter:v1.0
    env:
    - name: SOURCE_URL
      value: "http://localhost:9000/metrics"   # reads from main container
    - name: ADAPTER_PORT
      value: "9090"
    ports:
    - containerPort: 9090         # Prometheus scrapes this port
```

Prometheus scrapes `9090` on the Pod IP. The adapter reads from `localhost:9000` (the legacy format) and serves it as Prometheus metrics.

---

## Containers Communicating Via localhost

All containers in a Pod share the same network namespace. This means they all have the same IP address and communicate via `localhost`:

```
Pod IP: 10.0.1.45

Container: web-app      →  listens on 10.0.1.45:8080
Container: log-shipper  →  communicates via localhost:8080 internally
Container: metrics      →  listens on 10.0.1.45:9090
```

This means:
- Container A can call Container B at `http://localhost:8080`
- They cannot both bind to the same port (port conflict)
- External traffic reaches the Pod IP, then goes to the right container's port

---

## Checking Multi-Container Pod Status

```bash
# See all containers in a Pod
kubectl get pod web-with-logging -n production

# Output:
# NAME                 READY   STATUS    RESTARTS   AGE
# web-with-logging     2/2     Running   0          10m
# READY: 2/2 means both containers are ready

# Logs for a specific container
kubectl logs web-with-logging -n production -c web-app
kubectl logs web-with-logging -n production -c log-shipper

# Exec into a specific container
kubectl exec -it web-with-logging -n production -c web-app -- bash
kubectl exec -it web-with-logging -n production -c log-shipper -- sh

# Describe shows all containers' states
kubectl describe pod web-with-logging -n production
```

---

## When to Use Multi-Container Pods vs Separate Pods

| Situation | Use Multi-Container Pod | Use Separate Pods |
|-----------|------------------------|------------------|
| Helper needs to read the same files as main app | ✅ Sidecar with shared volume | ❌ |
| Helper and main app communicate via localhost | ✅ | ❌ Network overhead |
| Two services that scale independently | ❌ | ✅ |
| Two services with different resource needs | ❌ | ✅ |
| Proxy that must intercept container traffic | ✅ Ambassador | ❌ |
| Two completely unrelated microservices | ❌ | ✅ |

The rule of thumb: **if two containers must always run together and share local resources — same Pod. If they can run independently — separate Pods.**

---

## Summary

- Multi-container Pods make sense when containers need to share files (via volumes) or communicate via localhost
- The **Sidecar** pattern adds a helper that enhances the main container — log shipping, secret rotation, Envoy proxy injection
- The **Ambassador** pattern uses a proxy sidecar to abstract external service communication — database proxies, Redis cluster proxies
- The **Adapter** pattern transforms the main container's output into a standard format — Prometheus metrics adapters for legacy apps
- All containers in a Pod share the same IP address and communicate via `localhost` — they cannot use the same port
- `kubectl logs -c <container-name>` and `kubectl exec -c <container-name>` let you work with specific containers in a multi-container Pod
- If two processes can scale independently, they should be in separate Pods — not forced into a single Pod

In the next file, `04-pod-yaml-structure.md`, you will get a complete, field-by-field reference of the Pod YAML spec — covering every important field with explanations and examples.
