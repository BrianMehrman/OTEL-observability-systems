# Rails Observability Stack Analysis

**Visual guide:** `otel-rails-stack.html`
**Migration plan:** `otel-migration-plan.md` / `otel-migration-plan.html`
**Covers:** Current stack audit, identified pain points, three replacement options (Grafana OSS, SigNoz, and a Datadog hybrid for prod/non-prod), Rails OTEL SDK setup, and a side-by-side comparison.

---

## Current Stack

```
[ Rails App ]
    │  writes structured logs
    ▼
[ Log File on Disk ]
    │  Logstash file input (tails file)
    ▼
[ Logstash ]   (JVM process, ~1–2 GB RAM)
    │  Logstash Elasticsearch output
    ▼
[ Elasticsearch ]   (JVM process, ~4–8 GB RAM minimum)
    │  Datadog Elasticsearch integration / log forwarding
    ▼
[ Datadog ]   (SaaS — billed per host)
```

**Deployment context:** Docker containers on a single EC2 instance.

---

## Pain Points in the Current Stack

### 1. High resource cost on EC2

| Component | Runtime | Typical RAM | Notes |
|-----------|---------|-------------|-------|
| Logstash | JVM | 1–2 GB | Idle still consumes significant memory |
| Elasticsearch | JVM | 4–8 GB minimum | Heap must be ≥ 50% of available RAM for stable operation |
| Rails (Puma) | Ruby | 512 MB–1 GB | Depends on worker count |
| **Total** | | **5–11 GB+** | Leaves little headroom on common instance sizes |

Elasticsearch and Logstash are both JVM processes with large minimum footprints. On a t3.xlarge (16 GB) you're burning most of your RAM before your application even runs.

### 2. Datadog host-based pricing

Datadog bills per host per month. A single EC2 instance with the Datadog Agent means a fixed monthly line item regardless of traffic. Adding APM (traces) and Infrastructure Monitoring adds additional per-host costs. Logs ingest is billed separately by GB.

### 3. Logs only — no traces or metrics

The current pipeline carries only log data. There is:
- **No distributed tracing** — you cannot see request waterfalls or identify which service call is slow
- **No application metrics** — no p99 latency, error rate counters, or throughput dashboards sourced from the application itself
- **No infrastructure metrics** — CPU, memory, and disk come only from the Datadog Agent, not from the OTEL pipeline

### 4. Fragile log pipeline

The file-based log shipping approach has several failure modes:
- **Log rotation** — if Rails rotates logs and Logstash hasn't checkpointed, entries can be missed or re-read
- **Disk fill** — if Logstash falls behind, log files accumulate and can fill the disk
- **Single point of failure** — Logstash is the only buffer between Rails and Elasticsearch; if it crashes, logs are lost during the gap

### 5. Elasticsearch operational overhead

Elasticsearch requires index management (ILM policies, shard sizing), disk monitoring, and regular tuning. On a single EC2 node there is no replication, so a disk failure loses all log history.

---

## Option A — Grafana OSS Stack

Replace the entire pipeline with the standard OTEL-native Grafana stack. All components run as Docker containers on the same EC2 instance.

```
[ Rails App ]
    │  OTLP (gRPC :4317)   ← opentelemetry-ruby gem
    ▼
[ OTEL Collector ]   (Go binary, ~50–150 MB RAM)
    ├── receivers: otlp, filelog (legacy log tail, transitional)
    ├── processors: batch, memory_limiter, resourcedetection
    └── exporters:
         ├── logs    → Loki
         ├── metrics → Prometheus (remote_write)
         └── traces  → Tempo
         │
         ▼
┌─────────────────────────────┐
│  Loki       (logs)          │  Object storage or local disk
│  Prometheus (metrics)       │  Local TSDB (15-day retention)
│  Tempo      (traces)        │  Local disk or S3
└─────────────────────────────┘
         │
         ▼
[ Grafana ]   (unified UI — dashboards, alerting, trace linking)
```

### Why this works

