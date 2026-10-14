# Jaeger Setup

## What is Jaeger?

Jaeger is a distributed tracing backend:

```
Applications emit traces
    ↓
Jaeger collects and stores
    ↓
Jaeger UI visualizes traces
```

Created by Uber, popular for Kubernetes.

---

## Install Jaeger

### Option 1: Helm

```bash
# Add Helm repo
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

# Install Jaeger all-in-one
helm install jaeger jaegertracing/jaeger \
  -n tracing --create-namespace \
  --set provisionDataStore.cassandra=false \
  --set elasticsearch.enabled=true \
  --set storage.type=elasticsearch
```

### Option 2: All-in-One Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: tracing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:latest
        env:
        - name: COLLECTOR_ZIPKIN_HTTP_PORT
          value: "9411"
        ports:
        - containerPort: 6831
          protocol: UDP  # Jaeger agent
        - containerPort: 16686
          name: ui       # Web UI
        - containerPort: 14268
          name: collector
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger
  namespace: tracing
spec:
  type: LoadBalancer
  ports:
  - port: 16686
    targetPort: 16686
    name: ui
  - port: 6831
    protocol: UDP
    targetPort: 6831
  - port: 14268
    targetPort: 14268
  selector:
    app: jaeger
```

---

## Access Jaeger UI

```bash
# Get service IP
kubectl get svc -n tracing jaeger

# Port forward
kubectl port-forward -n tracing svc/jaeger 16686:16686

# Open browser
# http://localhost:16686
```

---

## Instrument Application

### Python with OpenTelemetry

```python
# Install packages
# pip install opentelemetry-api opentelemetry-sdk
# pip install opentelemetry-exporter-jaeger
# pip install opentelemetry-instrumentation-flask

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace.export import SimpleSpanProcessor
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from flask import Flask

app = Flask(__name__)

# Configure Jaeger exporter
jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger",
    agent_port=6831,
)

# Set up tracer
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    SimpleSpanProcessor(jaeger_exporter)
)

# Auto-instrument Flask and requests
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

@app.route("/api/checkout", methods=["POST"])
def checkout():
    # Automatic tracing
    # Jaeger captures Flask route, duration, errors
    return {"status": "success"}

if __name__ == "__main__":
    app.run()
```

### Go with OpenTelemetry

```go
package main

import (
	"github.com/opentelemetry/go-otel/exporters/jaeger/jaegergrpc"
	"github.com/opentelemetry/go-otel/sdk/trace"
	"net"
)

func initTracer() error {
	exp, err := jaegergrpc.New(
		context.Background(),
		jaegergrpc.WithEndpoint("jaeger:14250"),
	)
	if err != nil {
		return err
	}

	tp := trace.NewTracerProvider(
		trace.WithBatcher(exp),
	)
	global.SetTracerProvider(tp)
	return nil
}

func main() {
	initTracer()
	tracer := global.Tracer("my-app")
	
	// Use tracer
	ctx, span := tracer.Start(context.Background(), "operation")
	defer span.End()
	
	// Do work
}
```

---

## Real-World: Microservices Tracing

```yaml
---
# Service A (API Gateway)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  template:
    spec:
      containers:
      - name: api
        image: api-gateway:v1
        env:
        - name: JAEGER_AGENT_HOST
          value: jaeger
        - name: JAEGER_AGENT_PORT
          value: "6831"

---
# Service B (Payment Service)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    spec:
      containers:
      - name: payment
        image: payment-service:v1
        env:
        - name: JAEGER_AGENT_HOST
          value: jaeger
        - name: JAEGER_AGENT_PORT
          value: "6831"

---
# Service C (Database)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
spec:
  template:
    spec:
      containers:
      - name: db
        image: postgres:14
        # Database instrumentation via postgres-exporter
```

---

## Generate Traces

```bash
# Make request through instrumented app
curl http://api-gateway/api/checkout \
  -X POST \
  -d '{"user_id": 123, "items": [...]}' \

# Jaeger automatically captures trace

# View in UI
# http://jaeger:16686
# Service: api-gateway
# Operation: POST /api/checkout
# See full trace with latency breakdown
```

---

## Querying Traces

Jaeger UI features:

1. **Find Traces**
   - Service dropdown
   - Operation (endpoint)
   - Min/max duration
   - Status (OK/error)

2. **View Timeline**
   - Each span (horizontal bar)
   - Duration shown
   - Click to expand details

3. **Latency Breakdown**
   - Time in each service
   - Parallel vs. sequential calls

---

## Common Issues

### Application Not Sending Traces

```bash
# Check Jaeger receiver
kubectl logs -n tracing deployment/jaeger | grep receiver

# Verify app environment variables
kubectl describe pod <app-pod> | grep JAEGER

# Check connectivity
kubectl exec <app-pod> -- \
  telnet jaeger 6831
```

### Traces Not Appearing

```
Problem: App instrumented, but no traces in Jaeger

Checklist:
1. Jaeger running? kubectl get pods -n tracing
2. App environment variables set?
3. App making requests? kubectl logs <app-pod>
4. Firewall allowing traffic on 6831?
5. Sampling rate > 0?
```

### High Memory Usage

```yaml
# Jaeger in-memory storage fills up
# Configure persistence

storage:
  type: elasticsearch
  elasticsearch:
    host: elasticsearch
    port: 9200

# Or reduce span retention
# Or export/backup old data
```

---

## Production Setup

```yaml
# High-availability Jaeger
apiVersion: v1
kind: ConfigMap
metadata:
  name: jaeger-config
  namespace: tracing
data:
  sampling.json: |
    {
      "default_strategy": {
        "type": "probabilistic",
        "param": 0.1
      },
      "service_strategies": [
        {
          "service": "payment-service",
          "type": "probabilistic",
          "param": 0.5
        },
        {
          "service": "checkout",
          "type": "const",
          "param": 1
        }
      ]
    }

# Sampling strategy:
# - Default: Trace 10% of requests (probabilistic)
# - Payment: Trace 50% (higher sensitivity)
# - Checkout: Trace 100% (critical service)
```

---

## Jaeger with Kubernetes

Deploy collector and agent separately:

```bash
# Collector (central)
helm install jaeger-collector jaegertracing/jaeger-collector

# Agents (one per node)
helm install jaeger-agent jaegertracing/jaeger-agent \
  --set mode=daemonset
```

Benefits:
- Scales independently
- High availability
- Better performance

---

## Summary

Jaeger setup:

**Install:**
- Helm or manifest
- Runs as single pod (or distributed)
- Listens on port 6831 (agent)

**Instrument:**
- OpenTelemetry library
- Environment variables for Jaeger location
- Auto-instrumentation (Flask, Django, etc.)

**Usage:**
- Make request
- Trace automatically captured
- View in Jaeger UI

**Sampling:**
- Configure sample rate
- Critical services: 100%
- Non-critical: 10-50%

**Next:** File 03 covers OpenTelemetry in detail.
