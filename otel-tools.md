# Tools Reference: Libraries, Backends & Platforms

**Visual guide:** `otel-tools.html`
**Covers:** Three reference tables spanning the full observability tool landscape — instrumentation libraries and agents, backend storage systems, and commercial/SaaS observability platforms.

---

## How to Read These Tables

- **OTLP native** — accepts OpenTelemetry Protocol directly without a translation layer
- **Signals** — T = Traces, M = Metrics, L = Logs (e.g. `T M L` = all three)
- **License** — Apache 2.0 / MIT = fully open source; SSPL/BUSL = source-available with restrictions; Commercial = proprietary

---

## Table 1 — Instrumentation Libraries & Agents

Tools that run inside (or alongside) your application to produce telemetry.

| Library / Agent | Vendor | Languages | Signals | OTLP native | Auto-instrumentation | License |
|----------------|--------|-----------|---------|-------------|---------------------|---------|
| **OpenTelemetry SDK** | CNCF / Community | Python, Node.js, Go, Java, .NET, Ruby, PHP, Rust, Swift, Erlang, C++ | T M L | ✅ Yes | ✅ Yes (Python, Node, Java, .NET) | Apache 2.0 |
| **OpenTelemetry Java Agent** | CNCF / Community | Java (JVM) | T M L | ✅ Yes | ✅ Yes (100+ frameworks via `-javaagent`) | Apache 2.0 |
| **Elastic APM Agent** | Elastic | Python, Node.js, Go, Java, .NET, Ruby, PHP | T M L | ✅ Yes (via bridge) | ✅ Yes | Apache 2.0 |
| **Datadog APM (ddtrace)** | Datadog | Python, Node.js, Go, Java, Ruby, PHP, .NET | T M L | ⚠️ Partial (sends to DD Agent) | ✅ Yes | BSD 3-Clause |
| **New Relic APM Agent** | New Relic | Python, Node.js, Go, Java, Ruby, PHP, .NET | T M L | ✅ Yes | ✅ Yes | Apache 2.0 |
| **Dynatrace OneAgent** | Dynatrace | All major (bytecode injection) | T M L | ✅ Yes (OTLP ingest) | ✅ Yes (full-stack, kernel-level) | Commercial |
| **Micrometer** | VMware / Community | Java | M | ⚠️ Via OTLP registry | ❌ Manual | Apache 2.0 |
| **Prometheus Client Libs** | CNCF / Community | Go, Python, Java, Ruby, .NET, Rust | M | ❌ (Prometheus format) | ❌ Manual | Apache 2.0 |
| **OpenTracing (legacy)** | CNCF (archived) | Multiple | T | ❌ (pre-OTLP) | ❌ Manual | Apache 2.0 |
| **OpenCensus (legacy)** | Google (archived) | Go, Java, Python, Ruby | T M | ❌ (pre-OTLP) | ❌ Manual | Apache 2.0 |
| **Sentry SDK** | Sentry | 30+ languages | T L | ❌ (Sentry protocol) | ✅ Partial | MIT / Commercial |
| **Pyroscope SDK** | Grafana Labs | Go, Python, Java, Ruby, Rust | Profiles | ❌ | ❌ Manual | Apache 2.0 |

**Key notes:**
- OpenTelemetry SDK is the only vendor-neutral, actively maintained choice — prefer it for new projects
- OpenTracing and OpenCensus are archived; migrate to OTEL SDK via shim libraries
- Vendor agents (Datadog, Dynatrace) offer deeper platform integration at the cost of vendor lock-in

---

## Table 1b — Log Shippers & Forwarders

Tools dedicated to collecting and forwarding log data. These sit **before** or **alongside**
the OTEL Collector in your pipeline and are often already present in Kubernetes clusters.

