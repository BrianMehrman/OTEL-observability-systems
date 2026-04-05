# Non-Production Observability Options

**Visual guide:** `otel-nonprod-options.html`
**Context:** Production will keep Datadog. This document compares self-hosted, zero-cost options for dev and staging environments. All options receive OTEL telemetry via the same `opentelemetry-ruby` instrumentation used in production — only the OTEL Collector's exporter config differs per environment.

---

## Environment Tiers

Different non-production environments have different needs:

| Tier | Persistence | Signals needed | Audience | Priority |
|------|-------------|----------------|----------|----------|
| **Local dev** | Ephemeral (restart = wipe is fine) | Traces (primarily), logs | Individual developer | Instant startup, zero config |
| **CI / test** | None | Traces only (optional) | CI pipeline | Optional — no UI needed |
| **Staging** | Persistent (survives restarts) | Traces + Metrics + Logs | QA, product, developers | Mirrors production observability |

---

## Option 1 — Grafana LGTM (Single Container)

**Best for: Local dev**

Grafana maintains an official all-in-one Docker image (`grafana/otel-lgtm`) that ships Loki + Grafana + Tempo + Mimir (Prometheus-compatible) in a single container. It accepts OTLP natively on port 4317/4318 with zero configuration.

```yaml
# docker-compose.dev.yml
services:
  rails:
    environment:
      OTEL_SERVICE_NAME: rails-app
      OTEL_RESOURCE_ATTRIBUTES: "deployment.environment=development"
      OTEL_EXPORTER_OTLP_ENDPOINT: http://lgtm:4317

  lgtm:
    image: grafana/otel-lgtm:latest
    ports:
      - "3000:3000"   # Grafana UI
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
```

**No OTEL Collector needed** — Rails sends OTLP directly to the LGTM container.

### Why this works for local dev

| Concern | Answer |
|---------|--------|
| Startup time | `docker compose up` — one image pull, ready in ~15 seconds |
| Configuration | Zero — accepts all three signals out of the box |
| RAM | ~300–500 MB total |
| Persistence | In-memory (ephemeral by design — perfect for local) |
| Grafana access | `http://localhost:3000` (admin/admin) |

### Limitation

Not suitable for staging — in-memory storage means all data is lost on container restart, and there is no support for external object storage configuration.

---

## Option 2 — Jaeger All-in-One

**Best for: Local dev (traces only)**

When developers only need to debug request waterfalls — slow queries, N+1 problems, downstream service calls — Jaeger all-in-one is the lightest possible option. A single Docker container with a built-in UI.

> **Important:** Always pin to a specific Jaeger **v1.x** tag. The `:latest` tag now pulls **Jaeger v2**, which removed `COLLECTOR_OTLP_ENABLED` and other CLI flags. OTLP is always-on in v2 but requires a YAML config file — far more setup than needed for local dev.

```yaml
# docker-compose.dev.yml
services:
  rails:
    environment:
      OTEL_SERVICE_NAME: rails-app
      OTEL_EXPORTER_OTLP_ENDPOINT: http://jaeger:4317

  jaeger:
    image: jaegertracing/all-in-one:1.65.0   # pin to v1 — do NOT use :latest (pulls v2)
    ports:
      - "16686:16686"   # Jaeger UI
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP
    environment:
      COLLECTOR_OTLP_ENABLED: "true"         # v1 flag — enables OTLP receiver
      SPAN_STORAGE_TYPE: badger              # disk-backed (survives restarts)
    volumes:
      - jaeger-data:/tmp/badger

volumes:
  jaeger-data:
```

### Why this works for local dev

| Concern | Answer |
|---------|--------|
| RAM | ~100–200 MB |
| Signals | Traces only (no logs, no metrics) |
| Storage | Badger (persisted to disk) or memory |
| UI | Clean trace waterfall viewer at `http://localhost:16686` |
| Config | Zero — OTLP enabled by a single env var |

### Limitation

No log or metric support. Use Grafana LGTM if you need all three signals locally.

---

## Option 3 — SigNoz

**Best for: Staging**

SigNoz is the strongest all-in-one option for a shared staging environment. One Docker Compose deployment gives you traces, metrics, and logs with trace-to-log correlation built in. ClickHouse storage is significantly cheaper and faster than Elasticsearch.

