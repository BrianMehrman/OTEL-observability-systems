# 03 — The OTEL Collector: Pipeline, Config & Scale

**Visual guide:** `otel-collector.html`
**Covers:** How the Collector pipeline works, three annotated config examples (simple / agent / gateway), critical processors, and when to use a single vs. two-tier topology.

---

## What the Collector Does

The Collector is a standalone binary that sits between your application and your observability backend. It is not a requirement for getting started, but it becomes essential once you need any of:

- Sampling (dropping a percentage of healthy traces)
- PII scrubbing (removing sensitive fields before storage)
- Fan-out routing (sending signals to multiple backends)
- Enrichment (attaching Kubernetes metadata, environment tags)
- Buffering (absorbing traffic spikes without dropping to the backend)

The Collector is vendor-neutral. Switching backends means changing one exporter block in config — not touching application code.

---

## The Three-Stage Pipeline

Every signal (traces, metrics, logs) flows through the same three stages:

```
Receivers  →  Processors  →  Exporters
```

You declare independent pipelines per signal type in the `service.pipelines` section. Each pipeline can have different processors and exporters.

### Stage 1 — Receivers

Receivers accept incoming telemetry. The most common are:

| Receiver | Purpose |
|----------|---------|
| `otlp/grpc` | Receives OTLP over gRPC (port 4317) — preferred |
| `otlp/http` | Receives OTLP over HTTP (port 4318) |
| `hostmetrics` | Scrapes CPU, memory, disk, network from the host |
| `prometheus` | Scrapes Prometheus `/metrics` endpoints |
| `filelog` | Tails log files from disk |
| `k8s_events` | Receives Kubernetes cluster events |

### Stage 2 — Processors

Processors transform or filter data in-flight. **Order matters** — processors run sequentially in the order listed.

**Mandatory ordering rule:**
```
memory_limiter  →  [your processors]  →  batch
```

`memory_limiter` must always be first. `batch` must always be last.

Key processors:

| Processor | Purpose | When to use |
|-----------|---------|-------------|
| `memory_limiter` | Drops data before OOMing the process | Always — no exceptions |
| `batch` | Groups items before export | Always — no exceptions |
| `resourcedetection` | Auto-attaches host/cloud/container metadata | Almost always |
| `k8sattributes` | Attaches pod/namespace/deployment labels | Any Kubernetes setup |
| `attributes` | Add, update, delete, or hash span/metric attributes | PII scrubbing, env tagging |
| `filter` | Drop entire spans/metrics/logs by condition | Removing health check noise |
| `tail_sampling` | Sample traces after they complete | Gateway-tier only |
| `transform` | Rename or recompute fields using OTTL | Advanced reshaping |

### Stage 3 — Exporters

Exporters send data to backends. Multiple exporters of the same type use `type/name` syntax:

```yaml
exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
  otlp/loki:
    endpoint: loki:3100
```

---

## Configuration Examples

### Simple (small app)

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:            # always first
    check_interval: 1s
    limit_mib: 400
    spike_limit_mib: 100

  resourcedetection:
    detectors: [env, system, docker]

  batch:                     # always last
    send_batch_size: 1000
    timeout: 5s

exporters:
  otlp:
    endpoint: https://your-backend:4317
    headers:
      Authorization: "Bearer ${env:BACKEND_API_KEY}"

service:
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, resourcedetection, batch]
      exporters:  [otlp]
    metrics:
      receivers:  [otlp]
      processors: [memory_limiter, resourcedetection, batch]
      exporters:  [otlp]
    logs:
      receivers:  [otlp]
      processors: [memory_limiter, resourcedetection, batch]
      exporters:  [otlp]
