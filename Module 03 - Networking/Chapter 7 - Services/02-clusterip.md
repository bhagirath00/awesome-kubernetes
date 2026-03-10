# ClusterIP Service

## Introduction

ClusterIP is the default Service type. When you create a Service without specifying a type, it becomes a ClusterIP Service. It gives the Service a virtual IP address that is only reachable **from inside the cluster** — no external traffic can reach it directly.

ClusterIP is the foundation for internal service-to-service communication in a microservices architecture. Every database, cache, message queue, and internal API should be exposed as a ClusterIP Service.

---

## How ClusterIP Works

When you create a ClusterIP Service, Kubernetes allocates an IP address from the cluster's Service CIDR range. This IP is assigned to the Service permanently and does not change:

```bash
kubectl get service web-service -n production

# NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# web-service   ClusterIP   172.20.45.123   <none>        80/TCP    5m

# The CLUSTER-IP (172.20.45.123) is assigned from the Service CIDR
# It never changes even if you update the Service
# EXTERNAL-IP is <none> — this Service is not reachable from outside
```

---

## Creating a ClusterIP Service

```yaml
# web-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: production
  labels:
    app: web-server
spec:
  type: ClusterIP       # default — can be omitted
  selector:
    app: web-server     # routes to all Pods with this label
  ports:
  - name: http
    port: 80            # Service listens on port 80
    targetPort: 8080    # forwards to Pod port 8080
    protocol: TCP
```

```bash
kubectl apply -f web-service.yaml

# Test from inside the cluster (exec into any Pod)
kubectl exec -it api-pod-abc123 -n production -- \
  wget -O- http://web-service:80
# Works because api-pod is in the same namespace

kubectl exec -it api-pod-abc123 -n production -- \
  wget -O- http://web-service.production.svc.cluster.local:80
# Full DNS name — works from any namespace
```

---

## Multi-Port Services

A single Service can expose multiple ports — useful for services that have both HTTP and gRPC endpoints, or HTTP and metrics:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  selector:
    app: api-server
  ports:
  - name: http          # name is required when multiple ports are defined
    port: 80
    targetPort: 8080
  - name: grpc
    port: 9090
    targetPort: 9090
  - name: metrics
    port: 9091
    targetPort: 9091    # Prometheus scrapes this port
```

Referencing a named port from another resource:

```yaml
# Readiness probe can reference port by name
readinessProbe:
  httpGet:
    path: /ready
    port: http          # references the 'http' port name in the container spec
```

---

## Named targetPort — Decoupling Service from Container

Instead of using port numbers, reference container port names in targetPort:

```yaml
# Container defines named ports
spec:
  containers:
  - name: web
    image: web-app:v1.0
    ports:
    - name: http-port    # name the port
      containerPort: 8080

---
# Service references the port by name
spec:
  ports:
  - port: 80
    targetPort: http-port   # references the container port name
```

This decouples the Service from the container's port number. If the container switches from port 8080 to 8000, you only update the container spec — the Service does not change because it references the name, not the number.

---

## Specifying a Static ClusterIP

By default, Kubernetes assigns a random IP from the Service CIDR. You can request a specific IP:

```yaml
spec:
  type: ClusterIP
  clusterIP: 172.20.0.10    # request a specific IP from the Service CIDR
  selector:
    app: coredns
```

This is useful for well-known services that other systems need to hard-code (like CoreDNS). Most of the time, let Kubernetes assign the IP automatically.

```bash
# Find out what your Service CIDR is (cannot request an IP outside this range)
kubectl cluster-info dump | grep service-cluster-ip-range

# Output:
# "--service-cluster-ip-range=172.20.0.0/16"
```

---

## ClusterIP: None — Headless Services

Setting `clusterIP: None` creates a **Headless Service** — it has no ClusterIP and kube-proxy does not create any iptables rules for it. Instead, DNS returns the individual Pod IPs directly:

```yaml
spec:
  clusterIP: None     # headless — no virtual IP
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

```bash
# DNS lookup for a headless Service returns all Pod IPs
nslookup postgres-headless.production.svc.cluster.local

# Output:
# Name: postgres-headless.production.svc.cluster.local
# Address: 10.0.1.45    ← Pod 1 IP directly
# Address: 10.0.1.46    ← Pod 2 IP directly
# Address: 10.0.2.67    ← Pod 3 IP directly

# For a regular ClusterIP Service, DNS returns only the ClusterIP:
nslookup web-service.production.svc.cluster.local
# Name: web-service.production.svc.cluster.local
# Address: 172.20.45.123   ← ClusterIP only
```

Headless Services are covered in depth in `06-headless-services.md`. They are the standard way to expose StatefulSets (databases, Kafka, ZooKeeper).

---

## Common ClusterIP Use Cases

### Internal API Communication

```yaml
# The orders service reaches the payments service via ClusterIP
apiVersion: v1
kind: Service
metadata:
  name: payments-service
  namespace: production
spec:
  selector:
    app: payments
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

```python
# In the orders service code:
import requests
# Uses short Service DNS name within same namespace
response = requests.post("http://payments-service/api/v1/charge", json=payload)
```

### Database Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: production
spec:
  selector:
    app: postgres
    role: primary             # only route to the primary (not read replicas)
  ports:
  - name: postgres
    port: 5432
    targetPort: 5432
```

```bash
# Connection string in application config:
# DB_URL=postgresql://postgres-service.production.svc.cluster.local:5432/mydb
```

### Redis Cache Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: production
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

---

## ClusterIP vs Direct Pod Access — Performance

ClusterIP routing through iptables/IPVS adds a small overhead (microseconds) compared to direct Pod IP access. For the vast majority of applications this is completely negligible.

The stability and dynamic load balancing benefits far outweigh the minimal performance cost. Always use Services, not direct Pod IPs, for any production application communication.

---

## Summary

- ClusterIP is the default Service type — it assigns a stable virtual IP accessible only from inside the cluster
- Pods reach ClusterIP Services by name: `<service-name>` (same namespace) or `<service-name>.<namespace>` (cross-namespace)
- Multi-port Services require each port to have a `name` field
- Use named `targetPort` to decouple the Service port from the container's port number — changing the container port only requires updating the container spec
- You can request a specific ClusterIP from the Service CIDR range — useful for well-known internal services
- `clusterIP: None` creates a Headless Service — DNS returns individual Pod IPs instead of a VIP — used for StatefulSets
- Always use ClusterIP Services for internal communication — never hard-code Pod IPs in application config

In the next file, `03-nodeport.md`, you will learn about NodePort Services — which expose traffic externally via a port on every worker node and when this approach is appropriate.
