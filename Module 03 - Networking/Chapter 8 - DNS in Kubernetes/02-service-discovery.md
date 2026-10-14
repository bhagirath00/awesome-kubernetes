# Service Discovery in Kubernetes

## Introduction

Service discovery is how applications find each other in a dynamic environment where Pod IPs change constantly. In Kubernetes, service discovery is built on DNS — every Service gets a DNS name, and applications connect using that name rather than hard-coded IPs.

This file covers exactly how Kubernetes DNS-based service discovery works, the full DNS name hierarchy, cross-namespace patterns, and how to structure your application configuration for reliable service discovery.

---

## The DNS Naming Convention for Services

Every Service in Kubernetes gets a DNS name following this pattern:

```
<service-name>.<namespace>.svc.<cluster-domain>

cluster-domain is typically: cluster.local
```

So a Service named `api-server` in the `production` namespace gets:

```
api-server.production.svc.cluster.local
```

This is the **fully qualified domain name (FQDN)**. You can also use shorthand names depending on where the query comes from.

---

## The Four DNS Name Forms — From Long to Short

Kubernetes DNS supports four forms of Service names, each valid in different contexts:

```bash
# FQDN — always works from anywhere in the cluster
curl http://api-server.production.svc.cluster.local:8080

# Short form 1 — service + namespace (works from any namespace)
curl http://api-server.production:8080

# Short form 2 — service name only (works within the same namespace)
curl http://api-server:8080

# DO NOT use IP addresses directly — Pods are ephemeral
curl http://10.0.1.45:8080   # breaks when Pod is replaced
```

The short forms work because of the `search` directive in `/etc/resolv.conf`. When a Pod in namespace `payments` queries `api-server`, the resolver tries:
1. `api-server.payments.svc.cluster.local` (same namespace)
2. `api-server.svc.cluster.local`
3. `api-server.cluster.local`
4. `api-server.ec2.internal` (AWS VPC domain)

If the Service is in `payments` namespace, step 1 resolves it immediately.

---

## DNS Lookup Behavior — ndots:5

The `ndots:5` setting in `/etc/resolv.conf` has a subtle but important effect:

```
If the query has fewer than 5 dots → search domains are appended first
If the query has 5 or more dots → treat as absolute first, then search
```

```bash
# "api-server" has 0 dots → search domains appended
# Tries: api-server.production.svc.cluster.local → found!

# "api.external-co.com" has 2 dots → still appended (2 < 5)
# Tries: api.external-co.com.production.svc.cluster.local → NOT FOUND
# Tries: api.external-co.com.svc.cluster.local → NOT FOUND
# Tries: api.external-co.com.cluster.local → NOT FOUND
# Tries: api.external-co.com.ec2.internal → NOT FOUND
# Tries: api.external-co.com → FOUND (absolute)
# Total: 5 DNS lookups instead of 1!

# "api.external-co.com." (trailing dot) → absolute name, 0 lookups for search domains
# Tries: api.external-co.com → FOUND immediately
```

For external hostnames used frequently, add a trailing dot to avoid unnecessary search domain lookups:

```python
# Python requests — trailing dot forces absolute DNS lookup
import requests
# 5 lookups (ndots:5 behavior):
response = requests.get("https://api.external-company.com/v1/data")

# 1 lookup (trailing dot = absolute FQDN):
response = requests.get("https://api.external-company.com./v1/data")
# Note: trailing dot in hostname — forces absolute DNS resolution
```

In practice, the extra lookups add only a few milliseconds. For performance-critical code making thousands of external HTTP calls per second, this matters. For most applications, it does not.

---

## Service Discovery Patterns in Application Code

### Pattern 1: Environment Variables (Legacy — Not Recommended)

Kubernetes automatically injects environment variables for every Service into every Pod at Pod start time:

```bash
kubectl exec web-pod-abc123 -- env | grep API_SERVER

# API_SERVER_SERVICE_HOST=172.20.45.123    ← ClusterIP
# API_SERVER_SERVICE_PORT=80
# API_SERVER_PORT=tcp://172.20.45.123:80
# API_SERVER_PORT_80_TCP=tcp://172.20.45.123:80
# API_SERVER_PORT_80_TCP_PROTO=tcp
# API_SERVER_PORT_80_TCP_PORT=80
# API_SERVER_PORT_80_TCP_ADDR=172.20.45.123
```

The variable name format is: `<SERVICE_NAME_UPPERCASE>_SERVICE_HOST` and `<SERVICE_NAME_UPPERCASE>_SERVICE_PORT`.

