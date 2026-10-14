# Monitoring Overview

## Introduction

Logging tells you *what happened*. Monitoring tells you *if something is wrong*.

```
Logging: "Pod CPU was 80%, then 90%, then crashed"
         (historical record)

Monitoring: "Pod CPU at 95%, approaching limit!"
            (real-time alert: fix NOW)
```

Kubernetes monitoring answers:
- What's the CPU and memory usage per pod?
- Which nodes are running hot?
- Is any pod using more than allocated?
- Which deployments have slow response times?

---

## The Three Pillars

```
Metrics      Logs        Traces
│            │           │
├→ What?     ├→ Why?     └→ How?
│  CPU, mem  │  Error    │  Request
│  latency   │  messages │  flow
│  requests  │  debug    │  timeline
```

**Metrics:** Numbers (time series)
**Logs:** Text (events)
**Traces:** Paths (request journey)

This chapter covers metrics.

---

## Kubernetes Built-in Metrics

Kubernetes tracks metrics at two levels:

```
1. Kubelet (on each node)
   Collects per-pod metrics
   CPU, memory, disk I/O, network
   Stored locally

2. API Server
   Can query /metrics endpoint
   But only recent data (15 minutes)
```

**Problem:**
```
Query API server for historical data
"Data not available"
"Only have last 15 minutes"

Can't analyze trends over time
Can't see what happened yesterday
```

---

## What Gets Measured?

### Pod Metrics

```
CPU:         millicores (1000m = 1 full CPU)
Memory:      bytes (1Gi = 1 gigabyte)
Network RX:  bytes received
Network TX:  bytes sent
Disk I/O:    reads/writes per second
```

### Node Metrics

```
CPU:         total available
Memory:      total available
Allocatable: what can be given to pods
Pods:        how many running on node
```

---

## Real-World Scenario: Performance Issue

```
Customer: "App is slow today"
You: "Let me check metrics"

Questions to answer:
- Which pod is slow? (check latency)
- Is it CPU-bound? (check CPU usage)
- Is it memory-bound? (check memory)
- How many requests per second? (check throughput)
- When did it start? (check timeline)

With proper monitoring:
- See spike at 10:30 AM
- CPU jumped from 20% to 95%
- Coincides with new deployment
- Root cause: New code is inefficient
```

---

## Metrics Components

```
Kubelet (collects)
   ↓ exposes /metrics endpoint
   ↓
Metrics Server (aggregates)
   ↓ stores in memory (short-term)
   ↓
Prometheus (scrapes)
   ↓ pulls metrics periodically
   ↓
Time-series database
   ↓ stores historical data
   ↓
Grafana (visualizes)
   ↓ shows dashboards
   ↓
PromQL queries
   ↓ complex analysis
```

---

## Metrics Server

Metrics Server is required for:
- HPA (Horizontal Pod Autoscaler)
- VPA (Vertical Pod Autoscaler)
- CA (Cluster Autoscaler)
- kubectl top (viewing resource usage)

```bash
# Check if Metrics Server running
kubectl get deployment -n kube-system | grep metrics

# If not running: Install it
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## Using kubectl top

```bash
# View node resource usage
kubectl top nodes

# View pod resource usage
kubectl top pods -n default

# View pod with containers
kubectl top pods -n default --containers

# View all namespaces
kubectl top pods --all-namespaces

# Example output:
NAME               CPU(cores)   MEMORY(bytes)
web-app-pod-1      150m         256Mi
db-pod-1           500m         1Gi
```

---

## Common Metrics

```
container_cpu_usage_seconds_total        CPU time used
container_memory_usage_bytes             Memory used
container_network_receive_bytes_total    Network received
container_network_transmit_bytes_total   Network sent
container_fs_usage_bytes                 Disk space used
container_fs_limit_bytes                 Disk limit

Pod metrics (aggregated):
pod_cpu_usage                            Total pod CPU
pod_memory_usage                         Total pod memory
pod_network_io_bytes                     Total pod network
```

---

## HPA Metrics

HPA needs metrics to make scaling decisions:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  
  minReplicas: 1
  maxReplicas: 10
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # Scale at 70% CPU
```

Without Metrics Server, HPA can't calculate CPU and stays idle.

---

## Real-World Timeline

```
Time 0:00 — App running normally
  Metrics Server: ✓ Collecting
  CPU/pod: 10%
  Memory/pod: 256Mi

Time 1:00 — Traffic spike
  HPA detects CPU 75%
  Metrics Server provides data
  HPA: "Scale up"
  Replicas: 1 → 3

Time 1:30 — More traffic
  HPA detects CPU 80%
  Replicas: 3 → 10

Time 2:00 — Traffic subsides
  CPU drops to 20%
  HPA: "Scale down"
  Replicas: 10 → 1

All decisions based on metrics from Metrics Server.
```

---

## Limitations of Metrics Server

```
Metrics Server stores in memory
Data older than 15 minutes: Lost

If Metrics Server crashes:
  Recent metrics lost
  HPA can't scale (no data)
  Pods accumulate

Solution: Long-term monitoring
  Prometheus (stores weeks/months)
  Retains historical data
```

---

## Summary

Monitoring overview:

**Three pillars:**
- Metrics (numbers/time-series)
- Logs (events/text)
- Traces (request flows)

**Kubernetes metrics:**
- Per-pod CPU, memory, network
- Per-node usage

**Components:**
- Metrics Server: Short-term aggregation
- Prometheus: Long-term storage
- Grafana: Visualization

**Usage:**
- HPA: Automatic scaling
- VPA: Resource optimization
- kubectl top: Manual inspection

**Next:** File 02 covers Metrics Server setup.