| Tool | Vendor | Primary role | OTLP output | Footprint | K8s default | License |
|------|--------|-------------|-------------|-----------|-------------|---------|
| **Fluent Bit** | CNCF / Fluent | Lightweight log collector & forwarder | ✅ Yes (OTLP output plugin) | ~1 MB binary, ~10 MB RAM | ✅ Pre-installed on EKS, GKE, AKS | Apache 2.0 |
| **Fluentd** | CNCF / Fluent | Log aggregator & router (EFK stack) | ✅ Yes (`fluent-plugin-opentelemetry`) | ~40 MB, Ruby-based | ⚠️ Common but being replaced by Fluent Bit | Apache 2.0 |
| **Logstash** | Elastic | Log processing pipeline (ELK stack) | ✅ Yes (OpenTelemetry output plugin) | ~500 MB JVM | ❌ No | Elastic License / SSPL |
| **Vector** | Datadog (OSS) | High-performance log / metrics router | ✅ Yes (OTLP sink) | ~30 MB Rust binary | ❌ No | MPL 2.0 |
| **OTEL Collector** | CNCF / Community | Unified traces + metrics + logs pipeline | ✅ Native (OTLP) | ~50–100 MB | ❌ Needs Operator | Apache 2.0 |

**When to use which:**

| Scenario | Best choice |
|----------|------------|
| Already have Fluent Bit on K8s nodes | Keep it; add OTLP output → OTEL Collector |
| Need lightweight log collection at the edge / IoT | Fluent Bit |
| Running the EFK stack today, migrating incrementally | Fluentd → OTEL Collector bridge |
| Want a single tool for all three signals | OTEL Collector (filelog receiver) |
| High-throughput log routing with complex transforms | Vector or Fluent Bit |

**Fluent Bit → OTEL Collector hybrid pattern** (common in Kubernetes):
```
[Pod logs]
    │
    ▼ collected per-node
[Fluent Bit DaemonSet]   ← very low overhead, already present
    │ OTLP / forward protocol
    ▼
[OTEL Collector gateway] ← handles sampling, enrichment, fan-out
    │                │
    ▼                ▼
[Datadog]        [SigNoz / Loki]
```

This is the recommended pattern when your cluster already has Fluent Bit: don't replace it,
route its output into the OTEL Collector and let the Collector handle backend routing.

---

Systems that store and query telemetry data. These are the destination for Collector exporters.

### Trace Backends

| System | Signals | Query language | Storage backend | OTLP native | License | Self-hostable |
|--------|---------|---------------|----------------|-------------|---------|--------------|
| **Jaeger** | T | Jaeger UI / API | Elasticsearch, Cassandra, Badger, in-memory | ✅ Yes | Apache 2.0 | ✅ Yes |
| **Tempo** | T | TraceQL | Object storage (S3, GCS, Azure Blob) | ✅ Yes | Apache 2.0 | ✅ Yes |
| **Zipkin** | T | Zipkin UI / API | Elasticsearch, MySQL, Cassandra, in-memory | ❌ (Zipkin format; OTEL bridge exists) | Apache 2.0 | ✅ Yes |
| **SigNoz** | T M L | SQL-like (ClickHouse) | ClickHouse | ✅ Yes | Apache 2.0 | ✅ Yes |
| **Uptrace** | T M L | SQL | ClickHouse | ✅ Yes | BSL 1.1 | ✅ Yes |
| **ClickHouse** | T L (raw) | SQL | Native columnar | ✅ Via exporter | Apache 2.0 | ✅ Yes |

### Metrics Backends

| System | Signals | Query language | Storage backend | OTLP native | License | Self-hostable |
|--------|---------|---------------|----------------|-------------|---------|--------------|
| **Prometheus** | M | PromQL | Local TSDB | ⚠️ Via remote_write | Apache 2.0 | ✅ Yes |
| **Thanos** | M | PromQL | Object storage (S3, GCS) | ⚠️ Via Prometheus | Apache 2.0 | ✅ Yes |
| **Grafana Mimir** | M | PromQL | Object storage | ⚠️ Via remote_write | AGPL 3.0 | ✅ Yes |
| **VictoriaMetrics** | M | MetricsQL (PromQL-compatible) | Native (efficient TSDB) | ⚠️ Via remote_write | Apache 2.0 (single) / Commercial (cluster) | ✅ Yes |
| **InfluxDB** | M | Flux / InfluxQL | Native TSM engine | ✅ Yes (v2+) | MIT (v2) / Commercial (cloud) | ✅ Yes |
| **TimescaleDB** | M | SQL (PostgreSQL) | PostgreSQL (time-series extension) | ⚠️ Via exporter | Apache 2.0 / Commercial | ✅ Yes |
| **OpenTSDB** | M | HTTP API | HBase | ❌ | LGPL 2.1 | ✅ Yes |

### Log Backends

