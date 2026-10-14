# What Are Network Policies

## Introduction

You have locked down individual pods with SecurityContext and RBAC. Now it is time to control how pods talk to each other.

By default, any pod can communicate with any other pod. This is terrible for security because:
- If one pod is compromised, the attacker can reach any other pod
- There is no containment or isolation
- Lateral movement is unrestricted

**Network Policies** are like firewalls for pods. They control which pods can send traffic to which other pods.

---

## The Problem: No Isolation by Default

Imagine your cluster has three services:

```
Frontend Pod ─┐
              ├─ Backend Pod ─ Database Pod
Developer App ┘
```

By default:
- Frontend can talk to Backend ✓ (good)
- Frontend can talk to Database ✓ (BAD!)
- Backend can talk to Database ✓ (good)
- Developer App can talk to Database ✓ (BAD!)
- Any pod can talk to any pod ✓ (BAD!)

If Frontend is compromised:

```
Attacker gains code in Frontend
     ↓
Attacker sees DATABASE_HOST in environment variables
     ↓
Attacker connects directly to Database (no Backend needed)
     ↓
Attacker bypasses security in Backend
     ↓
Data is stolen
```

The attacker can jump directly to the database.

---

## How Network Policies Work

A Network Policy is a set of rules that define:
- **Which pods** can send traffic
- **To which pods** can receive traffic
- **On which ports/protocols**

Think of it like a whitelist. By default, no traffic. Only explicitly allowed traffic passes through.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
spec:
  podSelector:              # ← Rule applies to these pods
    matchLabels:
      tier: database
  
  policyTypes:
  - Ingress                 # ← Control incoming traffic
  
  ingress:
  - from:                   # ← Traffic FROM these pods
    - podSelector:
        matchLabels:
          tier: backend
    ports:                  # ← On these ports
    - protocol: TCP
      port: 5432
```

Translation: "Pods labeled `tier: database` can receive traffic from pods labeled `tier: backend` on port 5432 (TCP)."

---

## Network Policy Components

### 1. podSelector

Selects which pods this policy applies to.

```yaml
podSelector:
  matchLabels:
    tier: database
```

"This policy applies to pods with label `tier: database`"

Empty selector means all pods:

```yaml
podSelector: {}  # Applies to all pods in namespace
```

### 2. policyTypes

What types of traffic to control.

```yaml
policyTypes:
- Ingress   # Control incoming traffic
- Egress    # Control outgoing traffic
- Both      # (or list both)
```

### 3. ingress

Rules for **incoming** traffic (who can talk to these pods).

```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        tier: frontend
  - namespaceSelector:
      matchLabels:
        name: production
  ports:
  - protocol: TCP
    port: 8080
```

"Allow traffic from pods labeled `tier: frontend` in namespace labeled `name: production` on port 8080"

### 4. egress

Rules for **outgoing** traffic (these pods can talk to who).

```yaml
egress:
- to:
  - podSelector:
      matchLabels:
        tier: backend
  ports:
  - protocol: TCP
    port: 5432
```

"Allow these pods to send traffic to pods labeled `tier: backend` on port 5432"

---

## Simple Example

### Before Network Policies

```yaml
# Frontend
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  labels:
    tier: frontend
spec:
  containers:
  - name: app
    image: frontend:latest

---
# Database
apiVersion: v1
kind: Pod
metadata:
  name: database
  labels:
    tier: database
spec:
  containers:
  - name: db
    image: postgres:latest
```

Problem: Frontend can connect directly to Database.

```bash
# From frontend pod:
$ psql -h database -U postgres
# Success! Direct access to database (should be through backend)
```

### After Network Policies

```yaml
# Policy 1: Allow traffic to database only from backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-only
spec:
  podSelector:
    matchLabels:
      tier: database
  
  policyTypes:
  - Ingress
  
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
```

Now:

```bash
# From frontend pod (not in ingress rule):
$ psql -h database -U postgres
# Timeout (connection blocked by network policy)

