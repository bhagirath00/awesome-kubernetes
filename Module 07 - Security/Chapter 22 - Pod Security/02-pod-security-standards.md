# Pod Security Standards

## Introduction

You now know how to write a secure SecurityContext for one pod. But what about your entire cluster?

Do you really want to manually add SecurityContext to every pod? What if someone forgets and deploys an insecure pod?

**Pod Security Standards** is the answer. It is a policy that enforces SecurityContext rules cluster-wide (or namespace-wide). Think of it like a linter for pod security.

Pod Security Standards automatically:
- Reject unsafe pods
- Warn about unsafe pods
- Optionally auto-fix unsafe pods

---

## The Three Levels

Pod Security Standards defines three levels of security:

### Level 1: Privileged

No restrictions. Pods can do anything.

```yaml
# This pod is allowed at Privileged level
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      privileged: true  # Full access
```

**Use case:** Kubernetes system pods, storage drivers, network plugins.

### Level 2: Baseline

Blocks the most dangerous features.

**What's blocked:**
- Privileged containers
- hostNetwork, hostPID, hostIPC
- hostPath volumes (accessing host filesystem)

**What's allowed:**
- Running as root
- No SecurityContext

```yaml
# This pod is blocked at Baseline
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      privileged: true  # ✗ Blocked
```

### Level 3: Restricted (Recommended)

Most strict. Enforces best practices.

**What's required:**
- Must run as non-root user
- Linux capabilities — only NET_BIND_SERVICE
- readOnlyRootFilesystem: true
- allowPrivilegeEscalation: false

**What's blocked:**
- Privileged containers
- hostNetwork, hostPID, hostIPC, hostPath
- Root user
- Dangerous capabilities
- Privilege escalation

```yaml
# Restricted level requires this
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

---

## Enforcing Pod Security Standards

You set the level per namespace using labels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # enforce — block pods that don't meet standard
    pod-security.kubernetes.io/enforce: restricted
    
    # audit — log warnings but allow
    pod-security.kubernetes.io/audit: restricted
    
    # warn — show warnings in kubectl output
    pod-security.kubernetes.io/warn: restricted
```

### Mode 1: enforce

Rejects pods that don't meet the standard.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

Try to deploy an insecure pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: production
spec:
  containers:
  - name: app
    image: myapp:latest
    # Missing securityContext!
```

Result:
```
Error: pod "app" is forbidden: violates pod-security.kubernetes.io/enforce=restricted
```

Pod is rejected. Developers must fix the SecurityContext.

### Mode 2: audit

Logs violations but allows pods.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    pod-security.kubernetes.io/audit: restricted
```

Deploy insecure pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: staging
spec:
  containers:
  - name: app
    image: myapp:latest
```

Result:
- Pod is created (allowed)
- Violation logged in audit logs
- You can review audit logs and find the problem

### Mode 3: warn

Shows warnings in kubectl output.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    pod-security.kubernetes.io/warn: restricted
```

Deploy insecure pod:

```bash
$ kubectl apply -f pod.yaml
Warning: would violate PodSecurity "restricted:latest": 
  allowPrivilegeEscalation != false,
  unrestricted capabilities,
  runAsNonRoot != true,
  securityContext.runAsUser missing
pod/app created
```

Pod is created but shows warnings. Developers know to fix it.

---

## Common Pod Security Standard Policies

### Policy 1: Production (Strict)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

- Enforce restricted (strict)
- Audit violations
- Warn on violations

### Policy 2: Development (Permissive)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

- Enforce baseline (allow most things)
- But audit and warn about restricted violations (help developers learn)

### Policy 3: System (Privileged)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kube-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
```

- Allow privileged containers (system pods need it)

---

## Setting Up Pod Security Standards Cluster-Wide

Create a ClusterRole that enforces standards globally:

```yaml
# 1. Create a namespace with restricted label
apiVersion: v1
kind: Namespace
metadata:
  name: app
  labels:
    pod-security.kubernetes.io/enforce: restricted

---
# 2. Now all pods in this namespace must be secure
# This pod will be rejected
apiVersion: v1
kind: Pod
metadata:
  name: insecure-app
  namespace: app
spec:
  containers:
  - name: app
    image: myapp:latest
    # Missing securityContext!
```

---

## Exceptions to Pod Security Standards

Sometimes you need to allow exceptions (e.g., a third-party pod that doesn't follow standards).

Use annotations to exempt:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: third-party
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
spec: {}

---
# This pod is exempt from restrictions
apiVersion: v1
kind: Pod
metadata:
  name: vendor-app
  namespace: third-party
  annotations:
    # Manually exempt from pod security standards
    # (use with caution!)
spec:
  containers:
  - name: app
    image: vendor-app:latest
    securityContext:
      privileged: true  # Allowed via exemption
```

---

## Migrating to Pod Security Standards

**Step 1: Start with audit mode**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/audit: restricted
```

- All pods are allowed
- Violations are logged
- You can see which pods don't meet the standard

**Step 2: Fix the worst violations**

Review audit logs and update the most insecure pods.

**Step 3: Enable warn mode**

```yaml
pod-security.kubernetes.io/warn: restricted
```

Developers see warnings for violations.

**Step 4: Enforce**

```yaml
pod-security.kubernetes.io/enforce: restricted
```

New pods must meet the standard. Existing pods are allowed (enforcement only applies to new pods).

**Step 5: Clean up old pods**

Delete or update existing insecure pods.

---

## Real-World Example

```yaml
# Production namespace with strict security
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
# Production deployment must be secure
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment
  
  template:
    metadata:
      labels:
        app: payment
    
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      
      containers:
      - name: payment
        image: payment-service:latest
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/payment
      
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}

---
# Development namespace with looser security
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted

---
# Development pod can be less secure
apiVersion: v1
kind: Pod
metadata:
  name: dev-app
  namespace: development
spec:
  containers:
  - name: app
    image: dev-app:latest
    # Less secure, but still allowed in development
```

---

## Checking Pod Security Standard Violations

```bash
# See warnings about violations
kubectl apply -f insecure-pod.yaml

# Check pod security in a namespace
kubectl label namespace my-namespace pod-security.kubernetes.io/audit=restricted

# Review audit logs for violations
kubectl logs -n kube-apiserver

# Check which pods violate standards
kubectl get pods -n production -o wide
```

---

## Summary

Pod Security Standards enforce SecurityContext rules:

**Three levels:**
- **Privileged** — no restrictions
- **Baseline** — blocks most dangerous features
- **Restricted** — enforces best practices (recommended)

**Three modes:**
- **enforce** — reject non-compliant pods
- **audit** — log violations
- **warn** — show warnings

**Typical setup:**
```
Production: enforce=restricted, audit=restricted, warn=restricted
Development: enforce=baseline, warn=restricted
```

Use Pod Security Standards to ensure all pods meet your security requirements automatically.

In the next file, `03-pod-security-admission.md`, we will learn about the admission controller that actually enforces these standards.
