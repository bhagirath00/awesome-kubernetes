# ExternalName Service

## Introduction

An ExternalName Service is a special Service type that creates a **DNS alias** to an external hostname. Instead of routing traffic to Pods, it returns a CNAME record pointing to an external DNS name. This allows Pods inside the cluster to reference external services (like AWS RDS, ElastiCache, or a third-party API) using a stable Kubernetes Service name.

---

## The Problem ExternalName Solves

Imagine your application connects to an AWS RDS database:

```python
# Hard-coded in the application:
DB_HOST = "mydb.abc123.us-east-1.rds.amazonaws.com"
```

This creates two problems:
1. The RDS endpoint is embedded in application config — changing the database endpoint requires an application change and redeployment
2. The endpoint is environment-specific — your dev database has a different hostname than production

ExternalName solves this by giving the external service a stable, environment-agnostic Kubernetes name:

```python
# Uses the Service name — works identically in dev, staging, and production
DB_HOST = "postgres-db.production.svc.cluster.local"
# In production, this resolves to the production RDS endpoint
# In staging, the staging ExternalName Service resolves to the staging RDS endpoint
```

---

## Creating an ExternalName Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-db
  namespace: production
spec:
  type: ExternalName
  externalName: mydb.abc123.us-east-1.rds.amazonaws.com
  # No selector — ExternalName does not route to Pods
  # No ports — port handling is transparent (the external service handles it)
```

```bash
kubectl apply -f externalname-service.yaml

kubectl get service postgres-db -n production

# NAME          TYPE           CLUSTER-IP   EXTERNAL-IP                              PORT(S)   AGE
# postgres-db   ExternalName   <none>       mydb.abc123.us-east-1.rds.amazonaws.com  <none>    5m
```

---

## How ExternalName Works — DNS CNAME

When a Pod queries `postgres-db.production.svc.cluster.local`, CoreDNS does not return an IP. Instead, it returns a **CNAME record** pointing to the external hostname:

```bash
# From inside a Pod — DNS resolution
nslookup postgres-db.production.svc.cluster.local

# Output:
# Server:    172.20.0.10
# Address:   172.20.0.10#53
#
# postgres-db.production.svc.cluster.local  canonical name = mydb.abc123.us-east-1.rds.amazonaws.com
# Name:    mydb.abc123.us-east-1.rds.amazonaws.com
# Address: 10.0.3.45    ← actual RDS IP resolved from the CNAME
```

The application connects to `postgres-db` — DNS resolves it to the RDS CNAME, which then resolves to the RDS IP. The application never needs to know the actual RDS endpoint.

---

## Common ExternalName Use Cases

### AWS RDS Database

```yaml
# production namespace
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: production
spec:
  type: ExternalName
  externalName: myapp-prod.abc123.us-east-1.rds.amazonaws.com
---
# staging namespace — same Service name, different RDS endpoint
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: staging
spec:
  type: ExternalName
  externalName: myapp-staging.def456.us-east-1.rds.amazonaws.com
```

Application config in both namespaces:
```
DB_HOST=database  # or database.production.svc.cluster.local
```

The same connection string works in production and staging — the ExternalName Service routes to the right RDS instance for each environment.

### AWS ElastiCache Redis

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-cache
  namespace: production
spec:
  type: ExternalName
  externalName: myapp-redis.abc123.ng.0001.use1.cache.amazonaws.com
```

### Third-Party API with Changing IPs

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-gateway
  namespace: production
spec:
  type: ExternalName
  externalName: api.stripe.com
```

If Stripe changes their API hostname, update the ExternalName Service — no application change needed.

---

## Updating an ExternalName — Zero-Downtime Migration

ExternalName makes database migrations transparent to the application:

```bash
# Current state: application reaches old RDS instance via 'database' Service
# ExternalName: old-db.abc123.us-east-1.rds.amazonaws.com

# After database migration is complete, update the ExternalName:
kubectl patch service database -n production \
  --type='merge' \
  -p='{"spec":{"externalName":"new-db.def456.us-east-1.rds.amazonaws.com"}}'

# DNS TTL is 30 seconds by default for cluster DNS
# Within 30 seconds, all Pods will resolve 'database' to the new RDS instance
# No Pod restarts, no application changes, no redeployment
```

This is a powerful migration pattern. You can migrate the database without touching application configuration or triggering redeployments.

---

## ExternalName Limitations

### No Port Mapping

ExternalName is a pure DNS redirect — it does not perform port mapping. The application must use the port that the external service expects:

```python
# ExternalName does NOT translate ports
# If RDS listens on 5432, the application must connect on 5432
DB_CONNECTION = "postgres-db:5432/mydb"   # works — reaches RDS port 5432 directly
DB_CONNECTION = "postgres-db:80/mydb"     # fails — RDS is not listening on port 80
```

### TLS Considerations

If the external service uses TLS (which RDS does by default), the application must connect with TLS enabled and trust the external service's certificate. The ExternalName Service does not terminate or forward TLS — it is a pure DNS redirect.

```python
# Python connection to RDS via ExternalName Service with TLS
import psycopg2
conn = psycopg2.connect(
    host="postgres-db",          # ExternalName Service name
    port=5432,
    database="myapp",
    user="admin",
    password="...",
    sslmode="require"            # RDS requires TLS
)
```

### CNAME Chain Depth

ExternalName creates a CNAME. Some clients have limits on CNAME chain depth. AWS ELB DNS names often produce a chain (CNAME → CNAME → A record). Most modern clients handle chains up to 8 hops.

---

## ExternalName vs Manual Endpoints

An alternative to ExternalName is creating a Service with no selector and manually defining Endpoints:

```yaml
# Service without selector
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: production
spec:
  ports:
  - port: 5432
    targetPort: 5432
---
# Manually define the endpoints (specific IP — not DNS)
apiVersion: v1
kind: Endpoints
metadata:
  name: external-db      # must match the Service name exactly
  namespace: production
subsets:
- addresses:
  - ip: 10.100.1.45     # specific IP of the external service
  ports:
  - port: 5432
```

This approach routes to a specific IP (not DNS). Use manual Endpoints when:
- The external service does not have a stable DNS name
- You need port translation (ExternalName does not support this)
- You want to gradually migrate traffic from an external service to an in-cluster Service

---

## Summary

- ExternalName creates a DNS CNAME alias — Pods use a stable Service name (`database`) that resolves to an external hostname (`mydb.abc123.us-east-1.rds.amazonaws.com`)
- ExternalName has no ClusterIP, no selector, and no port mapping — it is a pure DNS redirect
- Common use cases: AWS RDS databases, ElastiCache Redis, third-party APIs — anything external that benefits from a stable internal name
- Same Service name (`database`) in different namespaces pointing to different external endpoints makes configuration portable across environments
- Updating the ExternalName changes DNS resolution within the TTL — enables zero-downtime database migrations without redeployments
- Limitation: no port mapping, no TLS handling — the application must connect using the external service's actual port and handle TLS itself

In the next file, `06-headless-services.md`, you will learn about Headless Services — which bypass the ClusterIP entirely and return individual Pod IPs via DNS, making them the essential companion for StatefulSets.
