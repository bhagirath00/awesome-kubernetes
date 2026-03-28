# Logging Architecture in Kubernetes

## Introduction

You just deployed an application to Kubernetes. A customer reports a bug. You need logs.

```
User: "Something's wrong!"
You: "Let me check the logs..."
You: kubectl logs pod-name
  ...only 100 lines, pod restarted yesterday
You: "I can't see what happened"
```

Kubernetes logs are temporary. A pod crashes, logs disappear. A node dies, all logs gone.

**Centralized logging** solves this:

```
Pod 1: Write log → Fluentd
Pod 2: Write log → Fluentd
Pod 3: Write log → Fluentd
  ↓
Fluentd (log processor)
  ↓
Elasticsearch (central storage)
  ↓
Kibana (search and visualize)
```

This chapter covers how logs flow through Kubernetes and how to capture them before they disappear.

---

## The Problem With Default Logging

### Docker Container Logs

```bash
# Pod logs are written to Docker
kubectl logs my-pod
# Shows last 100 lines from container stdout/stderr
```

**What happens:**
1. Container writes to stdout/stderr
2. Docker runtime captures it
3. Stored in `/var/lib/docker/containers/...`
4. Pod deleted? Logs deleted.

**Real scenario:**
```
Pod runs for 3 days
Performance issue discovered
Need to analyze logs from 8 hours ago
Result: Logs already rotated out
Can't diagnose problem
```

---

## Three-Layer Logging Architecture

```
Layer 1: Pod (Application)
  App writes to stdout
  Example: "User login from 1.2.3.4"

Layer 2: Node (Docker/kubelet)
  kubelet captures stdout/stderr
  Stores temporarily (/var/log/containers/)
  Logs rotate after size limit

Layer 3: Cluster (Centralized)
  Log aggregator (Fluentd, Logstash) reads from all nodes
  Sends to storage (Elasticsearch, S3)
  Permanent retention
```

---

## What Gets Logged?

```yaml
# Everything written to stdout/stderr
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: app:v1
    
# This pod's logs:
# - Anything printed to console
# - Errors and exceptions
# - Application debug output
```

**Example application logs:**

```python
# Python app logging
import logging
logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)

logger.info("App started")           # → stdout
logger.error("Database error")       # → stderr
```

All of this goes to `kubectl logs`.

---

## Node-Level Logging

Each node runs kubelet. Kubelet captures container stdout/stderr.

```
Node Structure:
/var/log/
├── containers/              # Pod logs
│   ├── pod-name_namespace_containerid.log
│   ├── pod-name_namespace_containerid.log.1 (rotated)
│   └── pod-name_namespace_containerid.log.2
├── pods/                   # Pod events
└── kubelet.log             # Kubelet system log
```

---

## Log Retention

By default, logs are kept for a limited time:

```bash
# Check node log configuration
ssh node-name
ls -lh /var/log/containers/

# Old logs are compressed and removed
# Keep: Last 5 rotations (default)
# Size: Up to 100MB per container (default)
```

**Problem:**
```
You investigate production issue 1 week later
Logs already purged
No historical data
```

---

## Cluster-Level Logging (Centralized)

Solution: Aggregate logs from all nodes to central storage.

```
Architecture:

All Nodes                          Log Aggregator            Storage
┌──────────────┐                 ┌──────────────┐          ┌──────────┐
│ Node1 Logs   │ ────────────→   │ Fluentd      │ ───────→ │Elasticsearch
│ Node2 Logs   │ ────────────→   │ (collects)   │          │ (stores)
│ Node3 Logs   │ ────────────→   │              │ ───────→ │Kibana
└──────────────┘                 └──────────────┘          │ (search)
                                                            └──────────┘
```

Log aggregator:
1. Runs on each node (DaemonSet)
2. Reads container logs
3. Parses and enriches
4. Sends to central storage
5. Permanently stored

---

## Common Logging Patterns

### Pattern 1: Sidecar Container

```yaml
# Extra container in pod reads app logs
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
  - name: app
    image: app:v1
    volumeMounts:
    - name: logs
      mountPath: /app/logs
  
  - name: log-shipper    # Sidecar
    image: filebeat:latest
    volumeMounts:
    - name: logs
      mountPath: /app/logs
  
  volumes:
  - name: logs
    emptyDir: {}
```

App writes to file, sidecar reads file and ships to Elasticsearch.

---

## Common Mistakes

### Mistake 1: Log to File Only

```yaml
# WRONG — logs won't be captured
containers:
- name: app
  image: app:v1
  # App writes to /app/logs/app.log
  # kubectl logs won't show anything
  # Only visible in DaemonSet log collection
```

**Fix:** Write to stdout/stderr.

```python
# RIGHT
sys.stdout.write("Log message")
# Now visible in kubectl logs AND collected by aggregator
```

### Mistake 2: No Structured Logging

```json
// WRONG — unstructured
"Error occurred, check database"

// RIGHT — structured JSON
{
  "timestamp": "2024-01-01T10:00:00Z",
  "level": "error",
  "component": "database",
  "message": "Connection failed",
  "error_code": 500
}
```

Structured logs can be parsed and filtered.

---

## Log Levels

```
DEBUG   - Detailed troubleshooting info
INFO    - Normal operation events  
WARNING - Unexpected but handled situations
ERROR   - Error conditions
FATAL   - Critical failure, app stops
```

**Example:**

```yaml
# Configure log level at startup
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
      - name: app
        image: app:v1
        env:
        - name: LOG_LEVEL
          value: INFO    # Only INFO and above
```

---

## Kubectl Logs Command

```bash
# View pod logs
kubectl logs pod-name

# View container logs (multi-container pod)
kubectl logs pod-name -c container-name

# View last N lines
kubectl logs pod-name --tail=50

# Stream live logs
kubectl logs pod-name -f

# View deleted pod logs (if node still has them)
kubectl logs pod-name --previous

# Get logs from all pods in deployment
kubectl logs deployment/app-name

# Filter by label
kubectl logs -l app=web
```

---

## Logging Best Practices So Far

1. **Log to stdout/stderr** (not files)
2. **Use structured logging** (JSON format)
3. **Set appropriate log level** (avoid DEBUG in prod)
4. **Add context** (request ID, user ID)
5. **Avoid logging sensitive data** (passwords, tokens)

---

## Summary

Kubernetes logging has three layers:

**Layer 1 - Pod:**
- App writes to stdout/stderr
- Short-term only

**Layer 2 - Node:**
- kubelet captures logs
- Stores temporarily (/var/log)
- Gets deleted after rotation

**Layer 3 - Cluster:**
- Log aggregator collects from all nodes
- Sends to central storage
- Permanent retention

**Next:** File 02 covers node-level logging in detail.
