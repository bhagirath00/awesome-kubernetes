# What is a Service

## Introduction

A Pod's IP address changes every time the Pod is recreated. You cannot build a reliable application that hard-codes Pod IPs — they are ephemeral by design. A **Service** solves this by providing a stable network endpoint that never changes, regardless of which Pods are running behind it.

A Service is a Kubernetes resource that selects a group of Pods using label selectors and provides a permanent, stable address (ClusterIP) to reach them. It also load-balances traffic across all matching Pods automatically.

---

## The Problem Services Solve

Without Services, connecting microservices is fragile:

```
API Server (10.0.1.45) → connects to Database Pod (10.0.2.67)

Database Pod crashes → Kubernetes creates a new Database Pod at 10.0.2.99

API Server still tries 10.0.2.67 → connection refused
API Server must somehow discover the new IP → no built-in mechanism
```

With a Service:

```
API Server → connects to "database-service" (172.20.10.5) → always works
             ClusterIP never changes
             Service routes to whichever database Pods are currently running
             
Database Pod at 10.0.2.67 crashes → replaced with Pod at 10.0.2.99
Service endpoints update automatically
API Server keeps using "database-service:5432" → still works
```

The API Server never needs to know the current Pod IP. It only needs to know the Service name and port.

---

## The Four Service Types

Kubernetes has four Service types, each serving a different traffic scenario:

| Type | Traffic Source | Use Case |
|------|---------------|---------|
| `ClusterIP` | Inside the cluster only | Service-to-service communication |
| `NodePort` | External, via node IPs | Development, on-premise setups |
| `LoadBalancer` | External, via cloud LB | Production AWS/cloud deployments |
| `ExternalName` | Inside cluster → external DNS | Reaching external services by stable name |

There is also a fifth variant — **Headless Services** (ClusterIP: None) — for StatefulSets and direct Pod DNS access.

---

## The Service Resource

A Service is a YAML object like any other Kubernetes resource:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: production
  labels:
    app: web-server
    environment: production
spec:
  type: ClusterIP            # ClusterIP | NodePort | LoadBalancer | ExternalName
  
  selector:                  # finds Pods with these labels
    app: web-server
    environment: production
  
  ports:
  - name: http               # port name (optional but good practice)
    port: 80                 # port clients use to reach the Service
    targetPort: 8080         # port on the Pod to forward traffic to
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
  
  sessionAffinity: None      # None (random) or ClientIP (sticky sessions)
```

```bash
kubectl apply -f service.yaml

kubectl get service web-service -n production

# NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
# web-service   ClusterIP   172.20.45.123   <none>        80/TCP,443/TCP   5m
```

---

## How a Service Finds Its Pods — Endpoints

A Service does not directly store Pod IPs. Instead, an **Endpoints** object (automatically created and managed by the Endpoints Controller) tracks the current Pod IPs that match the Service's selector:

```bash
# See the Endpoints for a Service
kubectl get endpoints web-service -n production

# NAME          ENDPOINTS                                AGE
# web-service   10.0.1.45:8080,10.0.1.46:8080,10.0.2.67:8080  5m

# Watch endpoints update in real time as Pods start/stop
kubectl get endpoints web-service -n production -w
```

The lifecycle:
1. A Pod with matching labels becomes `Ready` (passes readiness probe)
2. Endpoints Controller adds the Pod's IP:port to the Endpoints object
3. kube-proxy reads the updated Endpoints and updates iptables rules
4. New traffic is now routed to the new Pod

When a Pod fails its readiness probe:
1. Pod is marked `NotReady`
2. Endpoints Controller removes its IP from the Endpoints object
3. kube-proxy removes the iptables rule for that endpoint
4. No new traffic is sent to that Pod

---

## Service vs Pod Direct Access

```bash
# Access a Pod directly by IP (fragile — IP changes)
curl http://10.0.1.45:8080/healthz