| Concern | How it's addressed |
|---------|-------------------|
| Memory footprint | OTEL Collector uses ~50–150 MB vs 5–10 GB for Logstash + Elasticsearch |
| Cost | No SaaS fees; only EC2 instance cost |
| All three signals | Traces via Tempo, metrics via Prometheus, logs via Loki |
| Vendor lock-in | Every component is Apache 2.0 open source |
| Migration path | OTEL Collector can tail the existing log file while Rails is being instrumented |

### Resource estimate (Docker Compose on EC2)

| Container | RAM (typical) |
|-----------|-------------|
| otel-collector | 50–150 MB |
| loki | 100–300 MB |
| prometheus | 100–300 MB |
| tempo | 100–300 MB |
| grafana | 100–200 MB |
| **Total** | **~450 MB – 1.25 GB** |

Compare to current: 5–11 GB for Logstash + Elasticsearch alone.

### Docker Compose (Option A)

```yaml
version: "3.8"
services:

  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./otel-collector-config.yaml:/etc/otelcol-contrib/config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
    restart: unless-stopped

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - loki-data:/loki
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    restart: unless-stopped

  tempo:
    image: grafana/tempo:latest
    ports:
      - "3200:3200"
      - "4317"          # internal OTLP only
    volumes:
      - ./tempo.yaml:/etc/tempo.yaml
      - tempo-data:/tmp/tempo
    command: ["-config.file=/etc/tempo.yaml"]
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=changeme
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    restart: unless-stopped

volumes:
  loki-data:
  prometheus-data:
  tempo-data:
  grafana-data:
```

### OTEL Collector config (Option A)

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  # Transitional: tail existing log file while migrating
  filelog:
    include: [/var/log/rails/production.log]
    operators:
      - type: json_parser        # if Rails uses lograge with JSON output
        timestamp:
          parse_from: attributes.time
          layout: "%Y-%m-%dT%H:%M:%S.%LZ"

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 128
    spike_limit_mib: 32
  resourcedetection:
    detectors: [env, system, docker]
  batch:
    send_batch_size: 500
    timeout: 5s

exporters:
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, resourcedetection, batch]
      exporters:  [otlp/tempo]
    metrics:
      receivers:  [otlp]
      processors: [memory_limiter, resourcedetection, batch]
      exporters:  [prometheusremotewrite]
    logs:
      receivers:  [otlp, filelog]
      processors: [memory_limiter, resourcedetection, batch]
      exporters:  [loki]
```

---

## Option B — SigNoz (All-in-One)

SigNoz is a single-product OTEL-native observability platform. It replaces Logstash, Elasticsearch, and Datadog with one Docker Compose deployment backed by ClickHouse.

```
[ Rails App ]
    │  OTLP (gRPC :4317)   ← opentelemetry-ruby gem
    ▼
[ SigNoz OTel Collector ]   (built into SigNoz)
    │
    ▼
[ ClickHouse ]   (columnar DB — much cheaper storage than Elasticsearch)
    │
    ▼
[ SigNoz UI ]   (traces, metrics, logs in one interface)
    │  Alerts, dashboards, trace-to-log linking built-in
```

### Why this works

| Concern | How it's addressed |
|---------|-------------------|
| Simplicity | One product, one Docker Compose, one UI |
| ClickHouse vs Elasticsearch | ClickHouse uses 10–20× less disk space for the same log volume |
| OTEL native | Accepts OTLP directly — no Logstash translation layer |
| All three signals | Traces, metrics, logs all in the same product |
| Alerting | Built-in, no Grafana needed |

### Resource estimate (SigNoz Docker Compose)

| Container | RAM (typical) |
|-----------|-------------|
| signoz-otel-collector | 100–200 MB |
| clickhouse | 500 MB – 1 GB |
| signoz-frontend | 50–100 MB |
| signoz-query-service | 100–200 MB |
| zookeeper (ClickHouse dependency) | 100–200 MB |
| **Total** | **~850 MB – 1.7 GB** |

Still dramatically lower than the current Logstash + Elasticsearch footprint.

### SigNoz Docker Compose (minimal)

```bash
# Official install — pulls the full stack
git clone -b main https://github.com/SigNoz/signoz.git
cd signoz/deploy
docker compose -f docker/clickhouse-setup/docker-compose.yaml up -d
```

SigNoz exposes:
- **UI:** `http://your-ec2:3301`
- **OTLP gRPC:** `http://your-ec2:4317`
- **OTLP HTTP:** `http://your-ec2:4318`

