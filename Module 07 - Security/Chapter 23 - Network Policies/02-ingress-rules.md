# Ingress Rules

## Introduction

Ingress rules control incoming traffic to a pod. They answer: "Which pods/namespaces can reach this pod?"

Let me show you how to write them.

---

## Basic Ingress Rule

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend
spec:
  # This policy applies to database pods
  podSelector:
    matchLabels:
      tier: database
  
  policyTypes:
  - Ingress
  
  # Ingress rules (incoming traffic)
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend     # Only from backend pods
    ports:
    - protocol: TCP
      port: 5432
```

Translation: "Database pods accept connections from backend pods on port 5432."

---

## Rule Structure

Each ingress rule has three parts:

```yaml
ingress:
- from: [ ... ]      # From these sources
  ports: [ ... ]     # On these ports
  description: "optional description"
```

### Part 1: `from` (Where does traffic come from?)

Traffic source can be:

**Pod selector (same namespace):**
```yaml
from:
- podSelector:
    matchLabels:
      tier: frontend
```

Only pods with `tier: frontend` in the same namespace.

**Namespace selector:**
```yaml
from:
- namespaceSelector:
    matchLabels:
      name: monitoring
```

Any pod in the `monitoring` namespace.

**Both (Pod in specific namespace):**
```yaml
from:
- podSelector:
    matchLabels:
      tier: frontend
  namespaceSelector:
    matchLabels:
      env: production
```

Pods with `tier: frontend` in namespaces with `env: production`.

**Multiple sources (OR logic):**
```yaml
from:
# Source 1: Frontend pods in this namespace
- podSelector:
    matchLabels:
      tier: frontend

# Source 2: Any pod in monitoring namespace
- namespaceSelector:
    matchLabels:
      name: monitoring

# Both sources can reach
```

### Part 2: `ports` (Which ports?)

Which ports can receive traffic?

```yaml
ports:
- protocol: TCP
  port: 5432      # PostgreSQL
- protocol: UDP
  port: 53        # DNS
```

**All ports:**
```yaml
# If ports field is omitted, all ports are allowed
ingress:
- from:
  - podSelector: {}
```

**Specific port:**
```yaml
ports:
- protocol: TCP
  port: 8080
```

**Port range:**
```yaml
ports:
- protocol: TCP
  port: 5000
  endPort: 5099   # Allows 5000-5099
```

---

## Common Ingress Patterns

### Pattern 1: Accept from Specific Pod

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-to-db
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
```

Only backend pods can reach database on port 5432.

### Pattern 2: Accept from Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: accept-from-monitoring
spec:
  podSelector:
    matchLabels:
      app: web
  
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090   # Prometheus metrics
```

Any pod in the `monitoring` namespace can reach metrics on port 9090.

### Pattern 3: Accept from Internet (via Ingress)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-from-ingress
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
      port: 8080   # From ingress controller
```

Only the ingress controller can reach frontend.

### Pattern 4: Accept from Multiple Sources

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-ingress
spec:
  podSelector:
    matchLabels:
      tier: api
  
  ingress:
  # From frontend pods
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  
  # From monitoring namespace
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090
  
  # From load tester
  - from:
    - podSelector:
        matchLabels:
          name: load-tester
    ports:
    - protocol: TCP
      port: 8080
```

Three different sources with different ports/purposes.

### Pattern 5: Allow Specific Labels

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cache-access
spec:
  podSelector:
    matchLabels:
      tier: cache
  
  ingress:
  # Only pods with label "cache-consumer: true" can reach
  - from:
    - podSelector:
        matchLabels:
          cache-consumer: "true"
    ports:
    - protocol: TCP
      port: 6379
```

Only pods explicitly labeled as cache consumers can access.

---

## Complex Selectors

### Match Multiple Labels (AND logic)

```yaml
podSelector:
  matchLabels:
    tier: backend
    env: production
```

Match pods with BOTH `tier=backend` AND `env=production`.

### Match Any of Values (OR logic)

```yaml
podSelector:
  matchExpressions:
  - key: tier
    operator: In
    values: [backend, cache]
```

