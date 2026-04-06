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
| **AWS X-Ray SDK** | Amazon | Java, Python, Node.js, Go, .NET, Ruby | T | ❌ No (X-Ray format; use OTEL SDK instead) | ✅ Partial | Apache 2.0 |
| **Apache SkyWalking Agent** | Apache | Java, .NET, Node.js, Go, Python, PHP | T M L | ❌ No (SW8 protocol) | ✅ Yes (Java bytecode) | Apache 2.0 |
| **Telegraf** | InfluxData | Plugin-based (300+ sources, any language) | M L | ✅ Yes (OTLP output plugin) | ❌ Config-based, not code | MIT |
| **Instana Agent** | IBM | Java, Node.js, Python, Ruby, .NET, Go, PHP | T M L | ✅ Yes | ✅ Yes (bytecode injection) | Commercial |
| **Grafana Beyla** | Grafana Labs | Go, Java, Python, Node.js, Ruby, .NET, Rust (eBPF) | T M | ✅ Yes | ✅ Zero-code (eBPF) | Apache 2.0 |
| **Glowroot** | Community | Java | T M | ✅ Yes | ✅ Yes (Java agent) | Apache 2.0 |

**Key notes:**
- **OTEL SDK** is the only vendor-neutral, actively maintained choice — prefer it for all new code
- **OpenTracing and OpenCensus** are archived; migrate via the provided shim bridge libraries
- **Vendor agents** (Datadog, Dynatrace, Instana) offer deep auto-instrumentation at the cost of lock-in
- **AWS X-Ray SDK** is a proprietary trap — use OTEL SDK and export to X-Ray via its OTLP endpoint instead
- **Grafana Beyla** (eBPF) is the fastest path to traces from services you cannot change; use alongside SDK, not instead of it
- **Telegraf** excels for infrastructure metrics collection alongside OTEL; its 300+ input plugins cover things the OTEL Collector cannot

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
| **Promtail** | Grafana Labs | Log collector for Loki (Grafana stack) | ❌ No (Loki push API only) | ~20 MB | ⚠️ Common in Grafana stacks | Apache 2.0 |
| **Grafana Alloy** | Grafana Labs | Next-gen collector (replaces Promtail + Agent) | ✅ Yes | ~100 MB | ❌ Needs deployment | Apache 2.0 |
| **Filebeat** | Elastic | Lightweight log shipper (Elastic Beats family) | ✅ Yes (OTLP output plugin) | ~50 MB | ❌ No | Elastic License |
| **Metricbeat / Beats** | Elastic | System + service metrics (Elastic Beats family) | ✅ Yes (OTLP output plugin) | ~50 MB | ❌ No | Elastic License |
| **AWS FireLens** | Amazon | Log router for Fargate/ECS (Fluent Bit wrapper) | ✅ Via Fluent Bit | Serverless | ✅ Fargate/ECS only | AWS managed |
| **rsyslog / syslog-ng** | Community | System log daemon and forwarder | ❌ No | Minimal (system daemon) | ✅ Linux systems | GPLv3 / LGPL |
| **Splunk Universal Forwarder** | Splunk | Lightweight log shipper to Splunk | ❌ No (Splunk protocol) | ~50 MB | ❌ No | Proprietary |

**When to use which:**

| Scenario | Best choice |
|----------|------------|
| Already have Fluent Bit on K8s nodes | Keep it; add OTLP output → OTEL Collector |
| Need lightweight log collection at edge / IoT | Fluent Bit |
| Running EFK (Fluentd) today, migrating incrementally | Fluentd → OTEL Collector bridge |
| Want a single tool for all three signals | OTEL Collector (filelog receiver) |
| High-throughput routing with complex transforms | Vector or Fluent Bit |
| 100% committed to Grafana/Loki stack | Promtail (simple) or Grafana Alloy (more capable) |
| Need metrics + logs + traces from one collector | Grafana Alloy (consolidates Promtail + Agent patterns) |
| Running on AWS Fargate or ECS | AWS FireLens (wraps Fluent Bit, AWS-managed config) |
| Deep Elastic/ELK investment | Filebeat (already there) or migrate to Fluent Bit over time |
| Legacy Linux systems (non-K8s, non-container) | rsyslog or syslog-ng for OS-level log forwarding |
| Heavy Splunk investment, not migrating yet | Splunk Universal Forwarder (inside Splunk ecosystem) |

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