No additional configuration needed to receive OTLP from Rails.

---

## Option C — Datadog (Production) + Self-hosted (Non-production)

Keep Datadog where its value is highest — production incidents, SLOs, and on-call alerting — while eliminating the heavy Logstash + Elasticsearch pipeline and removing Datadog host fees from every non-production environment.

The key insight: the OTEL Collector replaces Logstash entirely and routes telemetry to different backends depending on the environment. Rails instrumentation stays identical across all environments.

### Architecture

```
Production environment (EC2):
[ Rails App ]
    │  OTLP (gRPC :4317)   ← opentelemetry-ruby gem
    ▼
[ OTEL Collector ]   (~100 MB RAM — replaces Logstash + Elasticsearch)
    │  datadog exporter (traces + metrics + logs)
    ▼
[ Datadog ]   (SaaS — APM, log management, SLOs, alerting, mobile app)

──────────────────────────────────────────────────────────────

Dev / Staging environment (smaller EC2):
[ Rails App ]
    │  OTLP (gRPC :4317)   ← same gem, same config, different env var
    ▼
[ OTEL Collector ]   (same binary, different config file)
    │
    ▼
[ SigNoz or Grafana OSS ]   ($0 — self-hosted, full traces + metrics + logs)
```

The OTEL Collector's `datadog` exporter sends OTLP-native telemetry directly to Datadog without needing a Logstash translation layer or Elasticsearch intermediate store. The Datadog Agent is optional — you can point the OTEL Collector straight at the Datadog OTLP endpoint.

### Why this works

| Concern | How it's addressed |
|---------|-------------------|
| Eliminate Logstash + Elasticsearch | OTEL Collector (~100 MB) replaces both JVM processes (~5–10 GB RAM) |
| Keep Datadog for production | `datadog` exporter in OTEL Collector sends all three signals natively |
| No Datadog fees for dev/staging | Non-prod environments route to free self-hosted stack |
| Same Rails code everywhere | Only `OTEL_EXPORTER_OTLP_ENDPOINT` env var differs per environment |
| Full observability everywhere | Traces, metrics, and logs in all environments |

### OTEL Collector config — Production (sends to Datadog)

```yaml
# otel-collector-config.production.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 128
    spike_limit_mib: 32
  resourcedetection:
    detectors: [env, system, docker]
  batch:
    send_batch_size: 500
    timeout: 5s

exporters:
  datadog:
    api:
      site: datadoghq.com       # or datadoghq.eu
      key: ${DD_API_KEY}

service:
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, resourcedetection, batch]
      exporters:  [datadog]
    metrics:
      receivers:  [otlp]
      processors: [memory_limiter, resourcedetection, batch]
      exporters:  [datadog]
    logs:
      receivers:  [otlp]
      processors: [memory_limiter, resourcedetection, batch]
      exporters:  [datadog]
```

### OTEL Collector config — Dev / Staging (sends to SigNoz)

```yaml
# otel-collector-config.staging.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 128
    spike_limit_mib: 32
  batch:
    send_batch_size: 500
    timeout: 5s

exporters:
  otlp/signoz:
    endpoint: signoz-otel-collector:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, batch]
      exporters:  [otlp/signoz]
    metrics:
      receivers:  [otlp]
      processors: [memory_limiter, batch]
      exporters:  [otlp/signoz]
    logs:
      receivers:  [otlp]
      processors: [memory_limiter, batch]
      exporters:  [otlp/signoz]
```

### Docker Compose — environment variables per environment

```yaml
# docker-compose.production.yml
services:
  rails:
    environment:
      OTEL_SERVICE_NAME: rails-app
      OTEL_RESOURCE_ATTRIBUTES: "deployment.environment=production"
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./otel-collector-config.production.yaml:/etc/otelcol-contrib/config.yaml
    environment:
      DD_API_KEY: ${DD_API_KEY}
```

```yaml
# docker-compose.staging.yml
services:
  rails:
    environment:
      OTEL_SERVICE_NAME: rails-app
      OTEL_RESOURCE_ATTRIBUTES: "deployment.environment=staging"
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./otel-collector-config.staging.yaml:/etc/otelcol-contrib/config.yaml
```

