# Pod Security Admission Controller

## Introduction

You know what Pod Security Standards are (enforce, audit, warn). But how does Kubernetes actually enforce them?

An **admission controller** is a system that runs before a pod is created. It can:
- Reject the pod ("no, this violates policy")
- Modify the pod ("I am automatically adding security controls")
- Allow the pod ("yes, this is fine")

The **Pod Security Admission** controller is what actually checks Pod Security Standards and rejects/warns about violations.

---

## How Pod Security Admission Works

When you create a pod:

```
Step 1: kubectl sends pod request to API server
        kubectl apply -f my-pod.yaml

Step 2: API server runs authentication
        "Who are you?"

Step 3: API server runs authorization
        "What are you allowed to do?"

Step 4: API server runs admission controllers
        This is where Pod Security Admission runs!
        "Does this pod violate Pod Security Standards?"

Step 4a: If violation found:
         - enforce mode: Reject the pod ✗
         - audit mode: Allow but log
         - warn mode: Allow but warn

Step 5: Pod is created (if allowed)
```

---

## Enabling Pod Security Admission

Pod Security Admission is built into Kubernetes (v1.25+) but must be enabled.

### Check if it's enabled

```bash
# See if Pod Security Admission admission controller is listed
kubectl get apiservice v1.admission.k8s.io -o yaml

# Or check API server configuration
kubectl describe pod kube-apiserver-node1 -n kube-system | grep admission
```

### Enable it

Usually enabled by default in modern clusters. If not, add to kube-apiserver flags:

```bash
kube-apiserver \
  --enable-admission-plugins=PodSecurity
```

---

## How Enforcement Works

### Step 1: Label the Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
```

The label tells the admission controller: "In this namespace, enforce the `restricted` standard."

### Step 2: Try to Create a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: insecure-app
  namespace: production
spec:
  containers:
  - name: app
    image: myapp:latest
    # Missing securityContext!
```

### Step 3: Admission Controller Checks

Pod Security Admission reads the namespace label and checks:

```
Is pod in namespace with pod-security.kubernetes.io/enforce=restricted?
    ↓ YES
Does pod meet `restricted` standard?
    ↓ NO (missing securityContext)
What mode is it?
    ↓ enforce (reject)
Result: Pod is REJECTED
```

Output:
```
Error: pod "insecure-app" is forbidden: 
violates PodSecurity "restricted:latest": 
  runAsNonRoot != true,
  allowPrivilegeEscalation != false,
  unrestricted capabilities,
  securityContext.runAsUser missing
```

---

## The Restricted Standard (In Detail)

What exactly does `restricted` enforce? Here is the full set of requirements:

### Requirement 1: Must Run as Non-Root

```yaml
# WRONG
spec:
  containers:
  - name: app
    image: myapp:latest
    # runAsUser not specified = runs as root

# CORRECT
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: app
    image: myapp:latest
```

### Requirement 2: Cannot Escalate to Root

```yaml
# WRONG
securityContext:
  allowPrivilegeEscalation: true  # Can escalate

# CORRECT
securityContext:
  allowPrivilegeEscalation: false  # Cannot escalate
```

### Requirement 3: Must Drop Dangerous Capabilities

```yaml
# WRONG
securityContext:
  capabilities:
    add:
    - SYS_ADMIN
    - NET_ADMIN

# CORRECT
securityContext:
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE  # Only if needed
```

### Requirement 4: No Privileged Containers

```yaml
# WRONG
securityContext:
  privileged: true  # Not allowed at Restricted level

# CORRECT
securityContext:
  privileged: false
```

### Requirement 5: No hostNetwork/hostPID/hostIPC

```yaml
# WRONG
spec:
  hostNetwork: true  # Not allowed
  hostPID: true      # Not allowed
  hostIPC: true      # Not allowed

# CORRECT
spec:
  hostNetwork: false
  hostPID: false
  hostIPC: false
```

### Requirement 6: No hostPath Volumes

```yaml
# WRONG
spec:
  volumes:
  - name: host-fs
    hostPath:
      path: /etc  # Accessing host filesystem

# CORRECT
spec:
  volumes:
  - name: data
    emptyDir: {}  # Use pod-local storage
```

### Requirement 7: SELinux/AppArmor (Optional)

```yaml
# GOOD (but optional)
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
```

---

## Common Violations and How to Fix Them

### Violation 1: Missing runAsNonRoot

**Error:**
```
runAsNonRoot != true
```

