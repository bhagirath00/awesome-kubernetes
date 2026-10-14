# Non-Root Containers

## Introduction

Running containers as root is the security equivalent of leaving your front door open.

If a container runs as root and gets compromised, the attacker is root. They can:
- Modify any file on the container
- Install backdoors and malware
- Access sensitive data
- Potentially escape to the host node
- Compromise the entire cluster

Running as non-root is the simplest, highest-impact security improvement you can make.

---

## Why Non-Root Matters

Let me show you the difference with a real attack scenario.

### Scenario 1: Container Runs as Root

```
Step 1: Application has a vulnerability
        A malicious request exploits it

Step 2: Attacker gains code execution inside container
        $ whoami
        root  ← Oh no!

Step 3: Attacker installs backdoor
        $ apt-get install openssh-server
        $ chmod 777 /etc/ssh/sshd_config
        $ service ssh start

Step 4: Attacker maintains persistent access
        Comes back anytime to access the system

Step 5: Attacker explores
        $ cat /etc/passwd
        $ ls -la /root
        $ find / -name "*.key"

Step 6: Attacker escalates
        $ grep -r "password" /app/config
        Finds credentials, accesses databases
```

Damage: TOTAL. Attacker is root, can do anything.

### Scenario 2: Container Runs as Non-Root (UID 1000)

```
Step 1: Application has same vulnerability
        A malicious request exploits it

Step 2: Attacker gains code execution inside container
        $ whoami
        appuser  ← Much better!

Step 3: Attacker tries to install backdoor
        $ apt-get install openssh-server
        E: Could not open lock file /var/lib/apt/lists/lock - open (13: Permission denied)
        Failed! Can't install packages

Step 4: Attacker tries to write to /etc/ssh
        $ chmod 777 /etc/ssh/sshd_config
        chmod: changing permissions of '/etc/ssh/sshd_config': Operation not permitted
        Failed! Can't modify system files

Step 5: Attacker tries to modify application code
        $ echo "malicious" > /app/main.py
        /app/main.py: Permission denied
        Failed! Can't modify code

Step 6: Attacker is stuck
        Very limited options as non-root user
```

Damage: CONTAINED. Attacker is non-root, can only read/write to their own files.

The difference is HUGE.

---

## How to Run Containers as Non-Root

### Method 1: Bake it into the Image

The best practice is to build the image as non-root from the start.

**Dockerfile:**

```dockerfile
# WRONG — defaults to root
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y python3

COPY app.py /app/
CMD ["python3", "/app/app.py"]

# When run: uid=0 (root)
```

**CORRECT — creates non-root user:**

```dockerfile
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y python3

# Create non-root user
RUN useradd -m -u 1000 appuser

# Copy files with correct ownership
COPY --chown=appuser:appuser app.py /app/

# Switch to non-root user
USER appuser

CMD ["python3", "/app/app.py"]

# When run: uid=1000
```

Now the container always runs as `appuser` (UID 1000).

### Method 2: Override at Pod Level

If the image defaults to root, override it in Kubernetes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000        # Run as UID 1000
    runAsGroup: 3000       # Optional: run as GID 3000
    fsGroup: 2000          # Optional: file group
  
  containers:
  - name: app
    image: myapp:latest
```

Even if the image is built as root, Kubernetes forces it to run as UID 1000.

### Method 3: Validate at Enforcement Level

Use Pod Security Standards to ensure all pods are non-root:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    # Restricted level requires non-root
```

---

## Common Non-Root Issues and Solutions

### Issue 1: Permission Denied on Files

```
Container starts but crashes:
Error: permission denied: /app/data
```

This happens when the application tries to write to a directory it doesn't own.

**Problem:**

```dockerfile
# Image is built as root
FROM ubuntu:22.04
COPY app /app  # Owned by root
# No USER directive, runs as root
```

Then in Kubernetes:

```yaml
securityContext:
  runAsUser: 1000  # Run as appuser, but /app is owned by root
```

**Solution 1: Fix in image**

```dockerfile
FROM ubuntu:22.04

RUN useradd -m -u 1000 appuser
COPY --chown=appuser:appuser app /app
USER appuser
```

**Solution 2: Fix in Kubernetes**

```yaml
securityContext:
  runAsUser: 1000
  fsGroup: 1000        # Make files readable by group 1000
```

Now the pod can read files (via group permissions).

### Issue 2: Cannot Bind to Port < 1024

Ports 1-1023 require root. Non-root can't use them.

**WRONG:**
```dockerfile
USER appuser  # Non-root
EXPOSE 80     # Port 80 requires root!
```

Container starts but can't listen on port 80.

**Solution 1: Use Port >= 1024**
```dockerfile
USER appuser
EXPOSE 8080   # Non-root can use this
```

**Solution 2: Use NET_BIND_SERVICE Capability**
```yaml
securityContext:
  runAsUser: 1000
  capabilities:
    add:
    - NET_BIND_SERVICE  # Can bind to ports < 1024
```