## Table 2 — Backends & Storage Systems

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
| **AWS X-Ray** | Amazon | T | X-Ray Console + Analytics Engine | AWS Managed | ✅ Via OTLP endpoint | Commercial | ❌ Managed only |
| **Google Cloud Trace** | Google | T | Cloud Trace UI | GCP Managed | ✅ Via OTLP | Commercial | ❌ Managed only |
| **Azure Application Insights** | Microsoft | T M L | Azure portal / App Insights UI | Azure Managed | ✅ Via OTLP | Commercial | ❌ Managed only |
| **Apache SkyWalking OAP** | Apache | T M L | SkyWalking UI | ES / MySQL / TiDB / H2 | ❌ No (SW8 protocol) | Apache 2.0 | ✅ Yes |
| **Elastic APM** | Elastic | T M L | Kibana APM UI | Elasticsearch | ✅ Yes | SSPL / Elastic License | ✅ Yes |
| **Pinpoint** | NaverCorp | T | Pinpoint Web UI | HBase | ❌ No (Pinpoint protocol) | Apache 2.0 | ✅ Yes |

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
| **M3DB** | Uber / CNCF | M | M3Query (PromQL-compatible) | M3DB (distributed TSDB) | ⚠️ Via remote_write | Apache 2.0 | ✅ Yes |
| **Graphite** | Community | M | Graphite / Grafana | Whisper (local files) | ❌ No (Graphite protocol) | Apache 2.0 | ✅ Yes |
| **Amazon Managed Prometheus** | Amazon | M | PromQL | AWS Managed | ⚠️ Via remote_write | Commercial | ❌ Managed only |
| **Google Cloud Monitoring** | Google | M | MQL / PromQL | GCP Managed | ✅ Via OTLP | Commercial | ❌ Managed only |
| **Azure Monitor Metrics** | Microsoft | M | KQL (Metrics Explorer) | Azure Managed | ✅ Via OTLP | Commercial | ❌ Managed only |

### Log Backends

