# Grafana Dashboards

## What is Grafana?

Grafana visualizes metrics from Prometheus:

```
Prometheus (metrics database)
       ↓
    Grafana (visualizes)
       ↓
   Dashboard (graphs)
```

Grafana:
- Queries Prometheus
- Shows graphs and charts
- Real-time alerting
- Shared dashboards

---

## Install Grafana

Already included if you used `kube-prometheus-stack` Helm chart:

```bash
# Check if running
kubectl get svc -n monitoring | grep grafana

# If not, install standalone
helm install grafana grafana/grafana \
  -n monitoring \
  --set adminPassword=admin123 \
  --set datasources."datasources\.yaml".datasources[0].url=http://prometheus:9090 \
  --set service.type=LoadBalancer
```

---

## Access Grafana

```bash
# Get password (if using Helm defaults)
kubectl get secret -n monitoring grafana -o jsonpath="{.data.admin-password}" | base64 -d

# Port forward
kubectl port-forward -n monitoring svc/grafana 3000:80

# Open browser
# http://localhost:3000
# Username: admin
# Password: (from above)
```

---

## Connect Prometheus Data Source

1. Click "Configuration" → "Data Sources"
2. Click "Add data source"
3. Select "Prometheus"
4. URL: `http://prometheus:9090`
5. Click "Save & Test"

---

## Create Dashboard

### Method 1: Pre-built Dashboard

```bash
# Import from Grafana dashboard library
# Dashboard ID: 6417 (Kubernetes Cluster Monitoring)
# 
# Steps:
# 1. Click "+" → "Import"
# 2. Enter ID: 6417
# 3. Click "Load"
# 4. Select Prometheus data source
# 5. Click "Import"
```

### Method 2: Build Custom Dashboard

1. Click "+" → "New Dashboard"
2. Click "Add new panel"
3. Configure query:
   ```promql
   rate(container_cpu_usage_seconds_total[5m])
   ```
4. Set visualization (graph, table, gauge)
5. Save

---

## Common Panels

### CPU Usage

```yaml
Panel Type: Graph
Metric: rate(container_cpu_usage_seconds_total[5m])
Title: CPU Usage per Pod
Filter: namespace="production"
```

### Memory Usage

```yaml
Panel Type: Graph
Metric: container_memory_usage_bytes / 1024 / 1024 / 1024
Title: Memory Usage (GB)
Alias: "{{pod_name}}"
```

### Request Latency

```yaml
Panel Type: Heatmap
Metric: histogram_quantile(0.95, request_duration_seconds_bucket)
Title: 95th Percentile Latency
```

### Pod Count

```yaml
Panel Type: Stat
Metric: count(container_last_seen{pod_name!=""})
Title: Total Pods Running
```

---

## Real-World Dashboard Example

```yaml
# Production monitoring dashboard
Rows:
1. Cluster Overview
   - Total pods running (stat)
   - Total memory used (gauge)
   - Total CPU used (gauge)
   - Node count (stat)

2. Pod Performance
   - CPU per pod (table)
   - Memory per pod (table)
   - Network I/O per pod (graph)

3. Errors
   - Error rate per service (graph)
   - Pod restarts (counter)
   - OOMKilled events (table)

4. Alerts
   - Firing alerts (table)
   - Alert history (graph)
```

---

## Alerts from Prometheus

Configure alerts in Prometheus:

```yaml
# PrometheusRule
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
  namespace: monitoring
spec:
  groups:
  - name: app
    interval: 30s
    rules:
    - alert: HighCPU
      expr: rate(container_cpu_usage_seconds_total[5m]) > 0.8
      for: 5m
      annotations:
        summary: "Pod {{ $labels.pod_name }} CPU high"
    
    - alert: OOMKilled
      expr: container_last_seen{reason="OOMKilled"} > 0
      for: 1m
      annotations:
        summary: "Pod {{ $labels.pod_name }} out of memory"
```

---

## Sharing Dashboards

```bash
# Export dashboard
1. Click dashboard settings (gear icon)
2. Click "JSON Model"
3. Copy JSON
4. Save to file: dashboard.json

# Import in another Grafana
1. Click "+" → "Import"
2. Paste JSON
3. Or upload file

# Share URL
1. Click "Share" (top right)
2. Get dashboard URL
3. Share with team
```

---

## Best Practices

1. **Meaningful Titles and Descriptions**
   - What metric? What does it mean?
   - Expected range?

2. **Consistent Layout**
   - Critical metrics at top
   - Detailed info below

3. **Color Coding**
   - Green: OK
   - Yellow: Warning
   - Red: Critical

4. **Set Thresholds**
   - When should alert?
   - Visual boundary lines

5. **Test Dashboards**
   - Verify metrics before sharing
   - Check data freshness

---

## Troubleshooting Dashboards

### Data Not Showing

```
Problem: Graph shows no data
1. Check Prometheus connection
2. Verify metric exists
3. Check query syntax
4. Verify data source has data
```

### Slow Dashboard

```
Problem: Dashboard loads slowly
1. Reduce time range
2. Aggregate data (use rate/avg)
3. Increase Prometheus scrape interval
4. Reduce panel count
```

### Wrong Data

```
Problem: Numbers don't match kubectl top
1. Check Prometheus retention
2. Verify scrape interval
3. Check metric units
4. Verify labels/filters
```

---

## Integration with Alerts

```yaml
# Grafana can show Prometheus alerts
# Configure in Grafana:
# Administration → Alerting → Contact points

# Alert rule in Prometheus fires
# Grafana displays in dashboard
# Can send notifications (Slack, PagerDuty, etc.)
```

---

## Summary

Grafana dashboards:

**Purpose:**
- Visualize metrics
- Real-time monitoring
- Dashboards and charts

**Setup:**
1. Connect Prometheus data source
2. Create or import dashboards
3. Configure panels

**Common Panels:**
- CPU and memory usage
- Request latency
- Error rates
- Pod counts

**Next:** File 05 covers Alertmanager.