| System | Signals | Query language | Storage backend | OTLP native | License | Self-hostable |
|--------|---------|---------------|----------------|-------------|---------|--------------|
| **Loki** | L | LogQL | Object storage (S3, GCS, Azure Blob) | ✅ Yes | AGPL 3.0 | ✅ Yes |
| **Elasticsearch** | T L | Lucene / EQL / KQL | Lucene shards | ✅ Via Elastic Agent | SSPL / Elastic License | ✅ Yes |
| **OpenSearch** | T L | Lucene / PPL / SQL | Lucene shards | ✅ Via exporter | Apache 2.0 | ✅ Yes |
| **ClickHouse** | T L (raw) | SQL | Native columnar | ✅ Via exporter | Apache 2.0 | ✅ Yes |
| **Quickwit** | L T | REST / SQL | Object storage (S3) | ✅ Yes | AGPL 3.0 | ✅ Yes |
| **Splunk** | L | SPL | Splunk proprietary | ✅ Via HEC / OTEL | Commercial | ✅ Yes (Enterprise) |

---

## Table 3 — Observability Platforms & SaaS

End-to-end platforms combining storage, visualisation, alerting, and often AI-driven analysis.

| Platform | Vendor | Signals | OTLP native | Query language(s) | Pricing model | Open source core |
|----------|--------|---------|-------------|-------------------|---------------|-----------------|
| **Grafana Cloud** | Grafana Labs | T M L Profiles | ✅ Yes | PromQL, LogQL, TraceQL | Usage-based (free tier) | ✅ Yes (Grafana OSS) |
| **Datadog** | Datadog | T M L RUM Profiles | ✅ Yes | Datadog query language | Host / container-based | ❌ No |
| **Honeycomb** | Honeycomb.io | T L | ✅ Yes | BubbleUp / HAPI | Event-based | ❌ No |
| **Dynatrace** | Dynatrace | T M L RUM AI | ✅ Yes | DQL (Dynatrace Query Language) | DPS (Davis® Data Units) | ❌ No |
| **New Relic** | New Relic | T M L RUM Profiles | ✅ Yes | NRQL | Data-ingest GB/month | ❌ No |
| **Elastic Observability** | Elastic | T M L RUM | ✅ Yes | KQL / EQL / Lucene | Host-based / ingest | ⚠️ SSPL |
| **Splunk Observability** | Cisco / Splunk | T M L | ✅ Yes | SPL / SignalFlow | Host-based | ❌ No |
| **AppDynamics** | Cisco | T M L | ✅ Yes (OTLP ingest) | AppDynamics query | Host-based | ❌ No |
| **Lightstep** | ServiceNow | T M | ✅ Yes | Lightstep query | Event-based | ❌ No |
| **Coralogix** | Coralogix | L M T | ✅ Yes | DataPrime / Lucene | Data-ingest GB/month | ❌ No |
| **Logz.io** | Logz.io | L T M | ✅ Yes | Lucene / Jaeger UI | Data-ingest GB/month | ❌ No |
| **Sumo Logic** | Sumo Logic | L M T | ✅ Yes | Sumo Logic query | Credits-based | ❌ No |
| **SigNoz** | SigNoz | T M L | ✅ Yes | SQL / PromQL | Usage-based (cloud) | ✅ Yes (Apache 2.0) |
| **Uptrace** | Uptrace | T M L | ✅ Yes | SQL | Self-host free / cloud paid | ⚠️ BSL 1.1 |
| **Last9** | Last9 | M L T | ✅ Yes | PromQL / LogQL | Usage-based | ❌ No |
| **Axiom** | Axiom | L T M | ✅ Yes | APL (Kusto-compatible) | Ingest + query | ❌ No |

---

## Choosing at a Glance

| Need | Recommended picks |
|------|------------------|
| Fully open source, self-hosted, all signals | SigNoz (all-in-one) or Grafana OSS stack (Tempo + Prometheus + Loki) |
| Best trace query UX | Honeycomb, Grafana Tempo + TraceQL, or SigNoz |
| Best metrics ecosystem | Prometheus + Grafana (OSS) or Grafana Cloud |
| Broadest integrations (enterprise) | Datadog or Dynatrace |
| Most affordable SaaS (high volume) | Grafana Cloud, Coralogix, or Axiom |
| Migration path (vendor → OSS) | OTEL Collector fan-out — run both simultaneously |

---

*See also: **[06 — Ecosystem](otel-ecosystem.md)** for stack recommendations by scale tier.*