| System | Signals | Query language | Storage backend | OTLP native | License | Self-hostable |
|--------|---------|---------------|----------------|-------------|---------|--------------|
| **Loki** | L | LogQL | Object storage (S3, GCS, Azure Blob) | ✅ Yes | AGPL 3.0 | ✅ Yes |
| **Elasticsearch** | T L | Lucene / EQL / KQL | Lucene shards | ✅ Via Elastic Agent | SSPL / Elastic License | ✅ Yes |
| **OpenSearch** | T L | Lucene / PPL / SQL | Lucene shards | ✅ Via exporter | Apache 2.0 | ✅ Yes |
| **ClickHouse** | T L (raw) | SQL | Native columnar | ✅ Via exporter | Apache 2.0 | ✅ Yes |
| **AWS X-Ray** | Amazon | T | X-Ray Console + Analytics Engine | AWS Managed | ✅ Via OTLP endpoint | Commercial | ❌ Managed only |
| **Google Cloud Trace** | Google | T | Cloud Trace UI | GCP Managed | ✅ Via OTLP | Commercial | ❌ Managed only |
| **Azure Application Insights** | Microsoft | T M L | Azure portal / App Insights UI | Azure Managed | ✅ Via OTLP | Commercial | ❌ Managed only |
| **Apache SkyWalking OAP** | Apache | T M L | SkyWalking UI | ES / MySQL / TiDB / H2 | ❌ No (SW8 protocol) | Apache 2.0 | ✅ Yes |
| **Elastic APM** | Elastic | T M L | Kibana APM UI | Elasticsearch | ✅ Yes | SSPL / Elastic License | ✅ Yes |
| **Pinpoint** | NaverCorp | T | Pinpoint Web UI | HBase | ❌ No (Pinpoint protocol) | Apache 2.0 | ✅ Yes |
| **Quickwit** | L T | REST / SQL | Object storage (S3) | ✅ Yes | AGPL 3.0 | ✅ Yes |
| **Splunk** | L | SPL | Splunk proprietary | ✅ Via HEC / OTEL | Commercial | ✅ Yes (Enterprise) |
| **Graylog** | Graylog | L | GELF / Graylog Search | Elasticsearch / MongoDB | ✅ Yes (OTLP input) | SSPL / Commercial | ✅ Yes |
| **AWS CloudWatch Logs** | Amazon | L | CloudWatch Insights | AWS Managed | ✅ Via OTLP exporter | Commercial | ❌ Managed only |
| **Google Cloud Logging** | Google | L | Cloud Logging query | GCP Managed | ✅ Via OTLP exporter | Commercial | ❌ Managed only |
| **Azure Monitor Logs** | Microsoft | L | KQL (Log Analytics) | Azure Managed | ✅ Via OTLP exporter | Commercial | ❌ Managed only |
| **Seq** | Datalust | L | SQL-like (CLEF format) | SQLite / PostgreSQL | ✅ Yes (OTLP ingest) | Free / Commercial | ✅ Yes |
| **OpenObserve** | OpenObserve | T M L | SQL | S3 / local disk | ✅ Yes | Apache 2.0 | ✅ Yes |

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
| **Instana** | IBM | T M L | ✅ Yes | Instana analytics | Host-based | ❌ No |
| **Chronosphere** | Chronosphere | M T | ✅ Yes | PromQL / TraceQL | Resource-based (DPU) | ❌ No |
| **Better Stack** | Better Stack | L M T | ✅ Yes | SQL | Usage-based (free tier) | ❌ No |
| **AWS CloudWatch** | Amazon | T M L | ✅ Yes (OTLP ingest) | CW Insights / X-Ray Analytics | Pay-per-use | ❌ No |
| **Google Cloud Ops Suite** | Google | T M L | ✅ Yes | MQL / Cloud Logging query | Pay-per-use | ❌ No |
| **Azure Monitor** | Microsoft | T M L | ✅ Yes | KQL (Log Analytics) | Pay-per-use | ❌ No |
| **Groundcover** | groundcover | T M L | ✅ Yes (eBPF) | PromQL / LogQL | Host-based | ❌ No |
| **OpenObserve** | OpenObserve | T M L | ✅ Yes | SQL | Usage-based (free self-host) | ✅ Yes (Apache 2.0) |
| **Observe Inc.** | Observe | L T M | ✅ Yes | OPAL | Usage-based | ❌ No |

---

## Table 4 — eBPF & Agent-less Observability

A growing category of tools that provide observability **without code changes**, instrumenting at the Linux kernel level via eBPF. Use these to gain quick baseline visibility into uninstrumented services — not as a permanent replacement for SDK instrumentation.

| Tool | Vendor | Signals | How it works | K8s native | OTLP output | License |
|------|--------|---------|-------------|-----------|-------------|---------|
| **Grafana Beyla** | Grafana Labs | T M | eBPF probes on HTTP, gRPC, DB calls | ✅ Yes | ✅ Yes | Apache 2.0 |
| **Pixie** | New Relic / CNCF | T M L | eBPF in-cluster agents streaming to PEM collectors | ✅ K8s only | ✅ Yes | Apache 2.0 |
| **Cilium Hubble** | Isovalent / CNCF | Network flows, T | eBPF network observability (requires Cilium CNI) | ✅ Yes | ✅ Yes | Apache 2.0 |
| **Coroot** | Coroot | T M L | eBPF service map + golden signals | ✅ Yes | ✅ Yes | Apache 2.0 |
| **Groundcover** | groundcover | T M L | eBPF full-stack — zero agent install, SaaS-managed | ✅ Yes | ✅ Yes | Commercial |
| **Odigos** | keyval | T M L | Auto-injects OTEL SDK or eBPF per service language | ✅ Yes | ✅ Yes | Apache 2.0 |
| **Parca** | Polar Signals / CNCF | Profiles | eBPF continuous CPU/memory profiling | ✅ Yes | ⚠️ Partial | Apache 2.0 |
| **Tetragon** | Isovalent / CNCF | Security events | Kernel-level eBPF for security + audit tracing | ✅ Yes | ⚠️ Partial | Apache 2.0 |
| **Falco** | Falco Security / CNCF | Security events, L | eBPF/kernel module threat detection and alerting | ✅ Yes | ✅ Yes | Apache 2.0 |

**eBPF vs. SDK — when to use each:**

