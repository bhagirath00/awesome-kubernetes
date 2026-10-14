# Network Policy Examples

## Introduction

Now you understand Network Policies in theory. This file shows you complete, real-world examples you can copy and use.

Each example is production-ready and follows security best practices.

---

## Example 1: Simple Three-Tier Web Application

```yaml
---
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: webapp
  labels:
    name: webapp

---
# 1. DEFAULT-DENY: Block all traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: webapp
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# 2. Allow internet to frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internet-to-frontend
  namespace: webapp
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
# 3. Allow frontend to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: webapp
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
# 4. Allow backend to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: webapp
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
# 5. Allow all pods to DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: webapp
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
# Deployment example
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: app
        image: frontend:latest
        ports:
        - containerPort: 8080
```

This setup:
- Blocks all traffic by default
- Frontend receives from internet
- Frontend calls backend
- Backend calls database
- All pods can reach DNS
- No other communication allowed

---

## Example 2: Microservices with Multiple Services

```yaml
---
# Default-deny
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
# Gateway to all services
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gateway-to-all
  namespace: production
spec:
  podSelector:
    matchExpressions:
    - key: tier
      operator: In
      values: [service1, service2, service3, payment]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: gateway
    ports:
    - protocol: TCP
      port: 8080

---
# Service1 to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: service1-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: service1
    ports:
    - protocol: TCP
      port: 5432

---
# Service2 to cache
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: service2-to-cache
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: cache
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: service2
    ports:
    - protocol: TCP
      port: 6379

---
# Service3 to external payment API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: service3-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: payment
  egress:
  # To Stripe API
  - to:
    - ipBlock:
        cidr: 151.101.1.1/32
    ports:
    - protocol: TCP
      port: 443
  # To database
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
# Allow DNS for all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
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
# Allow monitoring to scrape metrics
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

This setup supports multiple independent services with strict isolation.

---

## Example 3: Multi-Team Cluster

```yaml
---
# Team A namespace with default-deny
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
  labels:
    name: team-a

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Team A: Frontend to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: team-a-frontend-to-backend
  namespace: team-a
spec:
  podSelector:
    matchLabels:
      service: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          service: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# Team A: Backend to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: team-a-backend-to-db
  namespace: team-a
spec:
  podSelector:
    matchLabels:
      service: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          service: backend
    ports:
    - protocol: TCP
      port: 5432

---
# Team A: Allow DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: team-a-dns
  namespace: team-a
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
# Team B namespace (identical to Team A but separate)
apiVersion: v1
kind: Namespace
metadata:
  name: team-b
  labels:
    name: team-b

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: team-b
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Team B: Frontend to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: team-b-frontend-to-backend
  namespace: team-b
spec:
  podSelector:
    matchLabels:
      service: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          service: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# Team B: Backend to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: team-b-backend-to-db
  namespace: team-b
spec:
  podSelector:
    matchLabels:
      service: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          service: backend
    ports:
    - protocol: TCP
      port: 5432

---
# Team B: Allow DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: team-b-dns
  namespace: team-b
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
# Cluster-wide: Allow monitoring
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-team-a
  namespace: team-a
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

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-team-b
  namespace: team-b
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

Result: Complete isolation between teams. Team A and Team B cannot communicate.

---

## Example 4: Stateful Applications (Database)

```yaml
---
# Database with strict isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  ingress:
  # Backend pods
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
  
  # Database admin tool (in monitoring namespace)
  - from:
    - namespaceSelector:
        matchLabels:
          name: admin
    ports:
    - protocol: TCP
      port: 5432

---
# Database egress (minimal)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  egress:
  # DNS only
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
# Backup pod (reads from database)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backup-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backup
    ports:
    - protocol: TCP
      port: 5432
```

Database is completely locked down:
- Only backend can write
- Only admin tool can access
- Only backup can read
- No outgoing traffic except DNS

---

## Example 5: Cross-Namespace Communication

```yaml
---
# Production namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    name: production
    env: prod

---
# Default-deny
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Production pods can call external services in services namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: to-shared-services
  namespace: production
spec:
  podSelector: {}
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: shared-services
    ports:
    - protocol: TCP
      port: 443  # HTTPS

---
# Services namespace
apiVersion: v1
kind: Namespace
metadata:
  name: shared-services
  labels:
    name: shared-services

---
# Allow from production namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-production
  namespace: shared-services
spec:
  podSelector:
    matchLabels:
      type: api
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production
    ports:
    - protocol: TCP
      port: 443
```

Production pods can call shared services, but shared services cannot reach production.

---

## Implementation Checklist

Before deploying policies:

- [ ] CNI supports Network Policies (Calico, Weave, Cilium, etc.)
- [ ] Label all pods with meaningful labels (tier, service, env)
- [ ] Start with default-deny in one namespace
- [ ] Test each allow rule before deploying
- [ ] Document your policies (why each rule exists)
- [ ] Monitor for denied connections (logs)
- [ ] Have rollback plan (remove policies if issues)
- [ ] Review policies monthly (remove unused ones)

---

## Testing Your Policies

```bash
# Apply policies
kubectl apply -f network-policies.yaml -n production

# Test allowed traffic
kubectl exec frontend-pod -- curl backend:8080
# Expected: Success

# Test blocked traffic
kubectl exec frontend-pod -- curl database:5432
# Expected: Timeout

# Check policy status
kubectl get networkpolicies -n production

# View policy details
kubectl describe networkpolicy frontend-to-backend -n production

# Monitor denied connections (depends on CNI)
# For Calico:
kubectl logs -n calico-system deployment/calico-typha | grep "deny"
```

---

## Summary

Network Policies examples show:

1. **Simple three-tier**: Internet → Frontend → Backend → Database
2. **Microservices**: Gateway → Multiple independent services
3. **Multi-team**: Complete isolation between teams
4. **Databases**: Strict access controls for sensitive data
5. **Cross-namespace**: Communication with isolation

**Pattern for all:**
- Default-deny everything
- Explicitly allow each flow
- Label everything
- Test thoroughly
- Monitor violations

In this chapter, **Chapter 23 - Network Policies**, you have learned:
- File 1: What Network Policies are
- File 2: Ingress rules (incoming traffic)
- File 3: Egress rules (outgoing traffic)
- File 4: Default-deny pattern
- File 5 (this): Real-world examples

---

## Module 7 - Security Complete

You have completed **Module 7 - Kubernetes Security** with 4 chapters:

| Chapter | Topic | Files |
|---------|-------|-------|
| **20** | Security Basics | 3 |
| **21** | RBAC | 5 |
| **22** | Pod Security | 5 |
| **23** | Network Policies | 5 |

**Total: 18 files covering complete Kubernetes security**

Congratulations! You now understand production-grade Kubernetes security.