### Resource footprint comparison

| Environment | Before (current) | After (Option C) |
|-------------|-----------------|-----------------|
| Production pipeline | Logstash + Elasticsearch ≈ 6–12 GB RAM | OTEL Collector ≈ 100 MB |
| Dev / Staging pipeline | Same 6–12 GB + Datadog host fee | OTEL Collector ≈ 100 MB + SigNoz ≈ 1.3 GB |
| Datadog host fees | All environments | Production only |

### Cost impact

Datadog charges per host per month. If you run two non-production environments (dev + staging), eliminating Datadog from those saves 2× the monthly host fee. Removing Logstash and Elasticsearch also allows you to downsize EC2 instance types — saving additional compute cost on the production host too.

---

## Rails Instrumentation (All Options)

Both options use the same Rails instrumentation. The only difference is the OTLP endpoint URL.

### Gemfile

```ruby
gem 'opentelemetry-sdk'
gem 'opentelemetry-exporter-otlp'
gem 'opentelemetry-instrumentation-all'  # auto-instruments Rails, ActiveRecord, Redis, Sidekiq, etc.
```

```bash
bundle install
```

### config/initializers/opentelemetry.rb

```ruby
require 'opentelemetry/sdk'
require 'opentelemetry/exporter/otlp'
require 'opentelemetry/instrumentation/all'

OpenTelemetry::SDK.configure do |c|
  c.service_name    = ENV.fetch('OTEL_SERVICE_NAME', 'rails-app')
  c.service_version = ENV.fetch('OTEL_SERVICE_VERSION', '1.0.0')

  c.use_all  # auto-instruments Rails, Action Pack, Active Record, Net::HTTP, Faraday, Redis, Sidekiq, etc.

  c.add_span_processor(
    OpenTelemetry::SDK::Trace::Export::BatchSpanProcessor.new(
      OpenTelemetry::Exporter::OTLP::Exporter.new(
        endpoint: ENV.fetch('OTEL_EXPORTER_OTLP_ENDPOINT', 'http://otel-collector:4317')
      )
    )
  )
end
```

### Environment variables (docker-compose.yml)

```yaml
services:
  rails:
    environment:
      OTEL_SERVICE_NAME:    rails-app
      OTEL_SERVICE_VERSION: "1.4.2"
      OTEL_RESOURCE_ATTRIBUTES: "deployment.environment=production"
      # Option A — OTEL Collector:
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
      # Option B — SigNoz:
      # OTEL_EXPORTER_OTLP_ENDPOINT: http://signoz-otel-collector:4317
```

### What gets auto-instrumented

The `opentelemetry-instrumentation-all` gem covers:

| Library | Spans produced |
|---------|---------------|
| Rails (Action Controller) | Every request with route, method, status code |
| Active Record | Every SQL query with sanitised statement |
| Action View | Template rendering time |
| Net::HTTP | Outbound HTTP calls |
| Faraday | Outbound HTTP calls (if used) |
| Redis | Every Redis command |
| Sidekiq | Worker enqueue and perform |
| Rack | Middleware chain |

---

## Migration Path (Current → New)

A safe migration avoids a hard cutover. The OTEL Collector's `filelog` receiver allows you to run both pipelines simultaneously during transition.

```
Phase 1 — Parallel run (1–2 weeks)
  Rails → log file → Logstash → Elasticsearch → Datadog   (existing)
  Rails → log file → OTEL Collector filelog → Loki/SigNoz  (new, via filelog receiver)
  Verify: logs appear correctly in new stack

Phase 2 — Add OTEL instrumentation
  Add opentelemetry-ruby gems to Rails
  Rails now sends OTLP directly → OTEL Collector
  Traces and metrics appear for first time

Phase 3 — Decommission old pipeline
  Confirm parity in new stack (30-day retention check)
  Stop Logstash and Elasticsearch containers
  Remove Datadog Agent
  Recover RAM and EC2 instance cost
```

---

## Side-by-Side Comparison

