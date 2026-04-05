# 06 — Ecosystem: Backends, Platforms & Stack Choices

**Visual guide:** `otel-ecosystem.html`
**Covers:** The OSS backend landscape, SaaS observability platforms, which stack to choose at each scale tier, how to fan out to multiple backends, and when to self-host vs. buy.

---

## The OTEL Ecosystem Map

OTEL is a collection pipeline, not a storage or visualisation layer. The ecosystem splits cleanly into two layers:

```
[ Your Application ]
        │  OTLP
        ▼
[ OTEL Collector ]   ← vendor-neutral processing
        │
        ├──────────────────────────────────────┐
        ▼                                      ▼
[ OSS Backends ]                    [ SaaS Platforms ]
  Jaeger (traces)                     Grafana Cloud
  Prometheus (metrics)                Datadog
  Loki (logs)                         Honeycomb
  Tempo (traces)                      Dynatrace
  Grafana (dashboards)                New Relic
```

Your choice of backend(s) is independent of how you instrument your code. Switching from one backend to another means changing exporter config in the Collector — not touching application code.

---

## OSS Backends

### Jaeger — Distributed Tracing

Jaeger is the CNCF reference implementation for distributed trace storage and visualisation. It was purpose-built for OTEL-style tracing.

- **Accepts:** Traces (OTLP, Jaeger native)
- **Storage options:** In-memory (dev), Elasticsearch, Cassandra, Badger
- **UI:** Trace waterfall, service dependency graph, trace comparisons
- **Best for:** Teams that want full control over trace storage and query
- **Limitation:** Traces only — no metrics or logs

### Prometheus — Metrics

The de facto standard for metrics in cloud-native environments. Works by **scraping** `/metrics` endpoints, but OTEL can push via `prometheusremotewrite`.

- **Accepts:** Metrics
- **Storage:** Local TSDB (time-series database), or remote write to Thanos/Cortex/Mimir for HA
- **Query language:** PromQL — powerful and widely adopted
- **Alerting:** Via AlertManager (ships separately)
- **Best for:** Any Kubernetes environment — already installed in most clusters
- **Limitation:** Metrics only. Short-term local retention by default.

### Loki — Logs

Grafana Labs' log aggregation system, designed to index log metadata only (not full text), keeping storage costs low.

- **Accepts:** Logs
- **Query language:** LogQL (mirrors PromQL structure)
- **Storage:** Object storage (S3, GCS, Azure Blob)
- **Best for:** Teams already using Grafana who want low-cost log storage
- **Limitation:** Full-text search is slower than Elasticsearch-based solutions

### Tempo — Traces

Grafana Labs' trace backend. Designed specifically to store traces in object storage at minimal cost.

- **Accepts:** Traces (OTLP, Jaeger, Zipkin)
- **Storage:** Object storage only — no indexing overhead
- **Query:** TraceQL, or by `trace_id` from Grafana
- **Best for:** Teams using the Grafana stack who want cheap trace storage
- **Limitation:** Querying requires a `trace_id` — no full-text span search

### Grafana — Dashboards & Alerting

Grafana is the visualisation and alerting layer that ties everything together. It is not a backend — it queries backends via data sources.

- **Data sources:** Prometheus, Loki, Tempo, Jaeger, and dozens more
- **Dashboards:** Pre-built community dashboards for nearly every stack
- **Alerting:** Unified alerting across all data sources
- **Best for:** Any team using OSS backends — it is the standard UI for the OSS stack

---

## The Standard OSS Stack

This combination covers all three signals and is the default in most self-hosted cloud-native environments:

```
Traces  →  Tempo   ─┐
Metrics →  Prometheus ─┤  Grafana (unified UI)
Logs    →  Loki    ─┘
```

All three are deployable via Helm. Grafana provides a single pane of glass with native datasource integrations for each.

---

## SaaS Platforms

SaaS platforms bundle storage, visualisation, and alerting into a single product. They accept OTLP directly or via the Collector.

| Platform | Signal support | Pricing model | Strengths |
|----------|---------------|---------------|-----------|
| **Grafana Cloud** | Traces, metrics, logs | Usage-based (free tier) | Best OSS compatibility; same UX as self-hosted Grafana |
| **Datadog** | Traces, metrics, logs, RUM | Host/container based | Widest integrations; strong APM and anomaly detection |
| **Honeycomb** | Traces, logs | Event-based | Best-in-class trace query (BubbleUp); high-cardinality search |
| **Dynatrace** | Traces, metrics, logs, AI | Host-based | AI-driven root cause analysis; strong enterprise features |
| **New Relic** | Traces, metrics, logs | Data-ingest based | Large community; broad language support; free tier |

### Grafana Cloud
The hosted version of the OSS Grafana stack. Receives OTLP natively. For teams that want the OSS toolchain without running the infrastructure.