```bash
# Official install
git clone -b main https://github.com/SigNoz/signoz.git
cd signoz/deploy
docker compose -f docker/clickhouse-setup/docker-compose.yaml up -d
```

```yaml
# OTEL Collector config for staging (routes to SigNoz)
# otel-collector-config.staging.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  otlp/signoz:
    endpoint: signoz-otel-collector:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers:  [otlp]
      exporters:  [otlp/signoz]
    metrics:
      receivers:  [otlp]
      exporters:  [otlp/signoz]
    logs:
      receivers:  [otlp]
      exporters:  [otlp/signoz]
```

### Resource estimate

| Container | RAM (typical) |
|-----------|-------------|
| signoz-otel-collector | 100–200 MB |
| clickhouse | 500 MB – 1 GB |
| signoz-frontend | 50–100 MB |
| signoz-query-service | 100–200 MB |
| zookeeper | 100–200 MB |
| **Total** | **~850 MB – 1.7 GB** |

### Why this works for staging

| Concern | Answer |
|---------|--------|
| Signals | All three — traces, metrics, logs |
| Persistence | ClickHouse on disk — survives restarts |
| Trace → log correlation | Built-in |
| Dashboard / alerting | Built-in SigNoz UI |
| Operational complexity | Low — single `docker compose up` |
| SaaS cost | $0 |
| License | Apache 2.0 |

---

## Option 4 — Grafana OSS Stack

**Best for: Staging (when team knows Grafana/Prometheus)**

The canonical OTEL-native Grafana stack: Loki (logs) + Tempo (traces) + Prometheus (metrics) + Grafana (UI). More components than SigNoz but more flexible — each piece can be upgraded or replaced independently.

```yaml
# Relevant services in docker-compose.staging.yml
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./otel-collector-config.staging.yaml:/etc/otelcol-contrib/config.yaml

  loki:
    image: grafana/loki:latest

  prometheus:
    image: prom/prometheus:latest

  tempo:
    image: grafana/tempo:latest

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
```

```yaml
# otel-collector-config.staging.yaml — exports to Grafana stack
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
      exporters:  [otlp/tempo]
    metrics:
      receivers:  [otlp]
      exporters:  [prometheusremotewrite]
    logs:
      receivers:  [otlp]
      exporters:  [loki]
```

### Resource estimate

| Container | RAM (typical) |
|-----------|-------------|
| otel-collector | 50–150 MB |
| loki | 100–300 MB |
| prometheus | 100–300 MB |
| tempo | 100–300 MB |
| grafana | 100–200 MB |
| **Total** | **~450 MB – 1.25 GB** |

### Why this works for staging

| Concern | Answer |
|---------|--------|
| Signals | All three — traces, metrics, logs |
| Persistence | Disk or S3 (Loki and Tempo both support S3) |
| Dashboard / alerting | Grafana — team probably already knows this |
| Trace → log correlation | Requires Grafana datasource link config (15 min) |
| Operational complexity | Medium-High — 5 containers, each with its own config |
| SaaS cost | $0 |
| License | All Apache 2.0 |

---

## Option 5 — Uptrace

**Best for: Staging (minimal footprint alternative to SigNoz)**

Uptrace is a lightweight, open-source APM that accepts OTLP natively. It backs traces, metrics, and logs with ClickHouse. It requires far fewer containers than SigNoz and has a simpler setup, making it a good middle ground between Jaeger (too minimal) and SigNoz (slightly heavier).

```yaml
# docker-compose.staging.yml (Uptrace)
services:
  uptrace:
    image: uptrace/uptrace:latest
    ports:
      - "14317:14317"   # OTLP gRPC
      - "14318:14318"   # OTLP HTTP
      - "14320:14320"   # Uptrace UI
    volumes:
      - ./uptrace.yml:/etc/uptrace/uptrace.yml
    depends_on:
      - clickhouse

  clickhouse:
    image: clickhouse/clickhouse-server:latest
    volumes:
      - ch-data:/var/lib/clickhouse
    ulimits:
      nofile:
        soft: 262144
        hard: 262144

volumes:
  ch-data:
```

