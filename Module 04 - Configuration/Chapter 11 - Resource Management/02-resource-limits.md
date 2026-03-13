# Resource Limits

## Introduction

Resource limits set the **hard ceiling** on how much CPU and memory a container can use. Unlike requests (which guide the scheduler), limits are enforced at runtime by the Linux kernel. Exceeding a CPU limit causes throttling. Exceeding a memory limit causes the container to be killed immediately — `OOMKilled`.

Understanding how CPU throttling and OOMKill work differently is critical for diagnosing performance issues and setting appropriate limits.

---

## Setting Limits

```yaml
spec:
  containers:
  - name: api
    image: api-server:v2.0
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "1000m"    # never use more than 1 CPU core
        memory: "1Gi"   # never use more than 1 GiB — OOMKilled if exceeded
```

---

## CPU Limits — Throttling, Not Killing

When a container exceeds its CPU limit, the Linux kernel **throttles** it — the container process is paused until the next CPU scheduling interval (typically 100ms). It does not die. It just runs slower.

```bash
# Watch CPU throttling in real time
kubectl exec -it api-pod -n production -- cat /sys/fs/cgroup/cpu/cpu.stat
# nr_periods 1000
# nr_throttled 150   ← 150 out of 1000 periods were throttled (15% throttling)
# throttled_time 15000000  ← 15ms of throttle time

# Also visible in Prometheus metrics:
# container_cpu_cfs_throttled_seconds_total
# container_cpu_cfs_throttled_periods_total

# Grafana query: CPU throttle rate per container
# rate(container_cpu_cfs_throttled_periods_total{container="api"}[5m])
# / rate(container_cpu_cfs_periods_total{container="api"}[5m]) * 100
```

High CPU throttling (>10–20%) is a performance problem. The container is artificially slowed down, causing:
- Increased HTTP response latency
- Increased gRPC timeout rates
- Cascading delays to downstream services

**Fix: increase the CPU limit** — or better, reduce the CPU usage by optimizing the application.

### CPU Limits and Burstable Workloads

CPU limits also prevent CPU starvation of other Pods on the same node. Without limits, one poorly-written container could consume 100% of node CPU and starve other containers.

---

## Memory Limits — OOMKill

Memory limits are enforced differently. When a container's memory usage exceeds its limit, the Linux kernel's OOM (Out of Memory) Killer **kills the container process immediately** — with exit code 137.

```bash
# Check if a container was OOMKilled
kubectl describe pod api-pod -n production

# Containers:
#   api:
#     Last State:     Terminated
#       Reason:       OOMKilled        ← killed by OOM Killer
#       Exit Code:    137
#       Started:      Mon, 15 Jan 2024 10:00:00 +0000
#       Finished:     Mon, 15 Jan 2024 10:05:00 +0000
#     Restart Count:  3                ← restarted 3 times

kubectl get pod api-pod -n production
# NAME      READY   STATUS             RESTARTS   AGE
# api-pod   0/1     CrashLoopBackOff   3          15m
# ← CrashLoopBackOff from repeated OOMKills
```

OOMKill happens instantly — no graceful shutdown, no SIGTERM. The process is terminated without warning. This is why:
- Memory limits must be set generously enough to handle peak usage plus some headroom
- Applications with memory leaks eventually get OOMKilled even with high limits

### Diagnosing OOMKill Issues

```bash
# Check memory usage before the OOMKill happened
kubectl top pod api-pod -n production --containers

# Check the node's OOM event log
kubectl exec -it api-pod -n production -- dmesg | grep -i "killed process"
# Killed process 12345 (node) total-vm:2048000kB, anon-rss:1050000kB
# ← shows which process was killed and how much memory it was using

# Prometheus — was the container approaching its limit?
# container_memory_working_set_bytes{container="api"}
# vs
# kube_pod_container_resource_limits{resource="memory", container="api"}

# If working_set_bytes was close to the limit → set a higher limit
# If working_set_bytes was low but OOMKill happened → possible memory spike
```

---

