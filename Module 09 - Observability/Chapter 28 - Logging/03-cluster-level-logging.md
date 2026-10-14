# Cluster-Level Logging

## The Problem With Node-Only Logging

```
Node logs are deleted after rotation
You need logs from 3 days ago
Result: Can't investigate

Node crashes
All logs on that node gone
Other nodes still have their logs
Inconsistent picture of what happened
```

**Cluster-level logging** centralizes all logs:

```
Every Node                     Central Aggregator          Central Storage
┌──────────────┐             ┌──────────────┐            ┌──────────────┐
│ Node 1       │ ──────────→ │              │ ──────────→ │ Elasticsearch│
│ /var/log/*   │             │  Fluentd     │            │              │
└──────────────┘             │  (or Stack)  │            │ 7+ years of  │
                             │              │            │ logs stored  │
┌──────────────┐             │              │            └──────────────┘
│ Node 2       │ ──────────→ │              │
│ /var/log/*   │             └──────────────┘
└──────────────┘
       :
┌──────────────┐
│ Node N       │ ──────────→
└──────────────┘
```

---

## ELK Stack Overview

**ELK = Elasticsearch + Logstash + Kibana**

```
E - Elasticsearch    Storage and search engine
L - Logstash         Log processor and parser
K - Kibana           Web UI for searching logs
```

Modern version: **Fluentd** replaces Logstash (lighter weight).

---

## Cluster Logging Architecture

```
1. Fluentd DaemonSet (runs on every node)
   ↓ Reads /var/log/containers/
   ↓ Parses JSON, enriches with metadata
   ↓

2. Elasticsearch (central storage)
   ↓ Stores indexed logs
   ↓ Queryable by timestamp, pod, namespace, etc.
   ↓

3. Kibana (search UI)
   ↓ Web interface to search
   ↓ Create dashboards
   ↓ Set up alerts
```

---

## How Logs Flow

### Example: Pod Crashes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: crash-test
  namespace: default
spec:
  containers:
  - name: app
    image: bad-app
```

When pod crashes:

```
1. Container writes error to stdout
   "ERROR: Out of memory"

2. Docker captures
   Written to /var/lib/docker/containers/...

3. kubelet sees it
   Creates symlink in /var/log/containers/

4. Fluentd reads
   Tail /var/log/containers/crash-test*.log
   Sees new lines

5. Parses metadata
   {
     "pod": "crash-test",
     "namespace": "default",
     "node": "node-1",
     "log": "ERROR: Out of memory"
   }

6. Sends to Elasticsearch
   Indexed and searchable

7. Later: Search in Kibana
   Time: 2024-01-15 10:30:00
   Pod: crash-test
   → Find root cause from logs
```

---

## Elasticsearch Basics

Elasticsearch stores logs in indexes:

```
Index structure:
kubernetes-2024-01-15
├── shard 1 (partition)
├── shard 2
└── replica 1 (backup)

Each index contains documents (log lines):
{
  "_id": "abc123",
  "timestamp": "2024-01-15T10:30:45Z",
  "kubernetes": {
    "pod_name": "web-app",
    "namespace": "default",
    "node_name": "node-1"
  },
  "log": "User request received"
}
```

---

## Retention Policy

Elasticsearch stores logs with TTL (Time To Live):

```yaml
# Configuration
retention_days: 30       # Keep 30 days
retention_size: 500GB    # Or 500GB, whichever comes first

# Automatic
- Day 1-30: Actively searchable
- Day 31: Deleted by policy
- Before deletion: Can archive to S3
```

Typical production: 30-90 days retention.

---

## Querying Logs in Kibana

```
Kibana Search Syntax:

Find all error logs:
kubernetes.pod_name:web-app AND level:ERROR

Find logs in specific namespace:
kubernetes.namespace:production

Find logs from specific time:
@timestamp:[2024-01-15T10:00:00Z TO 2024-01-15T11:00:00Z]

Combine:
kubernetes.namespace:production AND message:*timeout* AND level:ERROR
```

---

## Real-World Scenario: Investigating Production Outage

```
Timeline:
2024-01-15 10:30 AM - Customer reports service down
2024-01-15 10:35 AM - You check Kubernetes status
2024-01-15 10:40 AM - All pods running, but no traffic

Investigation in Kibana:

1. Search for errors in past 30 minutes:
   level:ERROR AND @timestamp:["now-30m" TO "now"]
   
   Result: 1000 errors from payment-service pods

2. Filter by specific pod:
   kubernetes.pod_name:payment-service* AND level:ERROR
   
   Result: "Database connection timeout"

3. Check database pod logs:
   kubernetes.namespace:infrastructure AND level:ERROR
   
   Result: "Storage full, can't write to disk"

4. Find when storage filled:
   kubernetes.pod_name:postgres* AND message:*storage*
   
   Result: Filled at 10:20 AM

5. Root cause found: Disk full → Database stopped → Timeouts
```

---

## Common Mistakes

### Mistake 1: No Retention Policy

```yaml
# WRONG — logs grow forever
elasticsearch_config:
  # No TTL configured
  # Disk eventually fills up
  # Cluster crashes
```

**Fix:** Set retention policy.

```yaml
# RIGHT
lifecycle:
  policies:
  - policy_name: my-policy
    phases:
      hot:
        min_age: 0d
      delete:
        min_age: 30d
```

### Mistake 2: Logging Sensitive Data

```
# WRONG — logs contain passwords
"DEBUG: Database password is abc123"
"INFO: User token: eyJhbGciOiJIUzI1NiIs..."

# Logs are searchable by anyone with Kibana access
# Security breach if logs indexed
```

**Fix:** Filter sensitive data before logging.

```python
# RIGHT
# Redact passwords, tokens, API keys
log.info(f"Database connection failed for user {user_id}")
# Don't log actual credentials
```

---

## Log Parsing and Enrichment

Fluentd parses logs and adds metadata:

```yaml
# Raw log from /var/log
{
  "log": "INFO: Request received from 1.2.3.4\n",
  "stream": "stdout",
  "time": "2024-01-15T10:30:45.123Z"
}

# After Fluentd enrichment
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "INFO",
  "message": "Request received from 1.2.3.4",
  "kubernetes": {
    "pod_name": "web-app",
    "pod_namespace": "default",
    "pod_id": "abc123",
    "node_name": "node-1",
    "labels": {
      "app": "web",
      "version": "v1.2"
    }
  },
  "source": "stdout"
}
```

Much richer for searching and analysis!

---

## Summary

Cluster-level logging:

**Architecture:**
- Fluentd (DaemonSet) collects from all nodes
- Elasticsearch stores centrally
- Kibana provides search interface

**Flow:**
- Pod logs → Node /var/log → Fluentd → Elasticsearch → Kibana

**Benefits:**
- Centralized view of all logs
- Long-term retention (30+ days)
- Searchable by pod, namespace, time, level
- Survives node failures

**Next:** File 04 covers Fluentd setup.
