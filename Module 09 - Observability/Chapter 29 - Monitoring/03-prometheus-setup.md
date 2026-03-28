# Prometheus Setup

## What is Prometheus?

Prometheus stores metrics long-term:

```
Metrics Server (15 minutes): "Pod used 150m CPU"
Prometheus (weeks): "Pod used 150m on Monday, 200m on Tuesday"
```

Prometheus:
- Scrapes metrics from targets
- Stores in time-series database
- Queries via PromQL language
- Works with Grafana for dashboards

---

## Architecture

```
Kubelet (exposes /metrics)
    ↓ Prometheus scrapes (every 15s)
    ↓
Prometheus Server
    ↓ stores time-series data
    ↓
PromQL Queries
    ↓
Grafana (visualization)
```

---

## Install Prometheus

### Option 1: Helm (Recommended)

```bash
# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

Includes:
- Prometheus
- Grafana
- Alertmanager
- Node Exporter

### Option 2: Manifest

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  type: ClusterIP
  ports:
  - port: 9090
    targetPort: 9090
  selector:
    app: prometheus
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        args:
          - '--config.file=/etc/prometheus/prometheus.yml'
          - '--storage.tsdb.path=/prometheus'
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
        - name: storage
          mountPath: /prometheus
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 1Gi
      volumes:
      - name: config
        configMap:
          name: prometheus-config
      - name: storage
        emptyDir: {}
```

---

## Prometheus Configuration

```yaml
# ConfigMap with scrape config
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    scrape_configs:
    
    # Scrape Kubernetes API server
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    
    # Scrape kubelet
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    
    # Scrape pod metrics
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
```

---

## Accessing Prometheus

```bash
# Port forward
kubectl port-forward -n monitoring svc/prometheus 9090:9090

# Open browser
# http://localhost:9090
```

Prometheus UI:
- Graph: Query metrics
- Status: Check scrape targets
- Alerts: View alerting rules

---

## PromQL Queries

Query language for metrics:

```promql
# Simple query: CPU usage
container_cpu_usage_seconds_total

# Gauge: Current memory
container_memory_usage_bytes

# Rate: CPU usage per second (5m average)
rate(container_cpu_usage_seconds_total[5m])

# Filter by pod name
container_memory_usage_bytes{pod_name="web-app"}

# Filter by namespace
container_cpu_usage_seconds_total{namespace="default"}

# Multiple filters
rate(container_cpu_usage_seconds_total[5m]){pod_name="web-app", namespace="production"}

# Aggregate across pods
sum(rate(container_cpu_usage_seconds_total[5m]))

# Top 5 memory users
topk(5, container_memory_usage_bytes)

# Alert: Memory > 1Gb
container_memory_usage_bytes > 1000000000
```

---

## Real-World Scenario: Performance Investigation

```bash
# Access Prometheus UI
kubectl port-forward -n monitoring svc/prometheus 9090:9090

# Query 1: Which pod is using most CPU?
topk(5, rate(container_cpu_usage_seconds_total[5m]))

# Result: web-app-abc123 using 0.8 cores

# Query 2: Pod CPU trend (last 1 hour)
rate(container_cpu_usage_seconds_total[5m]){pod_name="web-app-abc123"}

# Result: Gradually increasing, 0.1 → 0.8 cores

# Query 3: Memory leaking?
container_memory_usage_bytes{pod_name="web-app-abc123"}

# Result: 256Mi → 512Mi → 1Gi (memory leak confirmed)

# Conclusion: Memory leak in new deployment
```

---

## Retention and Disk Usage

```yaml
# Configure retention
prometheus:
  prometheusSpec:
    retention: 15d          # Keep 15 days of data
    retentionSize: "50Gi"   # Or 50GB, whichever comes first
    storageSpec:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 50Gi
```

---

## Common Issues

### Issue 1: Prometheus Won't Scrape Targets

```bash
# Check Status → Targets in Prometheus UI
# Red "DOWN" targets?

# Check logs
kubectl logs -n monitoring prometheus-0

# Common fixes:
# 1. ServiceMonitor not created
# 2. Pod not exposing /metrics endpoint
# 3. Port number incorrect
```

### Issue 2: High Memory Usage

```bash
# Prometheus stores in memory
# Reduce retention or increase pod resources

# Reduce retention
retention: 7d  # Instead of 15d

# Increase memory
resources:
  limits:
    memory: 2Gi
```

---

## Summary

Prometheus setup:

**Purpose:**
- Long-term metrics storage
- Historical data (weeks/months)
- PromQL queries

**Install:**
- Helm (recommended) or manifest

**Configuration:**
- Scrape targets (Kubernetes components)
- Retention policy (15-90 days)
- Storage size

**Usage:**
- PromQL queries for analysis
- Dashboard source (Grafana)

**Next:** File 04 covers Grafana dashboards.