## CPU Limit Controversy — When to Skip CPU Limits

There is an active debate in the Kubernetes community about CPU limits. The argument against CPU limits:

**Without CPU limits (only CPU requests):**
- Containers can use spare CPU freely — better resource utilization
- No artificial throttling when the node has free CPU
- Requests still protect against starvation under contention

**With CPU limits:**
- Containers are throttled even when the node has spare CPU
- Throttling causes latency and performance degradation
- Adds complexity without always adding stability

**Recommendation for EKS:**
- **Always set memory limits** — OOMKill is predictable; unlimited memory is dangerous
- **Set CPU limits cautiously** — for latency-sensitive workloads, consider running without CPU limits (only requests); for batch/worker workloads, limits are appropriate

If you remove CPU limits, you MUST set CPU requests to ensure the scheduler allocates capacity fairly across Pods.

---

## Setting Limits Based on Observed Peak Usage

```bash
# See current usage for all containers
kubectl top pods --all-namespaces --containers

# Find the maximum memory usage over 7 days (Prometheus)
# max_over_time(container_memory_working_set_bytes{container="api"}[7d])

# Rule of thumb for limits:
# Memory limit = 2x to 3x the p95 memory usage
# CPU limit    = 2x to 4x the p95 CPU usage
# (gives headroom for spikes without being recklessly generous)

# Example: api-server typical usage: cpu=150m, memory=200Mi
# Set:
#   requests: cpu=200m, memory=256Mi
#   limits:   cpu=500m, memory=512Mi

# Gives:
# CPU: 3x headroom (500m vs 150m typical)
# Memory: 2.5x headroom (512Mi vs 200Mi typical)
```

---

## Ephemeral Storage Limits

Beyond CPU and memory, Kubernetes also manages ephemeral storage (disk space used by container logs, temporary files, and the container's writable layer):

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
    ephemeral-storage: "1Gi"    # request 1 GiB of local disk
  limits:
    cpu: "1000m"
    memory: "1Gi"
    ephemeral-storage: "5Gi"    # evict if container uses more than 5 GiB of disk
```

If a container writes large amounts of log files or temp data and exceeds the `ephemeral-storage` limit, the Pod is evicted. This prevents one Pod from filling the node's disk and crashing the node.

---

## Checking Effective Limits on Running Containers

```bash
# See requests and limits for all Pods in a namespace
kubectl get pods -n production -o json | \
  jq -r '.items[] | .metadata.name as $pod | .spec.containers[] | 
  "\($pod)/\(.name): req.cpu=\(.resources.requests.cpu) req.mem=\(.resources.requests.memory) lim.cpu=\(.resources.limits.cpu) lim.mem=\(.resources.limits.memory)"'

# View detailed resource usage vs limits
kubectl top pods -n production --containers

# NAME              CONTAINER   CPU(cores)   MEMORY(bytes)
# api-pod-abc123    api         150m         200Mi    ← 30% of cpu limit, 40% of memory limit
# web-pod-def456    web         80m          120Mi
# worker-ghi789     worker      600m         450Mi    ← 60% of cpu limit, 88% of memory limit — monitor closely!
```

---

## Summary

- CPU limits cause **throttling** — the container runs slower but does not die; high throttling (>10%) is a performance problem
- Memory limits cause **OOMKill** (exit code 137) — instant process termination with no graceful shutdown; set limits with enough headroom for peak usage
- Detect OOMKilled containers: `kubectl describe pod` shows `Reason: OOMKilled` in `Last State`
- Consider skipping CPU limits (using only requests) for latency-sensitive services — prevents artificial throttling when spare CPU is available
- Memory limits are always necessary — unbounded memory usage can kill the node
- Set limits at 2–3x the p95 observed usage to handle spikes without wasting resources
- Use `ephemeral-storage` limits for containers that write large amounts of temporary data

In the next file, `03-limit-ranges.md`, you will learn about LimitRanges — the namespace-level control that sets default requests and limits automatically, ensuring every Pod in a namespace has resource constraints.