Match pods with `tier=backend` OR `tier=cache`.

### Negation (NOT logic)

```yaml
podSelector:
  matchExpressions:
  - key: tier
    operator: NotIn
    values: [system]
```

Match pods where `tier` is NOT `system`.

---

## Real-World Example: Web Application

```yaml
---
# Frontend accepts from internet (ingress)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-ingress
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

---
# Backend accepts from frontend only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress
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
  
  # Also accept metrics requests
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090

---
# Database accepts from backend only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-ingress
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
```

Result:
```
Internet → Ingress Controller → Frontend (port 8080)
Frontend → Backend (port 8080)
Backend → Database (port 5432)
Monitoring → Backend (port 9090)
```

---

## Important Details

### Empty `from` List

```yaml
ingress:
- from: []     # Empty list = no sources allowed
  ports:
  - protocol: TCP
    port: 8080
```

No traffic is allowed (contradictory rule).

### Missing `from` vs Empty `from`

```yaml
# Version 1: No 'from' field
ingress:
- ports:
  - protocol: TCP
    port: 8080
# Means: from ANY source on port 8080

# Version 2: Empty 'from' list
ingress:
- from: []
  ports:
  - protocol: TCP
    port: 8080
# Means: from NO sources (deny all)
```

### Both podSelector and namespaceSelector

```yaml
from:
- podSelector:
    matchLabels:
      tier: frontend
  namespaceSelector:
    matchLabels:
      env: production
```

Means: "Pods with `tier=frontend` in namespaces with `env=production`"

If both are specified, it is AND logic (must match both).

---

## Testing Ingress Rules

```bash
# Create a test pod in different namespace
kubectl run test-pod --image=alpine --namespace=other -i --tty -- sh

# Try to reach protected pod
/ # wget -O- http://protected-pod:8080
# Should be blocked (connection timeout)

# Try from allowed namespace
kubectl run test-pod --image=alpine --namespace=allowed -i --tty -- sh

/ # wget -O- http://protected-pod:8080
# Should succeed
```

---

## Common Mistakes

### Mistake 1: Forgetting the Port

```yaml
# WRONG — port missing
ingress:
- from:
  - podSelector:
      matchLabels:
        tier: frontend
  # ports field missing! All ports allowed!

# CORRECT
ingress:
- from:
  - podSelector:
      matchLabels:
        tier: frontend
  ports:
  - protocol: TCP
    port: 8080  # Explicit port
```

### Mistake 2: Wrong Namespace Name

```yaml
# WRONG — namespace doesn't exist
from:
- namespaceSelector:
    matchLabels:
      name: production  # Namespace not labeled!

# CORRECT — label the namespace first
kubectl label namespace production name=production

# Then use the label
from:
- namespaceSelector:
    matchLabels:
      name: production
```

### Mistake 3: Mixing Pod and Namespace Selectors

```yaml
# WRONG — ambiguous
from:
- podSelector:
    matchLabels:
      tier: backend
- namespaceSelector:
    matchLabels:
      name: monitoring
# Is it "backend pods OR any pod in monitoring namespace"?
# Yes, it's OR (both rules apply separately)

# CLEAR
from:
- podSelector:
    matchLabels:
      tier: backend
- namespaceSelector:
    matchLabels:
      name: monitoring
# Explicitly: either backend pods, OR monitoring namespace
```

---

## Summary

Ingress rules define incoming traffic:

**Structure:**
```yaml
ingress:
- from: [ source1, source2, ... ]
  ports: [ port1, port2, ... ]
```

**Sources:**
- `podSelector` — specific pods in same namespace
- `namespaceSelector` — any pod in specific namespace
- Both together — specific pods in specific namespace

**Multiple rules = OR logic:**
```yaml
ingress:
- from: [ rule1 ]
- from: [ rule2 ]
# If either rule matches, traffic allowed
```

**Default if no rules:**
- No ingress field → all traffic allowed
- Empty ingress field → no traffic allowed
- Specific ingress rules → only those allowed

In the next file, `03-egress-rules.md`, we will learn about egress policies — controlling outgoing traffic.
