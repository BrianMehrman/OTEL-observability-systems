# 05 — Distributed Tracing: Context Propagation

**Visual guide:** `otel-distributed.html`
**Covers:** What context propagation is, propagation formats (W3C, B3, vendor), what breaks trace continuity, async patterns (queues, background jobs), and a deployment checklist.

---

## What Context Propagation Is

A trace spans multiple services. Each service must pass a small packet of context — the `trace_id` and the current `span_id` — to every downstream call it makes. Without this, each service records isolated spans with no connection to the others. The trace is broken.

```
Without propagation:          With propagation:
api-gateway  trace: aaa111    api-gateway  trace: abc123
order-svc    trace: bbb222    order-svc    trace: abc123  ← same
payment-svc  trace: ccc333    payment-svc  trace: abc123  ← same
```

Context propagation is what turns a collection of disconnected spans into a single trace waterfall.

---

## How It Works

1. The root service generates a `trace_id` (32 hex chars) and a `span_id` (16 hex chars)
2. Before making any outgoing call, the SDK injects these into the request headers
3. The downstream service reads the headers and creates a child span linked to the incoming `span_id`
4. That service then injects the same `trace_id` and its own `span_id` into its outgoing calls
5. This repeats for every service in the chain

Every service in the chain records spans with the same `trace_id`, allowing them to be assembled into a waterfall view.

---

## Propagation Formats

### W3C TraceContext (recommended)

The IETF standard. Two headers:

```
traceparent: 00-{32-char-traceId}-{16-char-spanId}-{flags}
tracestate:  vendor1=value1,vendor2=value2
```

- `00` — format version
- `trace_id` — 32 hex characters, same across all spans in the trace
- `span_id` — 16 hex characters, unique to the current span
- `flags` — `01` = sampled, `00` = not sampled

Supported by all OTEL SDKs and most modern observability platforms.

### B3 (Zipkin — legacy)

Used in older microservice stacks. OTEL supports both multi-header and single-header B3:

```
# Multi-header
X-B3-TraceId:      abc123def456...
X-B3-SpanId:       f9a2b1c3...
X-B3-ParentSpanId: d3e4c5f6...
X-B3-Sampled:      1

# Single-header
b3: {traceId}-{spanId}-{sampled}-{parentSpanId}
```

### Configuring multiple propagators

During a migration, configure OTEL to read and write multiple formats simultaneously:

```bash
OTEL_PROPAGATORS=tracecontext,baggage,b3multi
```

---

## What Breaks Trace Continuity

### 1. Raw HTTP clients bypassing the SDK

```python
# WRONG — context not propagated
response = requests.get(url)

# CORRECT — manually inject headers
from opentelemetry import propagate

headers = {}
propagate.inject(headers)
response = requests.get(url, headers=headers)
```

### 2. Third-party services

External services (Stripe, Twilio, SendGrid) won't continue the trace. The trace ends at your outbound call. Model these as leaf spans:

```python
with tracer.start_as_current_span("stripe.charge") as span:
    span.set_attribute("stripe.amount", amount)
    span.set_attribute("stripe.currency", "usd")
    result = stripe.charge(amount=amount)
    if result.error:
        span.record_exception(result.error)
        span.set_status(StatusCode.ERROR, result.error.message)
```

### 3. Thread and async boundaries

Trace context is thread-local. Spawning threads or async tasks without propagating context creates a new root span:

```python
from opentelemetry import context

# Capture context before spawning
ctx = context.get_current()

def worker_fn():
    # Restore context inside the thread
    token = context.attach(ctx)
    try:
        with tracer.start_as_current_span("background-work"):
            do_work()
    finally:
        context.detach(token)

thread = threading.Thread(target=worker_fn)
thread.start()
```

### 4. Mixed propagation formats

If one service emits B3 headers and the downstream only reads W3C headers, the trace breaks at that boundary. Enable multi-format propagation on all services until migration is complete.

---

## Async Patterns

### Message Queues (Kafka, SQS, RabbitMQ)

The producer and consumer are decoupled in time. Context must be serialised into message headers and deserialised on consumption:

```python
# Producer
headers = {}
propagate.inject(headers)
producer.send(topic, value=payload, headers=list(headers.items()))

# Consumer
ctx = propagate.extract(dict(msg.headers))
with tracer.start_as_current_span(
    "process-message",
    context=ctx,
    kind=SpanKind.CONSUMER
) as span:
    span.set_attribute("messaging.system", "kafka")
    span.set_attribute("messaging.destination", topic)
    process(msg.value)
```

The producer trace and consumer trace appear as separate but linked traces. Most modern trace backends support span links to visualise this relationship.

### Background Jobs and Scheduled Tasks

Jobs enqueued by a request can carry trace context in the job payload:

```python
# Enqueue — store context as job metadata
headers = {}
propagate.inject(headers)
job_queue.enqueue(
    "send-invoice",
    args=[order_id],
    meta={"trace_context": headers}
)

# Job handler — restore context
ctx = propagate.extract(job.meta["trace_context"])
with tracer.start_as_current_span("send-invoice", context=ctx) as span:
    send_invoice(order_id)
```

Purely scheduled jobs (cron-style, no triggering request) start new root spans. This is expected.

---

## Deployment Checklist

| # | Check | Config |
|---|-------|--------|
| 1 | OTEL SDK installed on all services | Language-specific package |
| 2 | Auto-instrumentation enabled | `OTEL_PYTHON_CONFIGURATOR=sdk` or equivalent |
| 3 | W3C TraceContext as primary propagator | `OTEL_PROPAGATORS=tracecontext,baggage` |
| 4 | B3 added during migration from Zipkin/Jaeger | `OTEL_PROPAGATORS=tracecontext,baggage,b3multi` |
| 5 | `service.name` set uniquely per service | `OTEL_SERVICE_NAME=order-service` |
| 6 | Message queue producers inject context | `propagate.inject(headers)` before send |
| 7 | Message queue consumers extract context | `propagate.extract(msg.headers)` before processing |
| 8 | Thread/async boundaries propagate context | Manual `context.attach()` / `context.detach()` |
| 9 | External service calls wrapped in manual spans | Model leaf spans with result attributes |

---

## What's Next

The five core sections are complete. The next phase adds:
- **06 — Ecosystem** — Backend tools, SaaS platforms, and OSS stack recommendations per scale tier
- **07 — Language SDKs** — Setup guides for Python, Node.js, Go, Java
- **08 — Kubernetes** — OTEL Operator, DaemonSet deployment, cert-manager integration
