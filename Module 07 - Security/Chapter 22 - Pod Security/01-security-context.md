# Security Context

## Introduction

A SecurityContext is where you tell Kubernetes "how should this container run?" It controls low-level security settings like:
- Which user should the container run as?
- Can the container escalate to root?
- Can it access the host filesystem?
- Which Linux capabilities does it have?

Think of SecurityContext like a container jail. The more restrictions you set, the smaller the jail is. If the container escapes the jail, it is already limited.

---

## What is a SecurityContext?

A SecurityContext defines OS-level security properties for pods and containers. Examples:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  # Pod-level security context (applies to all containers)
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  
  containers:
  - name: app
    image: myapp:latest
    # Container-level security context (overrides pod level)
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

---

## Core SecurityContext Fields

### runAsUser / runAsGroup

**What:** Which user/group should the container run as?

```yaml
securityContext:
  runAsUser: 1000       # Run as UID 1000 (not root)
  runAsGroup: 3000      # Run as GID 3000
```

**Why:** If the container is compromised, the attacker is UID 1000, not root. They cannot:
- Modify core system files
- Install packages
- Access root-owned files
- Escalate privileges

**Example:**

```yaml
# WRONG — runs as root
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:latest
    # No securityContext = runs as root

---
# CORRECT — runs as non-root
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  securityContext:
    runAsNonRoot: true  # Force non-root
    runAsUser: 1000
  containers:
  - name: app
    image: myapp:latest
```

If the second pod tries to run as root, Kubernetes rejects it.

### allowPrivilegeEscalation

**What:** Can the container become root?

```yaml
securityContext:
  allowPrivilegeEscalation: false  # Cannot escalate
```

Even if running as non-root, the container might escalate to root via:
- SETUID binaries
- Kernel exploits
- Misconfigured capabilities

Setting `allowPrivilegeEscalation: false` prevents this.

**Example:**

```yaml
# WRONG — can escalate
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:latest
    # allowPrivilegeEscalation not set = can escalate

---
# CORRECT — cannot escalate
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
```

### readOnlyRootFilesystem

**What:** Should the container's root filesystem be read-only?

```yaml
securityContext:
  readOnlyRootFilesystem: true  # Root filesystem is read-only
```

**Why:** If the container is compromised, the attacker cannot:
- Modify application code
- Install backdoors
- Modify configuration files
- Write logs to disk

**Example:**

```yaml
# WRONG — root filesystem is writable
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:latest

---
# CORRECT — root filesystem is read-only
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: tmp
      mountPath: /tmp  # Writable volume for temp files
  volumes:
  - name: tmp
    emptyDir: {}
```

The application can still write to `/tmp` (a temporary volume), but cannot modify the application code or system files.

### runAsNonRoot

**What:** Force the container to run as non-root?

```yaml
securityContext:
  runAsNonRoot: true  # Container MUST run as non-root
```

If the image is configured to run as root, Kubernetes rejects the pod.

```bash
# WRONG — image runs as root by default
$ docker run ubuntu:latest id
uid=0(root) gid=0(root) groups=0(root)

# In Kubernetes
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  securityContext:
    runAsNonRoot: true  # But we require non-root!
  containers:
  - name: app
    image: ubuntu:latest
    # This pod will be rejected!
```

### capabilities

**What:** Which Linux capabilities does the container have?

Linux capabilities are low-level permissions:
- `NET_ADMIN` — modify network settings
- `SYS_ADMIN` — various admin operations
- `DAC_OVERRIDE` — bypass file permissions
- `CHOWN` — change file ownership
- etc.

By default, containers have several capabilities. Drop all and add back only what's needed:

```yaml
securityContext:
  capabilities:
    drop:
    - ALL              # Drop all capabilities
    add:
    - NET_BIND_SERVICE # Add back only what's needed
```

**Example — web server:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    securityContext:
      runAsNonRoot: true
      runAsUser: 101
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE  # Can bind to port 80
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

Now nginx:
- Runs as user 101 (not root)
- Cannot escalate to root
- Root filesystem is read-only
- Only has `NET_BIND_SERVICE` capability (can listen on port 80)
- Very limited blast radius if compromised

---

## Pod-Level vs Container-Level

SecurityContext can be set at two levels:

### Pod-Level (applies to all containers)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  securityContext:
    runAsUser: 1000        # All containers run as UID 1000
    fsGroup: 2000
  
  containers:
  - name: app1
    image: myapp:latest
  
  - name: app2
    image: otherapp:latest
```

Both containers inherit the pod-level SecurityContext.

### Container-Level (overrides pod-level)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  securityContext:
    runAsUser: 1000
  
  containers:
  - name: app1
    image: myapp:latest
    securityContext:
      runAsUser: 2000  # Overrides pod-level (runs as UID 2000)
  
  - name: app2
    image: otherapp:latest
    # Uses pod-level (runs as UID 1000)
```

Container-level settings override pod-level settings.

---

## Common SecurityContext Patterns

### Pattern 1: Restricted Container (Most Secure)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
  
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: data
      mountPath: /data
  
  volumes:
  - name: tmp
    emptyDir: {}
  - name: data
    emptyDir: {}
```

This is the "restricted" Pod Security Standard — maximum protection.

### Pattern 2: Privileged Container (Least Secure)

```yaml
# WRONG — only use when absolutely necessary
apiVersion: v1
kind: Pod
metadata:
  name: privileged-app
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      privileged: true  # Full access
```

A privileged container:
- Runs as root
- Can access host filesystem
- Can manipulate kernel
- Can potentially escape to host

Only use for:
- Kubernetes system pods
- Network plugins
- Storage drivers
- Never for user applications

### Pattern 3: Storage Pod (needs to write)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000  # Files owned by GID 2000
  
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      # readOnlyRootFilesystem: true  (can't set for storage)
      capabilities:
        drop:
        - ALL
    
    volumeMounts:
    - name: data
      mountPath: /data  # Can write here
  
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

The pod can write to `/data` but not to the root filesystem.

---

## Debugging SecurityContext Issues

### Issue 1: Container Won't Start

```bash
# Pod logs
kubectl logs my-pod

# Often you see:
# Error: OCI runtime error: open /etc/ssl/certs: permission denied
```

This usually means the container tries to access a file it doesn't have permission for. The runAsUser doesn't own the file.

**Solution:**
```yaml
securityContext:
  runAsUser: 1000
  fsGroup: 2000  # Make files group-readable
```

### Issue 2: Disk Write Fails

```bash
# Container tries to write to disk
# Error: Read-only file system
```

This happens with `readOnlyRootFilesystem: true`. Mount a writable volume:

```yaml
securityContext:
  readOnlyRootFilesystem: true

volumeMounts:
- name: tmp
  mountPath: /tmp  # Writable
- name: data
  mountPath: /data  # Writable
```

### Issue 3: Permission Denied on Shared Volume

```bash
# Multiple containers writing to same volume
# Error: Permission denied
```

The containers run as different UIDs. Use `fsGroup`:

```yaml
securityContext:
  fsGroup: 2000  # All containers in group 2000
```

Now all containers can write to volume (via group permissions).

---

## Testing SecurityContext

```bash
# Check what user the container runs as
kubectl exec -it my-pod -- id
# Output: uid=1000 gid=2000 groups=2000

# Check filesystem permissions
kubectl exec -it my-pod -- ls -la /
# Output: -r-xr-xr-x (read-only)

# Try to write to root filesystem
kubectl exec -it my-pod -- touch /test.txt
# Error: Read-only file system (success! can't write)

# Check capabilities
kubectl exec -it my-pod -- cat /proc/self/status | grep Cap
# Output: low capabilities (success! most dropped)
```

---

## Summary

SecurityContext defines how a container runs:

**Key fields:**
- `runAsUser` / `runAsGroup` — which user to run as
- `runAsNonRoot` — force non-root
- `allowPrivilegeEscalation` — can't become root
- `readOnlyRootFilesystem` — root filesystem read-only
- `capabilities` — drop dangerous Linux capabilities

**Pattern:**
```
runAsNonRoot: true
runAsUser: 1000
allowPrivilegeEscalation: false
readOnlyRootFilesystem: true
capabilities: drop ALL, add only what's needed
```

If a container is compromised, the attacker is very limited — not root, can't escalate, can't modify files, very few capabilities.

In the next file, `02-pod-security-standards.md`, we will learn about Pod Security Standards — a policy layer that enforces SecurityContext rules across your cluster.
