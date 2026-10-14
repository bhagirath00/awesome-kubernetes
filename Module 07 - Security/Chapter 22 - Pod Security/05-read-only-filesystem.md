# Read-Only Filesystems

## Introduction

Imagine a container has a vulnerability. An attacker gains code execution. What can they do?

If the filesystem is writable, they can:
- Modify application code (inject malware)
- Install backdoors
- Plant rootkits
- Modify logs (cover their tracks)
- Change configuration

If the filesystem is read-only, they cannot do any of this. They are trapped.

**Read-only filesystems** are a powerful defense. They transform a compromised container from "fully controlled by attacker" to "very limited attacker."

---

## How Read-Only Filesystems Work

A read-only filesystem means the container cannot write to disk. Files can be read, but not modified.

### Container Tries to Write

```bash
$ echo "malware" > /app/main.py
/app/main.py: Read-only file system
```

Fails. The application code cannot be modified.

### What the Container CAN Still Do

- Read application code (needed)
- Read configuration files (needed)
- Read environment variables (needed)
- Execute applications (needed)

### What the Container CANNOT Do

- Modify code
- Create new files (except on writable volumes)
- Delete files
- Modify permissions
- Install packages

---

## Enabling Read-Only Filesystems

### In Kubernetes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      readOnlyRootFilesystem: true  # ← Read-only
```

Now the entire root filesystem is read-only.

### With Writable Volumes

Most applications need to write somewhere (temp files, logs, cache). Use volumes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  securityContext:
    runAsUser: 1000
  
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      readOnlyRootFilesystem: true
    
    # Mount writable volumes
    volumeMounts:
    - name: tmp
      mountPath: /tmp              # Writable
    - name: var-tmp
      mountPath: /var/tmp          # Writable
    - name: cache
      mountPath: /var/cache/app    # Writable
    - name: logs
      mountPath: /var/log/app      # Writable
  
  volumes:
  - name: tmp
    emptyDir: {}
  - name: var-tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: logs
    emptyDir: {}
```

---

## Real-World Scenarios

### Scenario 1: Web Application

```dockerfile
# Python web application
FROM python:3.11

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

EXPOSE 8080
CMD ["python", "app.py"]
```

**In Kubernetes:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  
  template:
    spec:
      containers:
      - name: web
        image: web-app:latest
        securityContext:
          readOnlyRootFilesystem: true
        
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: tmp-python
          mountPath: /tmp/.python-eggs  # Python bytecode
      
      volumes:
      - name: tmp
        emptyDir: {}
      - name: tmp-python
        emptyDir: {}
```

Even if the application is hacked, the attacker cannot modify the application code.

### Scenario 2: Java Application

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: java-app
spec:
  containers:
  - name: app
    image: java-app:latest
    securityContext:
      readOnlyRootFilesystem: true
    
    volumeMounts:
    # Java needs writable directories
    - name: tmp
      mountPath: /tmp
    - name: java-tmp
      mountPath: /var/tmp
    - name: java-cache
      mountPath: /root/.cache  # Java cache
  
  volumes:
  - name: tmp
    emptyDir: {}
  - name: java-tmp
    emptyDir: {}
  - name: java-cache
    emptyDir: {}
```

### Scenario 3: Application with Logs

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-logs
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      readOnlyRootFilesystem: true
    
    volumeMounts:
    # Application logs
    - name: logs
      mountPath: /var/log/myapp
    # Temp storage
    - name: tmp
      mountPath: /tmp
  
  # Separate sidecar to collect logs
  - name: log-forwarder
    image: filebeat:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/myapp
      readOnly: true  # Read-only for log collector
  
  volumes:
  - name: logs
    emptyDir: {}
  - name: tmp
    emptyDir: {}
```

Logs go to a writable volume, and a sidecar forwards them to Elasticsearch.

---

## Common Issues with Read-Only Filesystems

### Issue 1: Application Crashes - "Permission Denied"

```
Error: open /var/lib/app/config.cache: Permission denied
```

The application tries to write but root filesystem is read-only.

**Solution:** Mount a writable volume

```yaml
volumeMounts:
- name: cache
  mountPath: /var/lib/app/cache  # Writable
```

### Issue 2: Database Can't Write

```
Error: database locked
```

Databases need to write to disk (data files, logs). Can't use read-only.

**Options:**

1. **Run database with writable filesystem**
   ```yaml
   securityContext:
     readOnlyRootFilesystem: false  # Allow writes for database
     runAsUser: 1000
     allowPrivilegeEscalation: false
   ```

2. **Use persistent volume**
   ```yaml
   volumeMounts:
   - name: data
     mountPath: /var/lib/postgresql
   
   volumes:
   - name: data
     persistentVolumeClaim:
       claimName: postgres-data
   ```

### Issue 3: Container Cannot Create Temp Files

```
Error: /tmp: Read-only file system
```

The application tries to use `/tmp` but it's read-only.

**Solution:** Mount `/tmp` as writable

```yaml
volumeMounts:
- name: tmp
  mountPath: /tmp

