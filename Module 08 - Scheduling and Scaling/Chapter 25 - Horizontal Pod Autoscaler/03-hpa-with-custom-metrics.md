# HPA with Custom Metrics

## Introduction

CPU and memory are standard metrics. But what if you want to scale based on:

- HTTP requests per second
- Queue length (Kafka, RabbitMQ)
- Database connections
- Custom business metrics

That's where **custom metrics** come in.

---

## Custom Metrics Setup

Custom metrics require:

1. **Metrics collection** (Prometheus, Datadog, etc.)
2. **Adapter** (translates metrics to Kubernetes)
3. **Metric query** (HPA asks for specific metric)

---

## Prometheus + Custom Metrics

### Step 1: Install Prometheus Adapter

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm install prometheus-adapter prometheus-community/prometheus-adapter \
  -n monitoring --create-namespace
```

### Step 2: Configure Metrics

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
    - seriesQuery: 'http_requests_per_second{job="app"}'
      resources:
        overrides:
          pod: {resource: "pod"}
      name:
        matches: "^http_requests_per_second"
        as: "http_requests"
      metricsQuery: 'rate(http_requests_total[1m])'
```

---

## Example: Scale by HTTP Requests

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: request-based-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  
  minReplicas: 2
  maxReplicas: 100
  
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"  # Scale if > 1000 req/s per pod
```

---

## Example: Queue-Based Scaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: queue-consumer-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: message-consumer
  
  minReplicas: 1
  maxReplicas: 50
  
  metrics:
  - type: Pods
    pods:
      metric:
        name: queue_length
      target:
        type: AverageValue
        averageValue: "100"  # 100 messages per consumer
```

If queue has 5000 messages, scales to 50 consumers.

---

## External Metrics

Scale based on external system metrics:

```yaml
metrics:
- type: External
  external:
    metric:
      name: aws_sqs_queue_length
      selector:
        matchLabels:
          queue_name: job_queue
    target:
      type: AverageValue
      averageValue: "30"  # 30 jobs per consumer
```

---

## Real-World Example: Video Processing

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: video-processor-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: video-processor
  
  minReplicas: 2
  maxReplicas: 100
  
  metrics:
  # Scale on queue depth
  - type: Pods
    pods:
      metric:
        name: upload_queue_length
      target:
        type: AverageValue
        averageValue: "5"  # 5 videos per processor
  
  # Also track CPU (fallback)
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
```

Scales aggressively when videos queue up.

---

## Monitoring Custom Metrics

```bash
# List custom metrics
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq

# Query specific metric
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests" | jq

# Check HPA seeing metric
kubectl describe hpa request-based-hpa
Metrics:
  average value: 850m / 1000
```

---

## Challenges with Custom Metrics

1. **Setup complexity** — needs adapter + Prometheus
2. **Latency** — metrics might lag real state
3. **Cardinality** — too many metric variations
4. **Cost** — more infrastructure to maintain

**When to use:**
- Traffic patterns aren't CPU/memory driven
- Application-specific scaling needed
- Business metrics matter (revenue per pod, etc.)

**When NOT to use:**
- Standard CPU/memory sufficient
- Setup overhead not justified

---

## Summary

Custom metrics for specialized scaling:

**Sources:**
- Prometheus (most popular)
- Datadog, CloudWatch, etc.

**Examples:**
- HTTP requests per second
- Queue length
- Database connections
- Business metrics

**Setup:**
- Metrics collection (Prometheus)
- Adapter (translates to Kubernetes)
- HPA queries metric

**Next:** File 04 covers HPA best practices.