```yaml
# Collector exporter for Grafana Cloud
exporters:
  otlp/grafana-traces:
    endpoint: tempo-prod-xx.grafana.net:443
    headers:
      authorization: "Basic ${env:GRAFANA_CLOUD_TOKEN}"
  prometheusremotewrite/grafana-metrics:
    endpoint: https://prometheus-prod-xx.grafana.net/api/prom/push
    headers:
      authorization: "Basic ${env:GRAFANA_CLOUD_TOKEN}"
  loki/grafana-logs:
    endpoint: https://logs-prod-xx.grafana.net/loki/api/v1/push
    headers:
      authorization: "Basic ${env:GRAFANA_CLOUD_TOKEN}"
```

### Datadog
Accepts OTLP via its own Agent (which acts as a Collector) or directly via the Collector's `datadog` exporter.

```yaml
exporters:
  datadog:
    api:
      key: ${env:DD_API_KEY}
      site: datadoghq.com
```

### Honeycomb
Best suited for high-cardinality trace queries. Accepts OTLP directly.

```yaml
exporters:
  otlp/honeycomb:
    endpoint: api.honeycomb.io:443
    headers:
      x-honeycomb-team: ${env:HONEYCOMB_API_KEY}
```

---

## Stack Recommendations by Scale Tier

### Small Application (single service, low traffic)

**Recommendation:** SaaS platform, OTLP direct from SDK or single Collector

```
[ Service ]
    │ OTLP
    ▼
[ Grafana Cloud / New Relic / Honeycomb ]
```

- Skip self-hosted backends — operational overhead not worth it at this scale
- Free tiers on Grafana Cloud and New Relic cover most small workloads
- One Collector (optional) for batching and basic enrichment

| Concern | Choice |
|---------|--------|
| Cost | Grafana Cloud free tier or New Relic free tier |
| Simplicity | Export OTLP direct from SDK |
| Traces first | Honeycomb — best trace query UX |

### Medium (multiple services, moderate traffic)

**Recommendation:** Managed SaaS or self-hosted OSS stack with a single Collector tier

```
[ Services ]
    │ OTLP
    ▼
[ Collector ]
    ├── Tempo (traces)
    ├── Prometheus (metrics)
    └── Loki (logs)
         └── Grafana (UI)
```

- Self-host if you have ops capacity and data residency requirements
- SaaS if you want to move fast without managing infrastructure
- One Collector per service cluster is sufficient

### Large (many services, high traffic, multiple teams)

**Recommendation:** Two-tier Collector with OSS backends or enterprise SaaS

```
[ Services ]
    │ OTLP
    ▼
[ Agent Collectors ]   (per node)
    │ forward
    ▼
[ Gateway Collectors ]   (central, scaled)
    ├── tail sampling
    ├── PII scrubbing
    └── fan-out
         ├── Tempo
         ├── Prometheus / Thanos
         └── Loki
```

- OSS stack with Thanos (HA Prometheus) or Mimir for metrics durability
- Datadog or Dynatrace if you need AI-driven analysis or broad integrations
- Different teams can own different backends — fan-out in the gateway

---

## Fan-Out: Sending to Multiple Backends

The Collector can route a single signal to multiple backends simultaneously:

```yaml
service:
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, batch]
      exporters:  [otlp/jaeger, otlp/honeycomb]   # both receive traces

    metrics:
      receivers:  [otlp]
      processors: [memory_limiter, batch]
      exporters:  [prometheusremotewrite, datadog]  # both receive metrics
```

Common reasons to fan out:
- **Migration** — sending to old and new backend simultaneously during cutover
- **Ownership** — platform team owns Prometheus; app team owns Datadog APM
- **Redundancy** — mission-critical traces sent to two backends for durability

---

## Self-Host vs. SaaS Decision Guide

| Factor | Lean self-host | Lean SaaS |
|--------|---------------|-----------|
| Data residency requirements | ✓ | |
| Ops capacity available | ✓ | |
| High data volume (cost sensitive) | ✓ | |
| Need to iterate fast | | ✓ |
| Small or no ops team | | ✓ |
| Need AI/ML-driven analysis | | ✓ |
| Enterprise compliance requirements | | ✓ |

**Rule of thumb:** Start with SaaS. Migrate to self-hosted OSS when your data volume makes SaaS cost prohibitive and you have the ops team to support it.

---

## Tools Reference

For a comprehensive side-by-side comparison of all tools in this space, see the dedicated reference:

**[Tools Reference →](otel-tools.md)** — Instrumentation libraries & agents, backend storage systems, and SaaS platforms in a single set of comparison tables.

---

*See also: **[08 — Kubernetes](otel-kubernetes.md)** for deploying these backends in a Kubernetes cluster.*