volumes:
- name: tmp
  emptyDir: {}
```

---

## Combining with Other Security Controls

Read-only filesystem is most powerful when combined with other controls:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      readOnlyRootFilesystem: true              # ← Can't write
      allowPrivilegeEscalation: false           # ← Can't escalate
      runAsNonRoot: true                        # ← Not root
      capabilities:
        drop:
        - ALL                                   # ← No capabilities
        add:
        - NET_BIND_SERVICE                      # ← Only what's needed
    
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  
  volumes:
  - name: tmp
    emptyDir: {}
```

An attacker that breaks in cannot:
- ✗ Write to filesystem
- ✗ Escalate to root
- ✗ Install tools
- ✗ Call dangerous system functions

They are very contained.

---

## Filesystem Layers

When a container has read-only root filesystem but writable volumes:

```
Container's View of Filesystem:

/                    (read-only)
├── app/              (read-only) — application code
├── etc/              (read-only) — config files
├── usr/              (read-only) — system binaries
│
├── tmp/              (writable) ← emptyDir
├── var/cache/        (writable) ← emptyDir
├── var/log/          (writable) ← emptyDir
└── data/             (writable) ← persistentVolume
```

---

## Testing Read-Only Filesystems

```bash
# Try to write to root filesystem
kubectl exec -it my-pod -- touch /test.txt
# Error: Read-only file system

# Verify tmp is writable
kubectl exec -it my-pod -- touch /tmp/test.txt
# Success

# Check filesystem permissions
kubectl exec -it my-pod -- mount | grep "ro,"
# /dev/mapper/root on / type ext4 (ro,nosuid,nodev,noexec,...)

# Check mounted volumes
kubectl exec -it my-pod -- mount | grep "rw,"
# /dev/tmpfs on /tmp type tmpfs (rw,size=...)
```

---

## Best Practices for Read-Only Filesystems

1. **Start with read-only**
   ```yaml
   readOnlyRootFilesystem: true  # Default
   ```

2. **Mount only what's needed as writable**
   ```yaml
   volumeMounts:
   - name: tmp
     mountPath: /tmp
   - name: logs
     mountPath: /var/log/app
   # NOT /data, /config, /app
   ```

3. **Use separate volumes for different concerns**
   ```yaml
   volumeMounts:
   - name: tmp        # Temp files
     mountPath: /tmp
   - name: logs       # Application logs
     mountPath: /var/log/app
   - name: cache      # Cache files
     mountPath: /var/cache/app
   ```

4. **Make temp directories unique per pod**
   ```yaml
   volumes:
   - name: tmp
     emptyDir: {}    # Fresh for each pod instance
   ```

5. **For databases, allow writes to data directory only**
   ```yaml
   securityContext:
     readOnlyRootFilesystem: false  # Can write
   
   volumeMounts:
   - name: data
     mountPath: /data  # Only data
   # Can't write to /app, /etc, /usr
   ```

---

## Complete Example: Production-Ready Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 3
  
  template:
    spec:
      # Pod-level security
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
      
      containers:
      - name: api
        image: api-server:v1.2.3
        
        # Container-level security
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true         # ← Read-only
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        
        ports:
        - containerPort: 8080
        
        # Writable volumes
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/api
        - name: logs
          mountPath: /var/log/api
        
        # Resource limits
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
      
      # Define volumes
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
      - name: logs
        emptyDir: {}
```

This pod:
- ✅ Runs as non-root (UID 1000)
- ✅ Cannot escalate to root
- ✅ Root filesystem is read-only
- ✅ Only has NET_BIND_SERVICE capability
- ✅ Has temp storage for necessary writes
- ✅ Has resource limits
- ✅ Has health checks
- ✅ Very resistant to compromise

---

## Summary

Read-only filesystems prevent attackers from:
- Modifying application code
- Installing backdoors
- Changing configuration
- Covering tracks

**To implement:**
1. Add `readOnlyRootFilesystem: true` to SecurityContext
2. Mount writable volumes for `/tmp`, `/var/log`, `/var/cache`
3. Use `emptyDir` for temporary storage
4. Use `persistentVolume` for permanent data

**Best combined with:**
- Non-root user
- No privilege escalation
- Dropped Linux capabilities
- Resource limits

Read-only filesystems are one of the most effective security controls. Use them everywhere except system pods and databases.

---

## Module 7 Chapter 22 Summary

Chapter 22 covered pod-level security:

- **File 1**: SecurityContext — OS-level security settings
- **File 2**: Pod Security Standards — policy layer
- **File 3**: Pod Security Admission — enforcement controller
- **File 4**: Non-Root Containers — running as non-root user
- **File 5 (this)**: Read-Only Filesystems — preventing writes

In the next chapter, **Chapter 23 - Network Policies**, we will learn how to control network traffic between pods.
