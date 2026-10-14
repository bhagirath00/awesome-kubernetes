# OpenTelemetry

## What is OpenTelemetry?

OpenTelemetry (OTEL) is the standard for observability:

```
Before OTEL: Each vendor had own format
  Jaeger format, DataDog format, NewRelic format
  Vendor lock-in, costly to switch

OpenTelemetry: Standard format
  Works with any backend
  Easy to change vendors
```

---

## OTEL Architecture

```
Application Code
    ↓ Uses OTEL APIs
OTEL SDK
    ├─ Tracer (for traces)
    ├─ Meter (for metrics)
    ├─ Logger (for logs)
    ↓
Exporters
    ├─ Jaeger exporter
    ├─ Prometheus exporter
    ├─ OTLP exporter
    ↓
Backend (Jaeger, Prometheus, Datadog, etc.)
```

---

## OTEL Components

### 1. Traces (Request Journey)

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("checkout") as span:
    span.set_attribute("user_id", 123)
    span.set_attribute("amount", 99.99)
    
    # Do work
    result = process_payment()
    span.set_attribute("status", "success")
```

### 2. Metrics (Numbers)

```python
from opentelemetry import metrics

meter = metrics.get_meter(__name__)

# Counter (increases)
counter = meter.create_counter("requests_total")
counter.add(1)

# Gauge (current value)
gauge = meter.create_gauge("memory_usage")
gauge.record(256)

# Histogram (distribution)
histogram = meter.create_histogram("request_duration_ms")
histogram.record(150)
```

### 3. Logs (Events)

```python
from opentelemetry import logs

logger_provider = logs.get_logger_provider()
logger = logger_provider.get_logger(__name__)

logger.emit(
    name="user_login",
    body="User logged in",
    attributes={"user_id": 123}
)
```

---

## Instrumentation Types

### Manual Instrumentation

```python
# You add tracing code
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def checkout(user_id):
    with tracer.start_as_current_span("checkout") as span:
        span.set_attribute("user_id", user_id)
        # Your code
```

**Pros:** Full control
**Cons:** Requires code changes

### Auto-Instrumentation

```python
# OTEL automatically instruments
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

# Flask routes automatically traced
# Outgoing HTTP requests automatically traced
```

**Pros:** No code changes
**Cons:** Limited to supported libraries

---

## Practical Example: E-Commerce

```python
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
import requests
from flask import Flask

# Setup
app = Flask(__name__)
jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger",
    agent_port=6831,
)
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

tracer = trace.get_tracer(__name__)
meter = metrics.get_meter(__name__)

# Metrics
checkout_counter = meter.create_counter(
    "checkout_requests_total",
    unit="1",
    description="Total checkout requests"
)

@app.route("/checkout", methods=["POST"])
def checkout():
    checkout_counter.add(1)
    
    with tracer.start_as_current_span("checkout_process") as span:
        order_id = "order_123"
        span.set_attribute("order_id", order_id)
        
        # Call payment service
        with tracer.start_as_current_span("call_payment_service"):
            response = requests.post(
                "http://payment-service/process",
                json={"order_id": order_id}
            )
            span.set_attribute("payment_status", response.status_code)
        
        # Call inventory service
        with tracer.start_as_current_span("update_inventory"):
            requests.post(
                "http://inventory-service/deduct",
                json={"order_id": order_id}
            )
        
        return {"order_id": order_id, "status": "success"}

if __name__ == "__main__":
    app.run()
```

---

## Exporters

### OTLP (OpenTelemetry Protocol)

Modern standard, works everywhere:

```python
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

exporter = OTLPSpanExporter(
    endpoint="jaeger-collector:4317"
)
```

### Jaeger Direct

```python
from opentelemetry.exporter.jaeger.thrift import JaegerExporter

exporter = JaegerExporter(
    agent_host_name="jaeger",
    agent_port=6831,
)
```

### Prometheus (for metrics)

```python
from opentelemetry.exporter.prometheus import PrometheusMetricReader
from opentelemetry.sdk.metrics import MeterProvider