**Why this pattern is not recommended:**
- Services must exist before the Pod starts — if you create the Service after the Pod, the env vars are not set
- There is a limit to how many env vars can be injected (large clusters with hundreds of Services hit this)
- The Pod must be restarted if the ClusterIP changes (rare but possible)

### Pattern 2: DNS (Recommended — The Standard Approach)

Use DNS names in your application configuration. DNS is dynamic — no restart needed when cluster state changes:

```python
# Python application — database connection using DNS
import psycopg2

DB_HOST = os.getenv("DB_HOST", "postgres-service.production.svc.cluster.local")
DB_PORT = int(os.getenv("DB_PORT", "5432"))

conn = psycopg2.connect(
    host=DB_HOST,
    port=DB_PORT,
    database="myapp",
    user="admin",
    password=os.getenv("DB_PASSWORD")
)
```

```yaml
# Application Deployment — Service name in env var
spec:
  containers:
  - name: api
    env:
    - name: DB_HOST
      value: "postgres-service"              # same namespace → short name works
    - name: REDIS_HOST
      value: "redis-service"
    - name: PAYMENTS_API
      value: "payments-service.payments"    # different namespace → include namespace
```

---

## Cross-Namespace Service Discovery

When Service A in namespace `frontend` needs to reach Service B in namespace `backend`:

```yaml
# Service B is in the backend namespace
metadata:
  name: api-service
  namespace: backend
```

```yaml
# Service A (in frontend namespace) connects to Service B
spec:
  containers:
  - name: frontend
    env:
    - name: API_HOST
      value: "api-service.backend"    # must include namespace
      # or full FQDN:
      # value: "api-service.backend.svc.cluster.local"
```

```bash
# Test cross-namespace connectivity
kubectl exec -it frontend-pod -n frontend -- \
  wget -O- http://api-service.backend.svc.cluster.local:8080/healthz

# Or using the short cross-namespace form:
kubectl exec -it frontend-pod -n frontend -- \
  wget -O- http://api-service.backend:8080/healthz
```

---

## Service Discovery for Databases — Recommended Pattern

For databases, use a two-Service pattern — one for writes (primary) and one for reads (read replicas):

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
    role: primary             # only the primary Pod has this label
  ports:
  - port: 5432
    targetPort: 5432
---
# Read Service — routes to all read replicas
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
    targetPort: 5432
```

```yaml
# Application uses both Services
env:
- name: DB_WRITE_HOST
  value: "postgres-primary"     # all writes go here
- name: DB_READ_HOST
  value: "postgres-replica"     # all reads go here (load balanced across replicas)
```

---

## Debugging Service Discovery Issues

```bash
# Step 1: Can the Pod reach CoreDNS at all?
kubectl exec -it web-pod -n production -- \
  nslookup kubernetes.default.svc.cluster.local

# If this fails: CoreDNS is down or Pod's DNS config is wrong

# Step 2: Can the Pod resolve its own namespace's Services?
kubectl exec -it web-pod -n production -- \
  nslookup web-service.production.svc.cluster.local

# Step 3: Can the Pod resolve cross-namespace Services?
kubectl exec -it web-pod -n production -- \
  nslookup api-service.backend.svc.cluster.local

# Step 4: Can the Pod reach external DNS?
kubectl exec -it web-pod -n production -- \
  nslookup google.com

# Step 5: What DNS server is the Pod using?
kubectl exec -it web-pod -n production -- cat /etc/resolv.conf

# Step 6: Check CoreDNS logs for errors
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Common error patterns:
# [ERROR] plugin/errors: 2 web-service.prod.svc.cluster.local. NXDOMAIN
# → The Service doesn't exist or the namespace is wrong

# Step 7: Check if the Service exists and has endpoints
kubectl get service web-service -n production
kubectl get endpoints web-service -n production
```

---

## Summary

- Every Service gets a DNS name: `<service>.<namespace>.svc.cluster.local`
- Short names work within the same namespace (`api-server`); include the namespace for cross-namespace (`api-server.backend`)
- The `ndots:5` resolver behavior means queries with fewer than 5 dots go through search domains first — external hostnames trigger up to 5 lookups; add a trailing dot to force absolute resolution
- Use DNS for service discovery (not environment variables) — DNS is dynamic and does not require Pod restarts
- Two-Service database pattern: `postgres-primary` for writes, `postgres-replica` for reads — use labels to target the right Pods
- Debug DNS failures: check CoreDNS is running, resolve `kubernetes.default`, check `/etc/resolv.conf`, view CoreDNS logs

In the next file, `03-dns-for-pods.md`, you will learn the DNS records that Kubernetes creates for Pods themselves — how individual Pods get DNS names and how to use Pod DNS for troubleshooting and StatefulSet member discovery.
