# Default Deny Policy

## Introduction

By default, Kubernetes allows any pod to reach any other pod. This is terrible for security.

The **default-deny pattern** is the best practice for network security: **block everything by default, then explicitly allow what you need.**

This is a fundamental security principle called "fail-closed" or "deny by default."

---

## The Principle

```
Security without policies:
  Default: ALLOW ← Bad, assumes everything is safe

Security with default-deny:
  Default: DENY
  Exceptions: ALLOW (explicit list)
  ← Good, assumes everything is untrusted
```

Fail-closed is more secure because:
- New pods are secure by default
- Admin must explicitly allow flows
- Forgotten policies are safe (denial)
- Mistakes tend toward safety

---

## Basic Default-Deny

Block all ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  # No ingress rules = no traffic allowed
```

Block all egress:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  # No egress rules = no traffic allowed
```

Block both:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}          # All pods
  policyTypes:
  - Ingress
  - Egress
  # No rules = no traffic
```

---

## Implementing Default-Deny

### Step 1: Apply Default-Deny Policy

```bash
# In production namespace
kubectl apply -f default-deny-all.yaml -n production

# Now all pods are blocked from each other
```

### Step 2: Test (Everything Should Be Blocked)

```bash
# Frontend tries to reach backend
kubectl exec frontend-pod -n production -- curl backend:8080
# Timeout (blocked by default-deny)
```

### Step 3: Add Allow Rules

```yaml
# Allow frontend to receive from ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-frontend
spec:
  podSelector:
    matchLabels:
      tier: frontend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080

---
# Allow frontend to call backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
spec:
  podSelector:
    matchLabels:
      tier: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# Allow all pods to DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### Step 4: Test (Only Allowed Traffic)

```bash
# Frontend to backend (now allowed)
kubectl exec frontend-pod -n production -- curl backend:8080
# Success!

# Frontend to database (NOT allowed)
kubectl exec frontend-pod -n production -- curl database:5432
# Timeout (blocked, not in any allow rule)
```

---

## Complete Example: Production-Ready

```yaml
---
# 1. DEFAULT-DENY all traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# 2. ALLOW: Internet → Frontend (via Ingress Controller)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
    - protocol: TCP
      port: 443

---
# 3. ALLOW: Frontend → Backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# 4. ALLOW: Backend → Database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-to-database
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432

---
# 5. ALLOW: Backend → Cache
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-to-cache
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: cache
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 6379

---
# 6. ALLOW: All → DNS (needed for name resolution)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
# 7. ALLOW: Monitoring → All pods (for metrics collection)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: production
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090
```

Result:

```
Traffic allowed:
✓ Ingress Controller → Frontend (HTTP/HTTPS)
✓ Frontend → Backend (HTTP)
✓ Backend → Database (SQL)
✓ Backend → Cache (Redis)
✓ All → DNS (UDP 53)
✓ Monitoring → Prometheus metrics

Traffic denied:
✗ Frontend → Database (direct access blocked)
✗ Frontend → Cache (direct access blocked)
✗ Database → Backend (reverse direction denied)
✗ Any pod → External internet (blocked)
✗ Any pod → Other namespaces (blocked)
```

---

## Migration Strategy

If you have an existing cluster with no Network Policies, migrating to default-deny must be done carefully.

### Phase 1: Audit (1-2 weeks)

```yaml
# Use audit mode (log violations, allow traffic)
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/audit: restricted
```

Leave existing traffic running. Collect data on what pods communicate.

Tools to analyze traffic:
```bash
# Check API server audit logs
grep "NetworkPolicy" /var/log/audit.log

# Use network monitoring (Calico, Cilium)
kubectl logs -n calico-system deployment/calico-typha | grep "deny"
```

### Phase 2: Warn (1-2 weeks)

```yaml
# Add namespace labels to warn
labels:
  networking.networking.k8s.io/warn: restricted
```

Pods still run but developers see warnings.

### Phase 3: Enforce in New Namespace

Create new namespaces with default-deny:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production-v2
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

New deployments use production-v2.

### Phase 4: Migrate Old Pods

Gradually move pods from old namespace to new namespace with policies.

### Phase 5: Enforce Everywhere

Once comfortable, enforce in all namespaces.

---

## Common Patterns with Default-Deny

### Pattern 1: Three-Tier Application

```
Ingress → Frontend
Frontend → Backend
Backend → Database
All → DNS
Monitoring → All (metrics)
```

### Pattern 2: Microservices

```
Gateway → All services
Service1 → Database
Service2 → Cache
Service3 → External API
All → DNS
Monitoring → All
```

### Pattern 3: Multi-Team Cluster

```
Team A namespace:
  App A1 → App A2 (only)
  App A2 → Database A (only)

Team B namespace:
  App B1 → App B2 (only)
  App B2 → Database B (only)

No cross-team communication
All → DNS
Monitoring → All namespaces
```

---

## Troubleshooting Default-Deny

### Problem: Pod Can't Reach Service

```
Frontend → Backend fails
Error: Connection timeout
```

**Debugging:**

1. Check if default-deny is applied:
```bash
kubectl get networkpolicies -n production
# Should see default-deny-all
```

2. Add an allow rule:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
spec:
  podSelector:
    matchLabels:
      tier: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
```

3. Test again:
```bash
kubectl exec frontend-pod -- curl backend:8080
# Should work now
```

### Problem: DNS Doesn't Work

```
Pod can't resolve domain names
nslookup google.com
Timeout
```

**Reason:** DNS egress not allowed.

**Fix:** Allow DNS egress:
```yaml
egress:
- to:
  - namespaceSelector:
      matchLabels:
        name: kube-system
  ports:
  - protocol: UDP
    port: 53
```

### Problem: Prometheus Can't Collect Metrics

```
Prometheus timeout on pod metrics
Error: connection refused
```

**Reason:** Monitoring namespace can't reach pods.

**Fix:** Allow ingress from monitoring:
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: monitoring
  ports:
  - protocol: TCP
    port: 9090
```

---

## Best Practices

1. **Always use default-deny in production**
   ```
   Default: deny everything
   Explicit: allow only what's needed
   ```

2. **Label everything**
   ```yaml
   labels:
     tier: frontend
     app: payment
     env: production
   ```

3. **Document your policies**
   ```yaml
   # This policy allows frontend pods to
   # call backend pods on port 8080 only
   ```

4. **Test thoroughly**
   ```bash
   # Test allowed traffic
   kubectl exec frontend -- curl backend:8080
   # Test blocked traffic
   kubectl exec frontend -- curl database:5432
   ```

5. **Monitor violations**
   - Enable audit logging
   - Watch for connection timeouts
   - Review logs regularly

---

## Summary

Default-deny is the security best practice:

**Principle:**
- Default: DENY everything
- Exceptions: ALLOW specific flows
- Fail-closed (denying is safer than allowing)

**Implementation:**
1. Apply default-deny policy
2. Test (everything blocked)
3. Add allow rules per flow
4. Test (only allowed traffic passes)

**Result:**
- New pods secure by default
- No forgotten rules (safer)
- Compromised pods limited
- Prevents lateral movement

In the next file, `05-network-policy-examples.md`, we will see complete real-world examples of network policies.
