# emptyDir Volume

## Introduction

`emptyDir` is the simplest and most commonly used volume in Kubernetes. It creates an empty directory when the Pod starts and removes everything in it when the Pod stops. It lives and dies with the Pod, not with any individual container — so if a container restarts inside the Pod, the `emptyDir` data survives the restart.

Despite being temporary, `emptyDir` is incredibly useful for three specific patterns: inter-container file sharing (sidecar), scratch space for processing, and in-memory caching.

---

## The Basic emptyDir

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: processing-pod
  namespace: production
spec:
  volumes:
  - name: work-dir
    emptyDir: {}         # {} means use default settings (on-disk, no size limit)

  containers:
  - name: processor
    image: data-processor:v1.0
    volumeMounts:
    - name: work-dir
      mountPath: /var/work

  - name: output-shipper
    image: shipper:v1.0
    volumeMounts:
    - name: work-dir
      mountPath: /var/work    # same path — both containers see same files
```

Both containers access the same directory. When `processor` writes `/var/work/result.json`, `output-shipper` reads it immediately.

---

## emptyDir Lifetime — Survives Container Restart, Not Pod Restart

This is a common source of confusion:

```
Scenario A: Container restarts (OOMKill, crash, liveness probe failure)
  → emptyDir data SURVIVES the container restart
  → The new container starts and sees all files written by the previous container

Scenario B: Pod is deleted and recreated
  → emptyDir data is GONE
  → New Pod gets a fresh empty directory

Scenario C: Pod is rescheduled to a different node
  → emptyDir data is GONE (it was on the old node)
```

This makes `emptyDir` appropriate for data that:
- Needs to survive container restarts within a Pod (in-progress computations, partial results)
- Does NOT need to survive Pod restarts or node rescheduling

---

## In-Memory emptyDir — `medium: Memory`

Setting `medium: Memory` creates a `tmpfs` filesystem — a RAM-backed filesystem. Reads and writes to `tmpfs` are much faster than disk because they never hit the physical disk:

```yaml
volumes:
- name: fast-cache
  emptyDir:
    medium: Memory
    sizeLimit: 256Mi    # highly recommended — limits RAM usage
```

```bash
# Inside the Pod — verify it's a tmpfs mount
kubectl exec processing-pod -- df -h /cache
# Filesystem      Size  Used Avail Use% Mounted on
# tmpfs           256M     0  256M   0% /cache
# ← 'tmpfs' confirms it's in-memory
```

**Important:** Memory used by a `tmpfs` emptyDir counts against the container's memory limit. If your container has a `memory: 512Mi` limit and uses 256Mi of `tmpfs`, only 256Mi is left for the container process itself.

**When to use `medium: Memory`:**
- Data that needs the fastest possible read/write performance
- Cryptographic key material or tokens that should never touch disk
- Small, frequently accessed caches (< 512Mi)
- Intermediate data in a multi-stage processing pipeline

**When to use disk (default):**
- Large intermediate files that would pressure node memory
- Data where disk performance is acceptable
- When you cannot afford RAM overhead

---

## Size Limits

```yaml
volumes:
# Memory-backed with size limit (mandatory for memory)
- name: ram-cache
  emptyDir:
    medium: Memory
    sizeLimit: 512Mi    # Pod is evicted if it uses more than this

# Disk-backed with size limit
- name: temp-files
  emptyDir:
    sizeLimit: 5Gi      # Pod is evicted if the emptyDir exceeds 5Gi
```

Without a `sizeLimit` on disk-backed emptyDir, a runaway process could fill the entire node's disk — crashing all Pods on that node. **Always set `sizeLimit` in production.**

The Pod is evicted (not killed) when the size limit is exceeded. It is then rescheduled on another node, starting fresh.

---

## Pattern 1: Log Shipping Sidecar

The most common production use of `emptyDir`:

```yaml
spec:
  volumes:
  - name: app-logs
    emptyDir: {}

  containers:
  # Main app writes structured logs to a file
  - name: app
    image: api-server:v2.0
    volumeMounts:
    - name: app-logs
      mountPath: /var/log/app

  # Fluent Bit reads from the same directory and ships to CloudWatch
  - name: log-shipper
    image: amazon/aws-for-fluent-bit:latest
    env:
    - name: LOG_GROUP_NAME
      value: "/production/api-server"
    - name: AWS_REGION
      value: "us-east-1"
    volumeMounts:
    - name: app-logs
      mountPath: /var/log/app