| Factor | eBPF (Beyla, Pixie, Coroot) | OTEL SDK |
|--------|----------------------------|---------|
| Code changes required | ❌ None | ✅ Yes |
| Custom business spans | ❌ No | ✅ Yes |
| Trace propagation through Kafka / RabbitMQ | ❌ No | ✅ Yes |
| Trace depth | Entry/exit points only | Full span tree |
| Performance overhead | < 1% | ~2–5% |
| Best use | Quick baseline on uninstrumented services | All new services; anywhere deep visibility matters |

**Recommendation:** eBPF tools are complementary to, not a replacement for, SDK instrumentation. Use Beyla or Coroot to gain immediate visibility, then migrate to the OTEL SDK for services where you need custom spans, business context, and async message propagation.

---

## Choosing at a Glance

| Need | Recommended picks |
|------|------------------|
| Fully open source, self-hosted, all signals | SigNoz (all-in-one) or Grafana OSS (Tempo + Prometheus + Loki) |
| Best trace investigation UX | Honeycomb (BubbleUp), Grafana Tempo + TraceQL, or SigNoz |
| Best metrics ecosystem | Prometheus + Grafana (OSS) or Grafana Cloud |
| Broadest integrations (enterprise) | Datadog or Dynatrace |
| Most affordable SaaS at high volume | Grafana Cloud, Coralogix, or Axiom |
| Migration from vendor → OSS | OTEL Collector fan-out — run both exporters simultaneously |
| Already on AWS, minimal ops | AWS CloudWatch + X-Ray (convenient; expensive at scale) |
| Already on GCP | Google Cloud Ops Suite (Trace + Monitoring + Logging) |
| Already on Azure | Azure Monitor with KQL (Log Analytics) |
| Zero-code visibility for uninstrumented services | Grafana Beyla (web services) or Pixie (K8s debugging) |
| Log-heavy workload, cost-sensitive | Loki (OSS) or Axiom (SaaS, generous free tier) |
| High-cardinality metrics at high scale | VictoriaMetrics (3–4× more efficient than Prometheus) |
| Java services — fastest instrumentation win | OTEL Java Agent (`-javaagent`) — zero code changes required |
| Existing Fluent Bit on K8s nodes | Keep it; add OTLP output plugin → OTEL Collector |
| EFK (Fluentd) stack migration | Fluentd → OTEL Collector → Loki/SigNoz (incremental) |
| Maximum cost reduction vs. Datadog in non-prod | SigNoz + OTEL Collector — 80–95% cost reduction |
| Network-level K8s observability | Cilium + Hubble (eBPF, zero code, full service topology) |
| Continuous profiling (4th signal) | Pyroscope (SDK, language-native) or Parca (eBPF, zero-code) |
| Developer-friendly structured log search | Seq (SQL-like queries, runs locally, free tier) |
| Replace Elasticsearch complexity | Loki (simpler ops) or OpenObserve (SQL + S3, 100× cheaper) |
| Legacy Graphite metrics | Graphite exporter → Prometheus bridge; Graphite is end-of-life |
| Legacy OpenTSDB | Migrate to VictoriaMetrics — PromQL-compatible, far lighter |

---

## Informed Opinions

Direct assessments based on technical characteristics, not marketing.

### Instrumentation SDKs & Agents

- **Use OTEL SDK for all new code.** It is the only choice that keeps your backend options open. All other SDKs produce vendor-specific formats.
- **AWS X-Ray SDK is a trap.** Proprietary wire format, no OTLP output. Use the OTEL SDK and configure the OTEL Collector `awsxray` exporter if you need to send to X-Ray.
- **Dynatrace OneAgent and Instana Agent** are good for zero-code coverage of large inherited codebases when vendor lock-in is acceptable. For greenfield, OTEL SDK is better.
- **Grafana Beyla** is the most practical way to get traces from uninstrumented services in K8s without a dev cycle. Use as a bridge while migrating to SDK instrumentation.
- **Telegraf** is underrated for infrastructure metrics — 300+ input plugins cover things the OTEL Collector cannot. Run Telegraf alongside OTEL Collector: Telegraf → OTEL Collector → backend.

### Log Shippers

