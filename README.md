# OpenTelemetry Observability Systems

A practical research and reference guide for implementing OpenTelemetry (OTEL) across
distributed systems — from core concepts to real-world migration strategies.

Built as a growing knowledge base with visual HTML guides and companion markdown references,
covering instrumentation, collection, sampling, distributed tracing, and backend selection.

---

## 📖 Documentation Site

**[View the live guide →](https://brianmehrman.github.io/OTEL-observability-systems/)**

The site is served from the `docs/` folder and includes 13 guides organized into two tracks:
a core learning track (Sections 01–09) and a set of applied migration guides.

---

## What's Covered

### Core Sections (01–09)

| # | Guide | Key Question Answered |
|---|-------|-----------------------|
| 01 | [Overview](docs/otel-overview.html) | What is OTEL and how do the pieces fit together? |
| 02 | [The Three Signals](docs/otel-signals.html) | When should I use traces vs. metrics vs. logs? |
| 03 | [The Collector](docs/otel-collector.html) | How do I configure the Collector pipeline? |
| 04 | [Sampling](docs/otel-sampling.html) | How do I control data volume without losing signal? |
| 05 | [Distributed Tracing](docs/otel-distributed.html) | How does trace context flow across service boundaries? |
| 06 | [Ecosystem](docs/otel-ecosystem.html) | Which backends and tools should I use at my scale? |
| 07 | [Language SDKs](docs/otel-sdks.html) | How do I set up OTEL in Python, Node.js, Go, or Java? |
| 08 | [Kubernetes](docs/otel-kubernetes.html) | How do I deploy OTEL in a Kubernetes cluster? |
| 09 | [Tools Reference](docs/otel-tools.html) | What libraries, backends, and platforms exist? |

### Applied Guides

| Guide | Description |
|-------|-------------|
| [Rails Stack Analysis](docs/otel-rails-stack.html) | Analyzes a Rails → Logstash → Elasticsearch → Datadog pipeline and proposes three replacement options (Grafana OSS, SigNoz, Hybrid) |
| [Migration Plan](docs/otel-migration-plan.html) | Phase-by-phase visual migration timeline with parallel-run strategy and rollback guidance |
| [Non-Prod Options](docs/otel-nonprod-options.html) | Compares five dev/staging backends (Grafana LGTM, SigNoz, Uptrace, Jaeger) with RAM usage and config examples |
| [Distributed System Migration](docs/otel-distributed-migration.html) | Full migration guide for a polyglot distributed system: Rails, Sidekiq, Java microservices, Node.js, RabbitMQ, Kafka, and AWS SNS/SQS |

---

## Guiding Principles

- **Vendor-neutral** — All examples use OTEL-standard config; backend choices are illustrative
- **Scale-aware** — Every section covers both small-app and large distributed system variants
- **Diagrams first** — Visual HTML guides are the primary reference; markdown reinforces them
- **Opinionated where it helps** — Where there's a clear best practice, it's called out directly

---
