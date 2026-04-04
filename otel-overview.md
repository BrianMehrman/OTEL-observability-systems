# 01 — OTEL Overview: Architecture & Key Concepts

**Visual guide:** `otel-overview.html`
**Covers:** What OTEL is, the three signal types, the core pipeline, small vs. large system architecture, and the six decisions that shape your setup.

---

## What is OpenTelemetry?

OpenTelemetry (OTEL) is an open-source observability framework and a CNCF project. It provides:

- **Standardised SDKs** for instrumenting your application code (available in most major languages)
- **A wire protocol** (OTLP) for transmitting telemetry data
- **A Collector** — a standalone process that receives, processes, and exports telemetry
- **A vendor-neutral standard** — you instrument once, send to any backend

The critical thing OTEL does *not* provide is a storage or visualisation layer. That's the job of a backend (Jaeger, Prometheus, Grafana, Datadog, etc.).

---

## The Three Signal Types

### Traces
A trace represents the lifecycle of a single request through your system. It is made up of **spans** — one per operation (HTTP call, DB query, function, etc.). Each span records:
- Start time and duration
- Status (ok / error)
- Attributes (key-value metadata)
- A link to its parent span

Traces answer: *where did this request spend its time, and where did it fail?*

### Metrics
Numerical measurements collected over time. Three common types:
- **Counter** — monotonically increasing (e.g. total requests)
- **Gauge** — current value that can go up or down (e.g. active connections)
- **Histogram** — distribution of values (e.g. request latency percentiles)

Metrics answer: *is the system healthy right now, and how is it trending?*

### Logs
Timestamped, structured text records. OTEL's contribution to logging is adding **trace context** — attaching a `trace_id` and `span_id` to each log record so logs can be correlated with the trace that produced them.

Logs answer: *what exactly happened during this operation?*

---

## The Core Pipeline

```
[ Your Application ]
        │  OTLP (gRPC or HTTP)
        ▼
[ OTEL Collector ]
   ├── Receivers   (accept incoming telemetry)
   ├── Processors  (batch, filter, sample, enrich)
   └── Exporters   (send to backends)
        │
        ▼
[ Observability Backend ]
   e.g. Jaeger, Prometheus, Grafana Cloud, Datadog
```

The Collector is the key architectural component. It decouples your application from any specific backend. You can:
- Add, remove, or swap backends without touching application code
- Apply sampling, filtering, and PII scrubbing in one place
- Fan out to multiple backends from a single pipeline

---

## Small App vs. Large Distributed System

### Small Application

**Characteristics:** Single service, low request volume, small team.

```
[ Service ]
    │ OTLP
    ▼
[ Collector — single instance, same host or sidecar ]
    │
    ▼
[ Hosted Backend — e.g. Grafana Cloud ]
```

**Recommendations:**
- Use **auto-instrumentation** — covers HTTP, DB, and common frameworks with zero code changes
- The Collector can be a sidecar container or run on the same host
- 100% sampling is fine at low volume
- A single hosted backend keeps operational overhead low

### Large Distributed System

**Characteristics:** Many services, many teams, high request volume.

```
[ Service A ]  [ Service B ]  [ Service C ]
      │               │               │
      └───────────────┴───────────────┘
                      │ OTLP (to local agent)
                      ▼
         [ Agent Collectors — per node/pod ]
           DaemonSet in Kubernetes
           Handles: local buffering, initial filtering
                      │
                      │ Forward to gateway
                      ▼
         [ Gateway Collectors — centralised tier ]
           Handles: tail-based sampling, PII scrubbing,
                    routing logic, rate limiting
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       [Jaeger]  [Prometheus]  [Loki]
       (traces)   (metrics)    (logs)
```

**Recommendations:**
- Two-tier Collector architecture (agents + gateways)
- Tail-based sampling at the gateway tier — always keep errors and slow requests
- Context propagation (W3C TraceContext headers) must be enforced across all services
- Fan out to separate backends per signal type if different teams own different tooling

---

## Inside the Collector

Every signal passes through a three-stage pipeline defined in `collector-config.yaml`:

| Stage | Purpose | Key components |
|-------|---------|----------------|
| **Receivers** | Accept incoming telemetry | `otlp`, `prometheus`, `jaeger`, `hostmetrics` |
| **Processors** | Transform and control flow | `batch`, `memory_limiter`, `filter`, `attributes`, `tail_sampling` |
| **Exporters** | Send to backends | `otlp`, `prometheus`, `datadog`, `logging` |

**Always include:**
- `batch` processor — groups spans before sending; prevents flooding backends
- `memory_limiter` processor — prevents the Collector OOMing during traffic spikes

---

## Sampling

### Head-based sampling
Decision is made at the *start* of the request. Simple, low overhead. Cannot guarantee capturing failed requests because the decision happens before the outcome is known.

**Use when:** Low to medium volume, or when simplicity matters more than guaranteed error capture.

### Tail-based sampling
Decision is made *after the trace completes*, in the Collector. Rules can be defined:
- Always keep traces with errors
- Always keep traces over a latency threshold
- Keep N% of healthy traces for baseline visibility

**Use when:** High volume, or when you need guaranteed capture of errors and slow requests.

---

## Six Decisions That Shape Your Setup

| # | Decision | Small App | Large System |
|---|----------|-----------|--------------|
| 1 | Instrumentation | Auto-instrumentation | Auto + manual spans for key business logic |
| 2 | Collector | Optional (or sidecar) | Required, two-tier |
| 3 | Sampling | Head-based, 100% | Tail-based with error rules |
| 4 | Context propagation | Handled by SDK | Must be enforced across all services |
| 5 | Backend | Single hosted service | Multiple, per signal or per team |
| 6 | Collector scale | Single instance | Agent + gateway tiers, horizontally scaled |

---

## What's Next

- **[02 — The Three Signals]** — When to use traces vs. metrics vs. logs, and how to instrument each
- **[03 — The Collector]** — Full config YAML walkthrough, processor recipes
- **[04 — Sampling]** — Designing sampling rules for real workloads
- **[05 — Distributed Tracing]** — Context propagation, service graphs, and trace correlation
