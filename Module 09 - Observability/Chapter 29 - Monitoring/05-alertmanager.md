# Alertmanager

## Introduction

Dashboards are nice, but they require you to watch.

```
You're sleeping
Alert fires: CPU 95%
Dashboard updates, but you don't see it
2 hours later: Pod crashed, everything down

Solution: Alertmanager sends notifications
```

Alertmanager:
- Receives alerts from Prometheus
- Routes to notification channels
- Deduplicates (prevent spam)
- Groups related alerts

---

## Alert Flow

```
Prometheus Rule
  "CPU > 80% for 5 minutes"
        ↓
Alert Fires
  Pod CPU reached 85%
        ↓
Prometheus → Alertmanager
        ↓
Alertmanager evaluates routing
        ↓
Send notification (Slack, email, PagerDuty, etc.)
```

---

## Prometheus Alert Rules

```yaml
# PrometheusRule
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-alerts
  namespace: monitoring
spec:
  groups:
  - name: kubernetes
    interval: 30s
    rules:
    
    # Alert 1: High CPU
    - alert: PodHighCPU
      expr: sum(rate(container_cpu_usage_seconds_total[5m])) by (pod_name) > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod_name }} CPU is high"
        description: "CPU: {{ $value }}"
    
    # Alert 2: OOMKilled
    - alert: PodOOMKilled
      expr: container_last_seen{reason="OOMKilled"} > 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod_name }} was OOMKilled"
    
    # Alert 3: Pod Restarts
    - alert: PodRestartingTooOften
      expr: rate(container_last_seen{reason="ContainerRestarted"}[15m]) > 0.05
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod_name }} restarting too often"
```

---

## Alertmanager Configuration

```yaml
# AlertManager config
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  config.yml: |
    global:
      resolve_timeout: 5m
    
    route:
      # Default receiver
      receiver: default
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 30s        # Wait before sending
      group_interval: 5m     # Wait before updating group
      repeat_interval: 12h   # Re-send after 12 hours
      
      # Sub-routes for different alerts
      routes:
      - match:
          severity: critical
        receiver: pagerduty
        group_wait: 10s       # Faster for critical
      
      - match:
          severity: warning
        receiver: slack
    
    receivers:
    
    # Default: Just email
    - name: default
      email_configs:
      - to: team@company.com
        from: alertmanager@company.com
        smarthost: smtp.company.com:587
        auth_username: alerts@company.com
        auth_password: password
    
    # Critical: PagerDuty
    - name: pagerduty
      pagerduty_configs:
      - service_key: <key>
    
    # Warning: Slack
    - name: slack
      slack_configs:
      - api_url: <webhook-url>
        channel: '#alerts'
        text: '{{ .GroupLabels.alertname }}'
```

---

## Common Receivers

### Slack

```yaml
receivers:
- name: slack
  slack_configs:
  - api_url: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
    channel: '#alerts'
    text: 'Alert: {{ .GroupLabels.alertname }}'
    send_resolved: true  # Notify when resolved
```

### Email

```yaml
receivers:
- name: email
  email_configs:
  - to: team@company.com
    from: alerts@company.com
    smarthost: smtp.company.com:587
    auth_username: alerts@company.com
    auth_password: ${ALERTMANAGER_EMAIL_PASSWORD}
    headers:
      Subject: 'Kubernetes Alert: {{ .GroupLabels.alertname }}'
```

### PagerDuty

```yaml
receivers:
- name: pagerduty
  pagerduty_configs:
  - service_key: ${PAGERDUTY_SERVICE_KEY}
    description: '{{ .GroupLabels.alertname }}'
    details:
      firing: '{{ template "pagerduty.default.instances" .Alerts.Firing }}'
```

---

## Alert Grouping

Alertmanager groups related alerts:

```
5 pods have high CPU
Alertmanager groups them
Single notification: "5 pods high CPU"
Instead of: 5 separate notifications
```

Configuration:

```yaml
route:
  group_by: ['alertname', 'job']
  group_wait: 30s        # Wait 30s for more
  group_interval: 5m     # Check every 5m
  repeat_interval: 12h   # Remind after 12h
```

---

## Suppressing Alerts (Silences)

Maintenance window alert spam prevention:

```bash
# Via API
curl -X POST http://alertmanager:9093/api/v1/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {"name": "alertname", "value": "PodHighCPU", "isRegex": false}
    ],
    "startsAt": "2024-01-15T10:00:00Z",
    "endsAt": "2024-01-15T12:00:00Z",
    "createdBy": "admin",
    "comment": "Maintenance window"
  }'
```

---

## Real-World Alert Strategy

```
Severity Levels:
  Info:     Nice to know
  Warning:  Action recommended
  Critical: Page on-call immediately
```

Example setup:

```yaml
# Info: Just log
- alert: HighMemory
  expr: container_memory_usage_bytes > 800000000
  labels:
    severity: info

# Warning: Slack
- alert: PodRestartingTooOften
  expr: rate(container_restarts_total[1h]) > 0.1
  labels:
    severity: warning

# Critical: PagerDuty
- alert: PodCrashingLoop
  expr: rate(container_restarts_total[5m]) > 0.5
  labels:
    severity: critical

- alert: NodeNotReady
  expr: kube_node_status_condition{condition="Ready", status="true"} == 0
  labels:
    severity: critical
```

---

## Troubleshooting Alerts

### Alerts Not Firing

```bash
# Check PrometheusRule created
kubectl get prometheusrule -n monitoring

# Check Prometheus sees the rule
kubectl port-forward -n monitoring svc/prometheus 9090:9090
# Open http://localhost:9090/rules
# Verify rule listed and active

# Check rule evaluation
kubectl logs -n monitoring prometheus-0 | grep "alert"
```

### Too Many Alerts (Alert Fatigue)

```
Problem: Getting paged constantly
Solution: Adjust thresholds
  - Increase threshold (80% → 85%)
  - Increase duration (5m → 15m)
  - Only critical fires (remove warnings)
```

### Alert Spam

```yaml
# Use grouping to reduce
route:
  group_by: ['alertname', 'cluster', 'job']
  group_wait: 60s        # Wait longer before sending
```

---

## Testing Alerts

```bash
# Generate test alert
kubectl exec -it -n monitoring alertmanager-0 -- \
  curl -X POST http://localhost:9093/api/v1/alerts \
  -H "Content-Type: application/json" \
  -d '[{
    "labels": {
      "alertname": "TestAlert"
    },
    "endsAt": "0001-01-01T00:00:00Z"
  }]'

# Check if notification sent
```

---

## Best Practices

1. **Start Simple**
   - Few alerts first
   - Tune as you learn

2. **Meaningful Alert Names**
   - PodHighCPU vs Alert1?
   - Clear description

3. **Set Appropriate Thresholds**
   - Too low = false positives
   - Too high = missed issues

4. **Test Before Production**
   - Test Slack connection
   - Test PagerDuty integration

5. **Review Alert History**
   - Which alerts frequently fire?
   - Are thresholds tuned properly?

---

## Summary

Alertmanager:

**Purpose:**
- Route alerts to notification channels
- Group related alerts
- Prevent alert spam

**Setup:**
1. Define PrometheusRules
2. Configure Alertmanager receivers
3. Test notifications

**Common Receivers:**
- Slack (team)
- Email (async)
- PagerDuty (on-call)

**Best Practices:**
- Start simple
- Tune thresholds carefully
- Clear descriptions

**Module 9 Chapter 29 Complete - 5 Files!**

You now understand monitoring:
- File 01: Overview (metrics, logs, traces)
- File 02: Metrics Server (resource collection)
- File 03: Prometheus (long-term storage)
- File 04: Grafana (visualization)
- File 05: Alertmanager (notifications)

**Next:** Chapter 30 covers distributed tracing.