```yaml
# otel-collector-config.staging.yaml — routes to Uptrace
exporters:
  otlp/uptrace:
    endpoint: uptrace:14317
    headers:
      uptrace-dsn: http://project1_secret_token@localhost:14317/1
    tls:
      insecure: true
```

### Resource estimate

| Container | RAM (typical) |
|-----------|-------------|
| uptrace | 50–150 MB |
| clickhouse | 500 MB – 1 GB |
| **Total** | **~550 MB – 1.15 GB** |

### Why this works for staging

| Concern | Answer |
|---------|--------|
| Signals | All three — traces, metrics, logs |
| Footprint | Lighter than SigNoz — fewer containers |
| Persistence | ClickHouse on disk |
| Dashboard / alerting | Built-in Uptrace UI |
| Trace → log correlation | Built-in |
| SaaS cost | $0 |
| License | BSL 1.1 (source-available, free to self-host) |
| Maturity | Smaller community than SigNoz or Grafana |

---

## Side-by-Side Comparison

| | Grafana LGTM | Jaeger All-in-One | SigNoz | Grafana OSS | Uptrace |
|-|:------------:|:-----------------:|:------:|:-----------:|:-------:|
| **Best env** | Local dev | Local dev | Staging | Staging | Staging |
| **Signals** | T + M + L | Traces only | T + M + L | T + M + L | T + M + L |
| **RAM** | ~400 MB | ~150 MB | ~1.3 GB | ~700 MB | ~700 MB |
| **# Containers** | 1 | 1 | 5+ | 5 | 2 |
| **Persistence** | In-memory | Badger/disk | ClickHouse | Disk/S3 | ClickHouse |
| **OTLP native** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Trace ↔ log link** | ✅ | ❌ | ✅ | ⚠️ config needed | ✅ |
| **Dashboard/alerts** | Grafana | Jaeger UI | SigNoz UI | Grafana | Uptrace UI |
| **Setup complexity** | Minimal | Minimal | Low | Medium | Low |
| **License** | Apache 2.0 | Apache 2.0 | Apache 2.0 | Apache 2.0 | BSL 1.1 |

---

## Recommended Pairing

```
Local Development:  Grafana LGTM (single container, OTLP-native, all signals)
                    — or —
                    Jaeger All-in-One (if traces are the only need)

Staging:            SigNoz (simplest to operate, all signals, trace-to-log built-in)
                    — or —
                    Grafana OSS (if team uses Grafana and wants max flexibility)

Production:         Datadog (unchanged)
```

---

## OTEL Collector Routing Config

The OTEL Collector is the only piece that changes between environments. Rails instrumentation code is identical everywhere.

```yaml
# otel-collector-config.dev.yaml → Grafana LGTM (no Collector needed; Rails sends direct)
# otel-collector-config.staging.yaml → SigNoz

exporters:
  otlp/signoz:               # swap for otlp/uptrace, loki/tempo/prometheus, etc.
    endpoint: signoz-otel-collector:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:   { receivers: [otlp], exporters: [otlp/signoz] }
    metrics:  { receivers: [otlp], exporters: [otlp/signoz] }
    logs:     { receivers: [otlp], exporters: [otlp/signoz] }
```

```yaml
# docker-compose env var per environment
services:
  rails:
    environment:
      OTEL_SERVICE_NAME: rails-app
      OTEL_RESOURCE_ATTRIBUTES: "deployment.environment=${RAILS_ENV}"
      # Local dev (Grafana LGTM — no Collector):
      OTEL_EXPORTER_OTLP_ENDPOINT: http://lgtm:4317
      # Staging (via OTEL Collector):
      # OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
      # Production (via OTEL Collector → Datadog):
      # OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
```

---

## Related Guides

- **[Rails Stack Analysis](otel-rails-stack.md)** — Options A/B/C for the Rails pipeline
- **[Migration Plan](otel-migration-plan.md)** — Phase-by-phase migration plan
- **[Distributed System Migration](otel-distributed-migration.md)** — Full polyglot stack migration guide
- **[03 — Collector](otel-collector.md)** — OTEL Collector pipeline and config reference

[← Back to Guide Index](otel-overview.md)