**Solution 3: Port Forwarding**
```yaml
# Run on port 8080 inside
# Kubernetes service forwards 80→8080
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  ports:
  - name: http
    port: 80           # External port
    targetPort: 8080   # Internal port (non-root)
  selector:
    app: myapp
```

### Issue 3: Cannot Write to Root Filesystem

With `readOnlyRootFilesystem: true`, non-root can't write anywhere.

**Solution: Mount writable volumes**

```yaml
securityContext:
  runAsUser: 1000
  readOnlyRootFilesystem: true

volumeMounts:
- name: tmp
  mountPath: /tmp        # Writable temp
- name: cache
  mountPath: /var/cache  # Writable cache
- name: logs
  mountPath: /var/log    # Writable logs
- name: data
  mountPath: /data       # Writable data

volumes:
- name: tmp
  emptyDir: {}
- name: cache
  emptyDir: {}
- name: logs
  emptyDir: {}
- name: data
  persistentVolumeClaim:
    claimName: my-data
```

### Issue 4: Application Requires Root

Some applications legitimately need root (databases, reverse proxies, etc.).

**Options:**

1. **Find non-root alternative**
   ```
   Old: Apache (requires root for port 80)
   New: Nginx (can run non-root)
   ```

2. **Use minimal capabilities instead of full root**
   ```yaml
   securityContext:
     runAsUser: 1000
     allowPrivilegeEscalation: false
     capabilities:
       drop:
       - ALL
       add:
       - NET_BIND_SERVICE  # Only what's needed
   ```

3. **Init containers to prepare**
   ```yaml
   initContainers:
   - name: setup
     image: ubuntu:latest
     securityContext:
       runAsUser: 0  # Setup runs as root
     command: ["chmod", "644", "/data/config"]  # Prepare files
   
   containers:
   - name: app
     securityContext:
       runAsUser: 1000  # App runs as non-root
   ```

---

## Real-World Example: Web Application

```dockerfile
# Multi-stage build
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

---

FROM node:18-alpine

# Create non-root user
RUN addgroup -g 1000 app && \
    adduser -D -u 1000 -G app app

WORKDIR /app

# Copy built app
COPY --from=builder --chown=app:app /app/node_modules ./node_modules
COPY --chown=app:app . .

# Switch to non-root user
USER app

# Use non-privileged port
EXPOSE 8080

CMD ["node", "server.js"]
```

**Kubernetes Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  
  template:
    metadata:
      labels:
        app: webapp
    
    spec:
      # Pod-level security
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
      
      containers:
      - name: web
        image: mywebapp:latest
        
        # Container-level security
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
        
        # Resource limits
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # Writable volumes
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/app
      
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
```

---

## Verifying Non-Root

```bash
# Check what user the container runs as
kubectl exec -it my-pod -- id
# Output: uid=1000(app) gid=1000(app) groups=1000(app)

# Try to write to root filesystem (should fail)
kubectl exec -it my-pod -- touch /root-file
# Error: Read-only file system

# Try to install packages (should fail)
kubectl exec -it my-pod -- apt-get update
# Error: Permission denied

# Verify it's actually working
kubectl exec -it my-pod -- whoami
# Output: app
```

---

## Best Practices for Non-Root

1. **Build images as non-root**
   - Use `USER appuser` in Dockerfile
   - Don't rely on Kubernetes to fix root images

2. **Use numeric UIDs**
   ```dockerfile
   RUN useradd -u 1000 appuser  # Numeric UID
   # Not: RUN useradd appuser   # Unpredictable UID
   ```

3. **Combine with other controls**
   ```yaml
   securityContext:
     runAsNonRoot: true
     runAsUser: 1000
     allowPrivilegeEscalation: false       # Can't escalate
     readOnlyRootFilesystem: true          # Can't modify code
     capabilities:
       drop:
       - ALL                               # Drop dangerous capabilities
   ```

4. **Test locally**
   ```bash
   # Build and run locally before deploying
   docker build -t myapp .
   docker run --user 1000:1000 myapp
   ```

5. **Document requirements**
   ```dockerfile
   # In Dockerfile comments
   # This image requires:
   # - UID 1000, GID 1000
   # - Writable /tmp directory
   # - Port 8080 (not 80)
   ```

---

## Summary

Running as non-root is one of the highest-impact security improvements:

**Benefits:**
- Attacker is not root if container is compromised
- Cannot install packages or tools
- Cannot modify application code
- Cannot access sensitive system files
- Much smaller blast radius

**How to do it:**
1. Build image as non-root (best practice)
2. Use `USER appuser` in Dockerfile
3. Use `securityContext: runAsUser: 1000` in Kubernetes
4. Mount writable volumes for /tmp, /var/log, etc.
5. Combine with other controls (readOnlyRootFilesystem, capabilities drop)

**Next:** In the next file, `05-read-only-filesystem.md`, we will learn about making filesystems read-only — another powerful security control.
