# 02 — The Three Signals: Traces, Metrics & Logs

**Visual guide:** `otel-signals.html`
**Covers:** What each signal type captures, when to use each one, how they complement each other, and how they connect via trace context.

---

## Overview

OTEL defines three signal types. They are not interchangeable — each answers a different question about your system. Production observability requires all three.

| Signal | Core question | Storage cost | Query speed |
|--------|--------------|--------------|-------------|
| Traces | Why was this request slow or broken? | High | Slow (per-request) |
| Metrics | Is the system healthy right now? | Low | Fast (aggregated) |
| Logs | What exactly happened during this event? | Medium–High | Medium |

---

## Traces

### What a trace is

A trace represents the full lifecycle of a single request as it moves through your system. It is composed of **spans** — one span per operation. Spans are nested: a root span covers the entire request, and child spans represent individual steps.

```
POST /checkout  [root span — 412ms]
 └─ createOrder()  [child — 370ms]
     ├─ INSERT INTO orders  [db span — 122ms]
     └─ charge()  [child — 220ms]
         └─ POST api.stripe.com/charge  [external — 204ms ⚠ ERROR]
```

### Span anatomy

Every span carries:

```yaml
trace_id:       abc123def456...   # shared across all spans in the trace
span_id:        f9a2b1c3...       # unique to this span
parent_span_id: d3e4c5f6...       # null for the root span
name:           POST /checkout
start_time:     2024-03-01T10:00:00.000Z
end_time:       2024-03-01T10:00:00.412Z
status:         ERROR

attributes:                        # key-value metadata
  http.method:      POST
  http.status_code: 402
  service.name:     payment-service
  service.version:  2.4.1
```

### Auto vs. manual instrumentation

**Auto-instrumentation** (no code changes required) covers:
- Inbound/outbound HTTP calls
- Database queries (PostgreSQL, MySQL, Redis, MongoDB)
- Message queue operations (Kafka, RabbitMQ, SQS)
- gRPC calls
- Common web frameworks

**Manual spans** should be added for business logic that matters:
- Complex calculations (`calculateTax()`, `applyPromoCode()`)
- External integrations not covered by auto-instrumentation
- Any operation that is slow, can fail, or is important to business reporting

```python
# Example: manual span in Python
with tracer.start_as_current_span("validatePromoCode") as span:
    span.set_attribute("promo.code", code)
    result = validate(code)
    if not result.valid:
        span.set_status(StatusCode.ERROR, result.reason)
```

---

## Metrics

### What metrics are

Numerical measurements collected at regular intervals. Metrics are aggregated — they summarise many events into single numbers — which makes them cheap to store and fast to query. They are the foundation of dashboards and alerting.

### The three metric types

**Counter**
- Monotonically increasing. Only goes up. Resets to zero on process restart.
- Use for: total counts and derived rates
- Examples: `http.server.request.count`, `errors.total`, `messages.published`

**Gauge**
- Current value at a point in time. Can go up or down.
- Use for: current state of a resource
- Examples: `db.pool.connections.active`, `queue.depth`, `memory.heap.used`

**Histogram**
- Samples observations into configurable buckets. Enables percentile calculations.
- Use for: latency and size distributions
- Examples: `http.server.request.duration`, `db.query.duration`
- Buckets define the granularity: `[5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s, 2.5s, 5s]`

### Attributes (dimensions)

Attributes turn a single metric into a multi-dimensional dataset. Without attributes, `http.request.duration` is one number. With attributes, you can slice it by service, endpoint, method, status code, and region.

```
http.request.duration{service="order-service", method="POST", status="200", region="eu-west"}
http.request.duration{service="order-service", method="POST", status="500", region="eu-west"}
```

### ⚠ Cardinality warning

High-cardinality attributes (user IDs, request IDs, session tokens) create a unique time series per value. With millions of users, this means millions of time series — which will overload most metrics backends.

**Rule:** Use metrics for low-cardinality dimensions only. For high-cardinality data (per-user, per-request), use traces.

---

## Logs

### What logs are

Timestamped, structured records of discrete events. They provide the richest level of detail about what happened — exact error messages, stack traces, and business-level context.

### What OTEL adds to logging

You almost certainly already have logs. OTEL's contribution is attaching **trace context** — `trace_id` and `span_id` — to each log record. This creates a link between the log entry and the trace that produced it.

**Without OTEL context:**
```
2024-03-01 10:00:00.320 [ERROR] Payment charge failed: card_declined
2024-03-01 10:00:00.321 [ERROR] Returning 402 to client
```
No way to know which user, which request, or which trace produced this.

**With OTEL context:**
```json
{
  "timestamp": "2024-03-01T10:00:00.320Z",
  "severity": "ERROR",
  "message": "Payment charge failed: card_declined",
  "service.name": "payment-service",
  "trace_id": "abc123def456...",
  "span_id": "f9a2b1c3...",
  "user_id": "usr_8821",
  "order_id": "ord_4492"
}
```
The `trace_id` links directly to the full request waterfall in your tracing backend.

### Severity levels

| Level | When to use |
|-------|------------|
| `DEBUG` | Detailed internals. Dev/staging only. Never in production. |
| `INFO` | Normal operational events. |
| `WARN` | Unexpected situation, but handled. Warrants review. |
| `ERROR` | Something failed that needs attention. |
| `FATAL` | Process is about to exit. |

Use severity consistently across all services so filtering across a distributed system is reliable.

---

## The Standard Investigation Pattern

This is how the three signals work together in practice:

```
① Metric alert fires
   error.rate > 5% triggers PagerDuty
   → You know something is wrong, but not what

② Traces reveal the cause
   Filter traces: status=ERROR, time window=10:00–10:05
   → Waterfall shows the Stripe span failing at 200ms for 12% of requests

③ Logs give the detail
   Click trace_id → jump to correlated logs
   → "card_declined: insufficient_funds"
```

The `trace_id` is the connective tissue. It is present in the trace and in the log record, allowing any observability backend to navigate between them.

This workflow — **alert on metrics → investigate with traces → read detail in logs** — is the standard OTEL investigation pattern. It only works when:
1. All three signals are being collected
2. Logs are enriched with trace context
3. Context propagation is working across service boundaries (see Section 05)

---

## What's Next

- **[03 — The Collector](otel-collector.md)** — Pipeline config, processors, and exporter routing
- **[04 — Sampling](otel-sampling.md)** — Controlling trace volume without losing important signals
- **[05 — Distributed Tracing](otel-distributed.md)** — How trace context crosses service boundaries

---

[← Back to Guide Index](otel-overview.md)