# From backend pod (in ingress rule):
$ psql -h database -U postgres
# Success!
```

---

## Real-World Three-Tier Architecture

```
Internet
   ↓
[Ingress] (external traffic enters)
   ↓
[Frontend Pods] (web servers)
   ↓
[Backend Pods] (API servers)
   ↓
[Database Pod] (PostgreSQL)
```

Network Policies:

```yaml
---
# 1. Allow internet to frontend (via Ingress controller)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-frontend
spec:
  podSelector:
    matchLabels:
      tier: frontend
  
  policyTypes:
  - Ingress
  
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443

---
# 2. Allow frontend to backend (only what's needed)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
spec:
  podSelector:
    matchLabels:
      tier: backend
  
  policyTypes:
  - Ingress
  
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# 3. Allow backend to database (only what's needed)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-to-database
spec:
  podSelector:
    matchLabels:
      tier: database
  
  policyTypes:
  - Ingress
  
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432

---
# 4. Allow all pods to reach DNS (needed for service discovery)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}  # All pods
  
  policyTypes:
  - Egress
  
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

Result:

```
✓ Internet → Frontend (allowed by policy 1)
✓ Frontend → Backend (allowed by policy 2)
✓ Backend → Database (allowed by policy 3)

✗ Frontend → Database (NOT allowed, blocks direct access)
✗ Any pod → External (NOT allowed, except DNS)
✗ Frontend → Backend on wrong port (NOT allowed)
```

---

## Namespace vs Cluster Scope

Network Policies are **namespace-scoped**. A policy in namespace `production` only affects pods in namespace `production`.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-policy
  namespace: production  # Only applies in production namespace
spec:
  # ...
```

To isolate between namespaces, use namespace selectors:

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: production  # Allow from production namespace
```

---

## Prerequisites: CNI Plugin

Network Policies require a CNI (Container Network Interface) plugin that supports them:

**Supported:**
- Calico ✓
- Cilium ✓
- Weave ✓
- Kube-router ✓

**NOT supported (ignore Network Policies):**
- Flannel ✗

Check if your cluster supports Network Policies:

```bash
# See which CNI is installed
kubectl get daemonset -n kube-system

# Look for calico-node, cilium, weave, etc.
```

If your cluster doesn't support Network Policies, you need to install a CNI that does.

---

## Key Concepts

### Allow-List (Default Deny)

Network Policies work on an allow-list basis: **by default, block everything. Then explicitly allow what you need.**

This is opposite to firewalls on traditional servers (which usually default to allow).

```yaml
# First: Block EVERYTHING
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Then: Explicitly allow specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
spec:
  podSelector:
    matchLabels:
      tier: database
  
  policyTypes:
  - Ingress
  
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
```

### Matching Rules

Selectors use label matching:

```yaml
# Match pods with specific label
- podSelector:
    matchLabels:
      tier: backend      # tier=backend

# Match pods with multiple labels
- podSelector:
    matchLabels:
      app: payment       # app=payment AND
      tier: backend      # tier=backend

# Match from different namespace
- namespaceSelector:
    matchLabels:
      name: production

# Match anything (all pods)
- podSelector: {}
```

---

## Summary

Network Policies control communication between pods:

**Key points:**
- Default: no isolation (any pod can talk to any pod)
- Network Policies: explicit allow-list
- Default deny all, then explicitly allow what's needed
- Scope: per-namespace
- Requires CNI plugin support

**Typical structure:**
1. Default deny all ingress/egress
2. Allow specific ingress (which pods can receive traffic)
3. Allow specific egress (which pods can send traffic)
4. Allow DNS (all pods need to resolve names)

**Benefits:**
- Prevents lateral movement (compromised pod can't reach others)
- Enforces least-privilege communication
- Limits blast radius of breaches
- Improves security posture significantly

In the next file, `02-ingress-rules.md`, we will dive deep into ingress rules — controlling which pods can **receive** traffic.