```

### Agent (per node / DaemonSet)

The agent tier runs on every node in your cluster. Its job is lightweight: accept traffic locally, attach Kubernetes metadata, forward to the gateway.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
  hostmetrics:
    collection_interval: 30s
    scrapers:
      cpu:
      memory:
      disk:
      network:

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 256          # agents stay lean
    spike_limit_mib: 64

  k8sattributes:            # attach pod/namespace/deployment metadata
    extract:
      metadata:
        - k8s.pod.name
        - k8s.namespace.name
        - k8s.deployment.name
        - k8s.node.name

  resourcedetection:
    detectors: [env, k8snode, system]

  batch:
    send_batch_size: 500
    timeout: 2s

exporters:
  otlp:
    endpoint: gateway-collector:4317

service:
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, k8sattributes, resourcedetection, batch]
      exporters:  [otlp]
    metrics:
      receivers:  [otlp, hostmetrics]
      processors: [memory_limiter, k8sattributes, resourcedetection, batch]
      exporters:  [otlp]
```

### Gateway (centralised)

The gateway tier runs centrally as a scaled deployment. It handles expensive operations: tail sampling, PII scrubbing, and routing to multiple backends.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 2000          # gateway needs more headroom
    spike_limit_mib: 400

  tail_sampling:
    decision_wait: 10s       # buffer window; all spans must arrive
    policies:
      - name: keep-errors
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: keep-slow
        type: latency
        latency: {threshold_ms: 1000}
      - name: sample-healthy
        type: probabilistic
        probabilistic: {sampling_percentage: 10}

  attributes:                # scrub PII
    actions:
      - key: user.email
        action: delete
      - key: http.request.header.authorization
        action: delete
      - key: user.id
        action: hash         # hash instead of delete for debugging

  batch:
    send_batch_size: 2000
    timeout: 10s

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
  otlp/loki:
    endpoint: loki:3100

service:
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, tail_sampling, attributes, batch]
      exporters:  [otlp/jaeger]
    metrics:
      receivers:  [otlp]
      processors: [memory_limiter, attributes, batch]
      exporters:  [prometheusremotewrite]
    logs:
      receivers:  [otlp]
      processors: [memory_limiter, attributes, batch]
      exporters:  [otlp/loki]
```

---

## Topology: Single vs. Two-Tier

### Single Collector

```
App → Collector → Backend
```

- One config file, low operational overhead
- Suitable for small apps and single services
- At very low volume, you can skip the Collector and export directly from the SDK

### Two-Tier (Agent + Gateway)

```
Services → Agent (per node) → Gateway (central, ×N) → Backends
```

Use this topology when you need:

| Need | Why two-tier solves it |
|------|----------------------|
| Kubernetes metadata | Agent reads the k8s API locally, attaches pod labels |
| Tail-based sampling | Gateway buffers entire traces before deciding |
| Multiple backends | Gateway routes per signal type |
| Isolation | Agent failure affects one node; gateway failure is recoverable with buffering |
| Scale | Gateway replicas scale independently of app pods |

**Constraint:** Tail-based sampling requires that all spans for a given trace arrive at the same gateway instance. Use consistent hashing on `trace_id` for load balancing across gateway replicas.

---

## Critical Processors — Rules

### memory_limiter
- **Always first in every pipeline**
- When memory exceeds `limit_mib`, the Collector starts returning errors to senders
- Senders (SDKs and agent collectors) will retry with backoff
- This is intentional: a degraded pipeline is better than a crashed one

### batch
- **Always last in every pipeline**
- Buffers data and flushes on `send_batch_size` or `timeout`, whichever comes first
- Without batching, every span is a separate network call — which will overwhelm most backends

### filter (common recipe — drop health checks)

```yaml
filter:
  traces:
    span:
      - 'attributes["http.route"] == "/healthz"'
      - 'attributes["http.route"] == "/ready"'
      - 'attributes["http.route"] == "/metrics"'
```

Health check endpoints are typically polled every few seconds. Without filtering, they create a constant stream of noise that inflates costs and obscures real traffic in your traces.

---

## What's Next

- **[04 — Sampling](otel-sampling.md)** — Tail-based sampling policy design, volume calculations, error coverage guarantees
- **[05 — Distributed Tracing](otel-distributed.md)** — Context propagation and trace correlation across services

---

[← Back to Guide Index](otel-overview.md)