reader = PrometheusMetricReader()
provider = MeterProvider(metric_readers=[reader])
```

---

## Sampling Strategies

Control what gets traced:

### Probability Sampling

```python
from opentelemetry.sdk.trace.sampling import ProbabilitySampler

# Sample 10% of traces
sampler = ProbabilitySampler(rate=0.1)
provider = TracerProvider(sampler=sampler)
```

### Parent-Based Sampling

```python
from opentelemetry.sdk.trace.sampling import ParentBased, TraceIdRatioBased

# Respect parent's decision
sampler = ParentBased(
    root=TraceIdRatioBased(0.1)
)
```

### Custom Sampling

```python
class ErrorSampler(Sampler):
    def should_sample(self, trace_id, parent_ctx, name, attributes):
        # Sample 100% if marked as error
        if attributes.get("error", False):
            return RECORD_ALL
        # Otherwise sample 1%
        return TraceIdRatioBased(0.01)
```

---

## Testing OTEL

```bash
# Verify traces are sent
kubectl logs -n tracing deployment/jaeger | grep "span received"

# Generate traffic
curl -X POST http://app-service/api/checkout

# View in Jaeger
kubectl port-forward -n tracing svc/jaeger 16686:16686
# Open http://localhost:16686
# Select service, look for traces
```

---

## Common Mistakes

### Mistake 1: No Sampling

```python
# WRONG — traces every request
sampler = AlwaysOnSampler()
# In production: 1000 requests/sec = huge volume

# RIGHT
sampler = ProbabilitySampler(0.1)  # 10%
```

### Mistake 2: Losing Trace Context

```python
# WRONG — context not propagated
async def call_downstream():
    async with aiohttp.ClientSession() as session:
        response = await session.post("http://service/api")
        # Headers NOT injected with trace context

# RIGHT
from opentelemetry.propagate import inject

headers = {}
inject(headers)
async with aiohttp.ClientSession() as session:
    response = await session.post("http://service/api", 
                                 headers=headers)
```

### Mistake 3: Too Much Data

```python
# WRONG — set_attribute for every variable
span.set_attribute("temp_var_1", value1)
span.set_attribute("temp_var_2", value2)
span.set_attribute("temp_var_3", value3)
# Creates massive spans

# RIGHT
span.set_attribute("user_id", 123)
span.set_attribute("order_total", 99.99)
# Only important attributes
```

---

## Best Practices

1. **Use Auto-Instrumentation When Possible**
   - Less code to maintain
   - Consistent across libraries

2. **Sample Appropriately**
   - Critical paths: 100%
   - Normal operations: 10-50%
   - High-volume: 1-5%

3. **Meaningful Attributes**
   - user_id, order_id, status
   - Not: var1, temp, x

4. **Propagate Context**
   - Always pass trace headers
   - Maintain causal chain

5. **Monitor OTEL Itself**
   - Check exporter health
   - Monitor sampled traces

---

## Summary

OpenTelemetry:

**What it is:**
- Standard for observability
- Works with any backend
- Three pillars: traces, metrics, logs

**Components:**
- APIs (manual instrumentation)
- SDKs (implementation)
- Exporters (send to backend)
- Propagators (pass context)

**Instrumentation:**
- Auto-instrumentation (easy)
- Manual instrumentation (flexible)
- Both together (best)

**Sampling:**
- Probabilistic
- Parent-based
- Custom logic

**Module 9 Chapter 30 Complete - 3 Files!**

You now understand distributed tracing:
- File 01: Overview (spans, traces, journey)
- File 02: Jaeger setup (backend installation)
- File 03: OpenTelemetry (standard instrumentation)

**Module 9 Complete! (13 files total)**

Summary of Module 9 - Observability:
- Chapter 28 - Logging (5 files): Architecture, node logs, clustering, Fluentd, ELK
- Chapter 29 - Monitoring (5 files): Overview, Metrics Server, Prometheus, Grafana, Alerts
- Chapter 30 - Tracing (3 files): Overview, Jaeger, OpenTelemetry

**Next:** Module 10 - Package Management (Helm)