- **Fluent Bit is the clear winner** in K8s: smallest memory footprint (~10 MB), fastest processing, pre-installed on EKS/GKE/AKS. Default choice for any new K8s deployment.
- **Logstash is being retired** in most modern stacks. The OTEL Collector's `transform` and `filter` processors now cover most of its functionality at much lower overhead.
- **Promtail locks you to Loki.** Grafana Alloy is its more capable successor and supports multiple backends. For multi-backend routing, use Fluent Bit + OTEL Collector.
- **The Elastic Beats family** (Filebeat, Metricbeat) is only worth the overhead if you're already deep in Elastic. Otherwise, Fluent Bit + OTEL Collector covers the same ground more flexibly.
- **Grafana Alloy** is worth evaluating for a fresh Grafana-based stack — it consolidates Promtail, Telegraf, and OTEL collection into one binary.

### Backends & Storage

- **Prometheus is the standard for metrics.** Start here. Add Thanos or Grafana Mimir for multi-region HA. VictoriaMetrics is 3–4× more efficient at ingestion for high-volume environments.
- **Loki is cost-effective** (object storage, no index) but LogQL has a steeper learning curve than SQL or KQL. Best when Grafana is already your dashboarding layer.
- **ClickHouse is the best query engine** for logs and traces at scale. SigNoz and Uptrace both use it — which explains their query speed advantage over Elasticsearch-backed alternatives.
- **Jaeger is solid for dev/test** but Grafana Tempo (pure object storage, no index) is cheaper to operate in production. Prefer Tempo for new deployments.
- **Elasticsearch is powerful but expensive to operate.** OpenSearch is a drop-in OSS alternative. Both are being displaced by ClickHouse-backed tools in modern stacks.
- **Graphite is end-of-life.** Use the Graphite exporter for Prometheus as a migration bridge. No new projects should use it.
- **OpenTSDB has a heavyweight HBase dependency** — migrate to VictoriaMetrics or Prometheus if you're still running it.
- **Managed cloud log services** (CloudWatch Logs, GCP Logging, Azure Monitor Logs) are convenient but expensive at volume. Build a hybrid approach for high-volume workloads.
- **OpenObserve** (Apache 2.0) is impressive: S3-backed, OTLP-native, SQL queries, claims 100× cheaper per GB than Datadog. Worth evaluating for log-heavy workloads.

### Platforms

- **Datadog is the most complete platform** but pricing surprises teams — host + container + custom metrics + log ingestion add up fast. Audit your bill before committing.
- **Honeycomb has the best trace investigation UX** for high-cardinality data. BubbleUp makes root-cause analysis genuinely faster. Worth the premium if traces are your primary debug tool.
- **SigNoz is the best fully open-source Datadog alternative.** OTLP-native, ClickHouse-backed, active development, good UI. Top recommendation for non-production environments.
- **Chronosphere** was built for Kubernetes at Uber scale. Evaluate it if you're running large K8s clusters and struggling with Prometheus cardinality explosion.
- **AWS CloudWatch** costs spiral quickly. Custom metric charges, log ingestion fees, and alarm charges compound. At scale, a hybrid approach (Prometheus for metrics, selective CloudWatch for alerting) is more economical.
- **Dynatrace and AppDynamics** are enterprise APM incumbents with deep auto-discovery. Both support OTLP ingest, so OTEL SDK can coexist. Migration path exists but is not trivial.
- **Avoid rebuilding ELK** (Elasticsearch + Logstash + Kibana) for new deployments — high operational complexity and cost. Use Loki + Grafana or SigNoz instead.

### eBPF Tools

- **eBPF tools complement, not replace, SDK instrumentation.** They cannot add custom business spans or propagate trace context through Kafka or RabbitMQ.
- **Beyla or Coroot** are the fastest path to "some visibility" for uninstrumented services. Then replace with OTEL SDK where deep observability matters.
- **Cilium Hubble** is for network-level observability (service-to-service topology, L4/L7 flows, DNS failures) — not APM. Excellent complement to trace-level tooling.
- **Odigos** is uniquely useful for polyglot teams: it automatically selects and injects the right OTEL SDK or eBPF agent per service, removing the per-team instrumentation burden.

---

*See also: **[06 — Ecosystem](otel-ecosystem.md)** for stack recommendations by scale tier.*

[← Back to Guide Index](otel-overview.md)