**Fix:**
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
```

### Violation 2: allowPrivilegeEscalation Not Set to False

**Error:**
```
allowPrivilegeEscalation != false
```

**Fix:**
```yaml
spec:
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
```

### Violation 3: Unrestricted Capabilities

**Error:**
```
unrestricted capabilities: ["NET_RAW", "ALL"]
```

**Fix:**
```yaml
securityContext:
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE  # Only what's needed
```

### Violation 4: Privileged Container

**Error:**
```
privileged != false
```

**Fix:**
```yaml
securityContext:
  privileged: false
```

### Violation 5: Using hostPath

**Error:**
```
hostPath volume is restricted at Restricted level
```

**Fix:**
```yaml
# Instead of hostPath
volumes:
- name: data
  hostPath:
    path: /mnt/data

# Use emptyDir or persistentVolume
volumes:
- name: data
  emptyDir: {}

# Or persistent volume
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
```

---

## Audit and Warn Modes

### Audit Mode

Logs violations to the audit log without rejecting pods.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    pod-security.kubernetes.io/audit: restricted
```

Deploy insecure pod:

```bash
$ kubectl apply -f insecure-pod.yaml
pod/insecure-app created  # Pod is created!
```

In audit logs (usually in `/var/log/audit.log`):

```json
{
  "level": "Metadata",
  "auditID": "e3701beb-...",
  "stage": "ResponseComplete",
  "requestObject": {...},
  "annotations": {
    "pod-security.kubernetes.io/audit": "restricted:latest violation: runAsNonRoot != true"
  }
}
```

**Use cases:**
- Getting started with Pod Security Standards
- Measuring impact before enforcement
- Finding problematic pods

### Warn Mode

Shows warnings in kubectl output without rejecting.

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
$ kubectl apply -f insecure-pod.yaml
Warning: would violate PodSecurity "restricted:latest": 
  runAsNonRoot != true,
  allowPrivilegeEscalation != false,
  unrestricted capabilities,
  securityContext.runAsUser missing
pod/insecure-app created
```

**Use cases:**
- Helping developers learn best practices
- Gradually enforcing standards
- Development environments

---

## Cluster Policies Using Pod Security Admission

### Example 1: Strict Production

```yaml
# Production namespace with strict enforcement
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
# Try to deploy insecure pod
apiVersion: v1
kind: Pod
metadata:
  name: insecure
  namespace: production
spec:
  containers:
  - name: app
    image: myapp:latest

# Result: REJECTED
```

### Example 2: Lenient Development

```yaml
# Development namespace with loose restrictions
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted

---
# Less secure pod is allowed
apiVersion: v1
kind: Pod
metadata:
  name: dev-app
  namespace: development
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      privileged: true  # Allowed at baseline
```

### Example 3: System Pods

```yaml
# Kubernetes system namespace (privileged pods)
apiVersion: v1
kind: Namespace
metadata:
  name: kube-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
```

---

## Debugging Admission Decisions

### See What Happened

```bash
# Check if pod was rejected
kubectl describe pod my-pod -n production

# Look for admission controller events
kubectl get events -n production

# Check kube-apiserver logs
kubectl logs -n kube-system -l component=kube-apiserver | grep PodSecurity
```

### Test Before Applying

```bash
# Dry-run to see if pod would be rejected
kubectl apply -f pod.yaml --dry-run=server -n production

# If there's an error, the pod violates standards
```

---

## Common Mistakes with Pod Security Admission

### Mistake 1: Forgetting the Namespace Label

```yaml
# WRONG — no Pod Security Standard label
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  # Missing pod-security.kubernetes.io/enforce label!

---
# Insecure pod is allowed (no standard to check!)
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: myapp
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      privileged: true  # Allowed!
```

**Fix:** Add the label:
```yaml
labels:
  pod-security.kubernetes.io/enforce: restricted
```

### Mistake 2: Audit Without Action

```yaml
# WRONG — only audit, never enforce
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/audit: restricted
    # No enforce!
```

Insecure pods are logged but still deployed. Eventually enforce them.

### Mistake 3: Exempting Too Much

```yaml
# WRONG — exempting everything
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
  # exempting all pods defeats the purpose!
```

Only exempt specific pods when absolutely necessary.

---

## Summary

Pod Security Admission is the controller that enforces Pod Security Standards:

**How it works:**
1. Pod is created
2. Admission controller checks namespace labels
3. If violation found: enforce=reject, audit=log, warn=warn
4. Pod is created (or rejected)

**Three modes:**
- **enforce** — reject non-compliant pods
- **audit** — log violations
- **warn** — show warnings

**Typical workflow:**
1. Set namespace labels
2. Pod admission controller enforces standards
3. Violations are rejected/logged/warned
4. Developers fix pods to meet standards

In the next file, `04-non-root-containers.md`, we will dive deep into running containers as non-root — one of the most important security practices.