| | Current Stack | Option A — Grafana OSS | Option B — SigNoz | Option C — Datadog (prod) + Self-hosted (non-prod) |
|-|--------------|----------------------|------------------|--------------------------------------------------|
| **Log storage** | Elasticsearch | Loki | ClickHouse | Datadog (prod) / ClickHouse (non-prod) |
| **Trace storage** | ❌ None | Tempo | ClickHouse | Datadog APM (prod) / ClickHouse (non-prod) |
| **Metrics storage** | Datadog only | Prometheus | ClickHouse | Datadog (prod) / ClickHouse (non-prod) |
| **Log shipper** | Logstash | OTEL Collector | OTEL Collector (built-in) | OTEL Collector (all envs) |
| **Dashboards** | Datadog | Grafana | SigNoz UI | Datadog (prod) / SigNoz (non-prod) |
| **Alerting** | Datadog | Grafana Alerts | SigNoz Alerts | Datadog (prod) / SigNoz (non-prod) |
| **OTLP native** | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes |
| **Rails traces** | ❌ No | ✅ Yes (opentelemetry-ruby) | ✅ Yes (opentelemetry-ruby) | ✅ Yes (opentelemetry-ruby) |
| **EC2 RAM (pipeline)** | ~5–11 GB | ~450 MB – 1.25 GB | ~850 MB – 1.7 GB | ~100–300 MB (prod) / ~1.4 GB (non-prod) |
| **Prod SaaS cost** | Datadog host fee + log ingest | $0 | $0 | Datadog production host only |
| **Non-prod SaaS cost** | Datadog host fee per environment | $0 | $0 | $0 (self-hosted) |
| **Number of products** | 3 | 5 | 1 | 2 (Datadog prod / SigNoz non-prod) |
| **Operational complexity** | Medium | Medium–High | Low | Medium |
| **Best for** | — | Teams comfortable with Grafana/PromQL | Teams wanting one product everywhere | Teams that need Datadog for production but want to cut non-prod costs |

### Which to choose

**Choose Option A (Grafana OSS)** if:
- Your team already knows Grafana/Prometheus/PromQL
- You want the most widely adopted OSS stack with the largest community
- You want to add Pyroscope (profiling) later — it integrates natively with Grafana
- You may eventually move to Grafana Cloud as a managed upgrade path

**Choose Option B (SigNoz)** if:
- You want the least operational complexity — one product, one UI, one deploy
- Your team doesn't have existing Grafana expertise
- ClickHouse's storage efficiency matters (much cheaper than Elasticsearch at the same log volume)
- You want trace-to-log correlation out of the box without configuring Grafana datasource links

**Choose Option C (Datadog prod + self-hosted non-prod)** if:
- Datadog is required for production (compliance, existing integrations, on-call workflows, mobile app)
- You want to significantly reduce Datadog spend without abandoning it for production
- You want developers to have full observability in dev/staging without Datadog host fees
- The team prefers keeping the best-in-class tool for production incidents and SLOs

---

## S3 for Long-Term Log Retention (Both Options)

Both Loki and Tempo support S3 as their storage backend, which is significantly cheaper than local EBS for long-term retention:

```yaml
# Loki — S3 storage config
storage_config:
  aws:
    s3: s3://your-bucket/loki
    region: us-east-1
  boltdb_shipper:
    active_index_directory: /loki/index
    shared_store: s3

# Tempo — S3 storage config
storage:
  trace:
    backend: s3
    s3:
      bucket: your-bucket
      region: us-east-1
```

S3 Standard costs ~$0.023/GB/month. Elasticsearch on EBS typically costs ~$0.10/GB/month — roughly 4× more expensive for cold log storage.

---

## Related Guides

- **[Migration Plan](otel-migration-plan.md)** — Phase-by-phase migration from Logstash → OTEL
- **[Non-Prod Options](otel-nonprod-options.md)** — Detailed comparison of dev/staging backends
- **[Distributed System Migration](otel-distributed-migration.md)** — Full polyglot stack migration guide
- **[03 — Collector](otel-collector.md)** — OTEL Collector pipeline and config reference
- **[06 — Ecosystem](otel-ecosystem.md)** — Backend and SaaS platform landscape

[← Back to Guide Index](otel-overview.md)