```

The `app` container writes to `/var/log/app/app.log`. Fluent Bit tails the same file and ships log lines to CloudWatch Logs. They use `emptyDir` as the shared bridge.

---

## Pattern 2: Multi-Stage Data Processing

```yaml
spec:
  volumes:
  - name: pipeline-data
    emptyDir:
      sizeLimit: 2Gi

  initContainers:
  # Stage 1: Download the raw data from S3
  - name: download
    image: amazon/aws-cli:latest
    command:
    - /bin/sh
    - -c
    - aws s3 cp s3://my-data-bucket/input.csv /data/input.csv
    volumeMounts:
    - name: pipeline-data
      mountPath: /data

  # Stage 2: Transform the raw data
  - name: transform
    image: data-transformer:v1.0
    command: ["python", "transform.py", "/data/input.csv", "/data/output.json"]
    volumeMounts:
    - name: pipeline-data
      mountPath: /data

  containers:
  # Stage 3: Main container uploads the transformed data
  - name: upload
    image: data-uploader:v1.0
    command: ["python", "upload.py", "/data/output.json"]
    volumeMounts:
    - name: pipeline-data
      mountPath: /data
```

Each stage reads and writes to the same `emptyDir`. The init containers handle download and transform; the main container uploads the final result. All stages share the same directory through the volume.

---

## Pattern 3: Shared Configuration Between Containers

```yaml
spec:
  volumes:
  - name: rendered-config
    emptyDir: {}

  initContainers:
  # Render config from template + env vars before app starts
  - name: config-renderer
    image: envsubst-tools:v1.0
    command:
    - /bin/sh
    - -c
    - |
      envsubst < /templates/config.json.tmpl > /rendered/config.json
      echo "Config rendered successfully"
    env:
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
    volumeMounts:
    - name: rendered-config
      mountPath: /rendered

  containers:
  - name: app
    image: my-app:v1.0
    volumeMounts:
    - name: rendered-config
      mountPath: /etc/app
      readOnly: true           # app reads rendered config
```

The init container renders a configuration template into the `emptyDir`. The main container reads the rendered config as read-only.

---

## Debugging emptyDir Issues

```bash
# Verify the emptyDir is mounted correctly
kubectl exec pod-name -n production -- df -h

# Check what files exist in the volume
kubectl exec pod-name -n production -c app -- ls -la /var/log/app

# Check if the size limit is being approached
kubectl exec pod-name -n production -- du -sh /tmp
# If close to sizeLimit, the Pod will be evicted

# See emptyDir details in Pod describe
kubectl describe pod pod-name -n production
# Volumes:
#   app-logs:
#     Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
#     Medium:
#     SizeLimit:  <unset>   ← no limit set — add one in production!

# Check if Pod was evicted due to disk pressure
kubectl describe node ip-10-0-1-100.ec2.internal | grep -A 5 "Conditions:"
# DiskPressure = True → node is under disk pressure
```

---

## Summary

- `emptyDir` creates an empty directory at Pod start and destroys it when the Pod stops — it survives container restarts within the Pod but not Pod restarts
- Use `medium: Memory` for a `tmpfs` RAM-backed volume — faster than disk but counts against container memory limits
- Always set `sizeLimit` in production — without it, a runaway process can fill the node's disk and crash all Pods on that node
- Three primary patterns: log shipping sidecars (shared log directory), multi-stage processing (pass data between init containers and main container), shared config rendering (init container renders templates for main container)
- Memory-backed emptyDir is appropriate for cryptographic material, small hot caches, and data that should never touch disk

In the next file, `05-hostpath-volume.md`, you will learn about `hostPath` volumes — how they work, why they are dangerous, and the specific scenarios where they are the correct choice (DaemonSets and system-level components).
