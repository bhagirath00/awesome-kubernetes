# Distributed Tracing Overview

## Introduction

You have metrics and logs. But still can't answer:

```
User makes one request
Request flows through:
  API Gateway → Auth Service → Payment Service → Database

User says: "Payment failed"
You check logs:
  Auth: Success
  Payment: "Connection timeout"
  Database: Waiting for response

But which service is actually slow?
Which service made the timeout?
Where is time being spent?

Answer: Distributed tracing
```

---

## Three Pillars of Observability

```
Metrics:      "What?" (numbers)
Logs:         "Why?" (events/messages)
Traces:       "How?" (request journey)
```

**Traces** show the complete request path.

---

## What is a Trace?

A trace is a request's journey through the system:

```
User Request
    ↓
API Gateway (5ms)
    ↓
Authentication Service (20ms)
    ↓
Business Logic (150ms)
    ↓
Payment Service (100ms)
    ↓
Database (75ms)
    ↓
Response (350ms total)

Trace captures:
- Each service involved
- Time spent in each
- Errors at each step
- Service-to-service calls
```

---

## Span

A span represents one operation:

```
Span: "GET /api/users"
├─ Service: api-gateway
├─ Duration: 350ms
├─ Status: OK
├─ Tags: {user_id: 123}
└─ Child Spans:
    ├─ "Call auth service" (20ms)
    ├─ "Call business logic" (150ms)
    └─ "Call payment" (100ms)
```

Each span knows:
- Operation name
- Start time
- Duration
- Parent span (who called me?)
- Child spans (who did I call?)

---

## Real-World Scenario: Slow Checkout

```
Merchant: "Checkout is slow"
You investigate

Check metrics:
  API Gateway: 5ms latency ✓
  Auth Service: 20ms latency ✓
  Payment Service: 500ms latency ✗

Check logs:
  Auth: "User verified"
  Payment: "Processing payment"

But why is Payment slow?
Trace shows:
  Span: "Payment Service"
    - Database query: 450ms
    - External API: 40ms
    - Processing: 10ms

Root cause: Slow database query
Solution: Add index to database
```

---

## Tracing vs. Logging

```
Logging (textual, separate entries):
  10:30:00 Auth: User login
  10:30:02 Payment: Processing
  10:30:05 Database: Query executed

Tracing (structured, causal relationship):
  Request ID: abc123
  10:30:00 Auth → 10:30:02 Payment → 10:30:05 Database
  (shows the flow)
```

Tracing shows *relationships*.

---

## Instrumentation

To trace, applications must emit trace data:

```python
# Python example
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def process_payment(payment_id):
    with tracer.start_as_current_span("process_payment") as span:
        span.set_attribute("payment_id", payment_id)
        
        # Do work
        result = check_fraud(payment_id)
        span.set_attribute("fraud_check", "passed")
        
        result = call_bank(payment_id)
        span.set_attribute("bank_status", result)
        
        return result

# Automatically sends trace to Jaeger
```

---

## Trace Propagation

Request flows through multiple services. Each must:
1. Receive trace context from caller
2. Create span for its work
3. Pass trace context to downstream services

```
Client Request
  Headers: {trace_id: abc123, span_id: span1}
    ↓
Service A
  Creates span under trace abc123
  Calls Service B with headers
  Headers: {trace_id: abc123, span_id: span2, parent_id: span1}
    ↓
Service B
  Creates span under trace abc123
  Knows parent is span1 (Service A)
  Calls Database
    ↓
Database
  Span under same trace abc123
```

All connected by trace_id.

---

## OpenTelemetry Standard

OpenTelemetry is the standard for tracing:

```
OpenTelemetry (OTEL)
  ├─ API: How to instrument code
  ├─ SDK: How to collect traces
  ├─ Exporters: How to send (Jaeger, Zipkin, etc.)
  └─ Semantic Conventions: Naming standards
```

Works with:
- Jaeger (popular)
- Zipkin (open-source)
- DataDog
- NewRelic
- AWS X-Ray

---

## Common Mistake: No Trace Propagation

```python
# WRONG — loses trace context
def forward_request(data):
    response = requests.post(f"http://payment-service/pay", 
                            data=data)
    # Headers NOT passed
    # Trace chain broken

# RIGHT — propagates context
from opentelemetry.propagate import inject

headers = {}
inject(headers)  # Add trace context headers

response = requests.post(f"http://payment-service/pay",
                        data=data,
                        headers=headers)
# Trace chain continues
```

---

## Sampling

Tracing high-volume services creates lots of data:

```
1000 requests/second
All requests traced
= millions of spans per minute
= massive storage cost

Solution: Sample
  Trace 1% of requests
  Or: Trace errors and slow requests
```

---

## Real-World Example: Microservices

```
E-commerce checkout trace:

User makes purchase
  Request ID: xyz789
  Trace starts

API Gateway
  Creates span "process_checkout"
  Calls auth service

Auth Service
  Span "verify_user" (15ms)
  Creates JWT token
  Returns to API

Inventory Service
  Span "check_stock" (45ms)
  Queries database
  Returns availability

Payment Service
  Span "process_payment" (250ms)
    ├─ Span "verify_payment" (30ms)
    ├─ Span "call_bank_api" (200ms)
    └─ Span "store_receipt" (20ms)

Total trace: 350ms
Breakdown: Clear which service is slow
```

---

## Comparison: Jaeger vs. Zipkin

| Feature | Jaeger | Zipkin |
|---------|--------|--------|
| Backend | Go | Java |
| UI | Modern | Basic |
| Query | Advanced | Simple |
| Sampling | Yes | Yes |
| Storage | Multiple | ES, Cassandra |
| Performance | Fast | Slower |

Jaeger more popular for Kubernetes.

---

## Summary

Distributed tracing:

**What it is:**
- Request journey through services
- Shows latency at each step
- Identifies bottlenecks

**Why it matters:**
- Metrics say "API is slow"
- Tracing says "Slow at payment service, database query"

**Components:**
- Instrumentation (code)
- Trace propagation (headers)
- Trace backend (Jaeger, Zipkin)
- UI (visualize)

**Standard:**
- OpenTelemetry (OTEL)
- Works with any backend

**Next:** File 02 covers Jaeger setup.