# Access via Service (stable — always works)
curl http://web-service.production.svc.cluster.local:80/healthz
curl http://web-service.production:80/healthz   # within same cluster
curl http://web-service:80/healthz              # within same namespace
```

Within the same namespace, you only need the Service name. Across namespaces, use `<service>.<namespace>`. The full DNS name is always `<service>.<namespace>.svc.cluster.local`.

---

## Port Forwarding — Testing Services Locally

During development, use `kubectl port-forward` to reach a Service or Pod from your local machine without exposing it externally:

```bash
# Forward local port 8080 to Service port 80
kubectl port-forward service/web-service -n production 8080:80

# Now in another terminal:
curl http://localhost:8080/healthz
# Reaches the Service (and through it, one of the Pods)

# Forward directly to a specific Pod (bypasses Service load balancing)
kubectl port-forward pod/web-pod-abc123 -n production 8080:8080

# Useful for debugging a specific Pod without affecting the load balancer
```

---

## Session Affinity — Sticky Sessions

By default, each connection is routed to a random Pod. For stateful applications that need requests from the same client to always go to the same Pod, use `sessionAffinity: ClientIP`:

```yaml
spec:
  selector:
    app: web-server
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800    # sticky for 3 hours (default: 10800)
  ports:
  - port: 80
    targetPort: 8080
```

With `ClientIP` affinity, all connections from the same client IP go to the same Pod for the duration of the timeout window.

> ⚠️ **Warning:** Session affinity based on ClientIP breaks when clients are behind a NAT gateway or load balancer (many clients share one IP). For proper sticky sessions, use application-level session tokens or a load balancer with cookie-based stickiness.

---

## Creating Services — Imperative vs Declarative

```bash
# Create a Service imperatively (quick but not recommended for production)
kubectl expose deployment web-deployment \
  --name=web-service \
  --port=80 \
  --target-port=8080 \
  --type=ClusterIP \
  -n production

# Declarative (recommended — store in Git)
kubectl apply -f service.yaml -n production

# Update a Service
kubectl apply -f updated-service.yaml -n production

# Delete a Service
kubectl delete service web-service -n production
```

---

## Debugging Services

```bash
# Step 1: Is the Service there?
kubectl get service web-service -n production

# Step 2: Does it have endpoints?
kubectl get endpoints web-service -n production
# If ENDPOINTS shows <none>, the selector doesn't match any ready Pods

# Step 3: Check selector vs Pod labels
kubectl get service web-service -n production -o yaml | grep -A 5 selector:
kubectl get pods -n production --show-labels | grep web-server

# Step 4: Are the Pods actually ready?
kubectl get pods -n production -l app=web-server
# If READY shows 0/1, readiness probe is failing

# Step 5: Check the readiness probe failure reason
kubectl describe pod web-pod-abc123 -n production | grep -A 10 "Readiness"

# Step 6: Test direct Pod access (bypass Service)
kubectl exec -it test-pod -n production -- \
  wget -T 5 http://10.0.1.45:8080/healthz

# Step 7: Test Service access
kubectl exec -it test-pod -n production -- \
  wget -T 5 http://web-service.production.svc.cluster.local:80/healthz
```

The most common Service issue: **selector mismatch**. The Service's `selector` does not match the Pod's `labels`. Always check both with `kubectl get service -o yaml` and `kubectl get pods --show-labels`.

---

## Summary

- A Service provides a stable ClusterIP that never changes, regardless of which Pods are running behind it — solving the problem of ephemeral Pod IPs
- Services use **label selectors** to find matching Pods — any Pod with matching labels automatically receives traffic
- The **Endpoints** object tracks the current IPs of ready Pods — it is automatically updated when Pods start, stop, or fail readiness checks
- Four Service types: ClusterIP (internal), NodePort (external via node), LoadBalancer (external via cloud LB), ExternalName (DNS alias to external service)
- Within the same namespace: use Service name. Across namespaces: `<service>.<namespace>`. Full: `<service>.<namespace>.svc.cluster.local`
- Use `kubectl port-forward` to test Services locally during development
- Debug Services: check endpoints, verify selector matches Pod labels, verify Pods are Ready

In the next file, `02-clusterip.md`, you will go deep into ClusterIP — the default Service type, how it works inside the cluster, and when to use it.
