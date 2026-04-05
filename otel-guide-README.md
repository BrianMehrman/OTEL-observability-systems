# OpenTelemetry Architecture Guide

A practical, visual reference for understanding and implementing OTEL across applications of any scale.
Built as a growing knowledge base — each section contains a visual HTML guide and a companion markdown reference.

---

## Structure

```
otel-guide/
├── README.md                          ← you are here
│
├── 01-overview/
│   ├── otel-overview.html             ✅ Visual architecture map
│   └── otel-overview.md               ✅ Written reference
│
├── 02-signals/
│   ├── otel-signals.html              ✅ Visual signal deep dive
│   └── otel-signals.md                ✅ Written reference
│
├── 03-collector/
│   ├── otel-collector.html            ✅ Visual pipeline & config guide
│   └── otel-collector.md              ✅ Written reference
│
├── 04-sampling/
│   ├── otel-sampling.html             ✅ Visual sampling guide
│   └── otel-sampling.md               ✅ Written reference
│
├── 05-distributed/
│   ├── otel-distributed.html          ✅ Visual context propagation guide
│   └── otel-distributed.md            ✅ Written reference
│
├── 06-ecosystem/
│   ├── otel-ecosystem.html            ✅ Visual backend & platform guide
│   └── otel-ecosystem.md              ✅ Written reference
│
├── 07-sdks/
│   ├── otel-sdks.html                 ✅ Visual language SDK guide
│   └── otel-sdks.md                   ✅ Written reference
│
├── 08-kubernetes/
│   ├── otel-kubernetes.html           ✅ Visual Kubernetes deployment guide
│   └── otel-kubernetes.md             ✅ Written reference
│
├── 09-tools/
│   ├── otel-tools.html                ✅ Visual tools reference tables
│   └── otel-tools.md                  ✅ Written reference
│
├── rails-stack/
│   ├── otel-rails-stack.html          ✅ Visual Rails stack analysis (current + 3 options)
│   ├── otel-rails-stack.md            ✅ Written reference
│   ├── otel-migration-plan.html       ✅ Visual phase-by-phase migration timeline
│   └── otel-migration-plan.md         ✅ Written migration plan
│
├── distributed-migration/
│   ├── otel-distributed-migration.html  ✅ Visual polyglot distributed system migration guide
│   └── otel-distributed-migration.md   ✅ Written reference (Rails + Sidekiq + Java + Node.js)
```

---

## Sections at a Glance

| # | Topic | Status | Key Question Answered |
|---|-------|--------|-----------------------|
| 01 | Overview | ✅ Done | What is OTEL and how do the pieces fit together? |
| 02 | The Three Signals | ✅ Done | What should I instrument and when? |
| 03 | The Collector | ✅ Done | How do I configure the Collector for my scale? |
| 04 | Sampling | ✅ Done | How do I control data volume without losing signal? |
| 05 | Distributed Tracing | ✅ Done | How does context flow across service boundaries? |
| 06 | Ecosystem | ✅ Done | Which backends and tools should I use at my scale? |
| 07 | Language SDKs | ✅ Done | How do I set up OTEL in Python / Node / Go / Java? |
| 08 | Kubernetes | ✅ Done | How do I deploy OTEL in a Kubernetes cluster? |
| 09 | Tools Reference | ✅ Done | What libraries, backends, and platforms exist in the ecosystem? |
| — | Rails Stack Analysis | ✅ Done | How do I modernize my Rails log pipeline? |
| — | Migration Plan | ✅ Done | How do I migrate from Logstash → Elasticsearch → Datadog step by step? |
| — | Distributed System Migration | ✅ Done | How do I migrate a polyglot distributed system (Rails + Java + Node.js + messaging) to OTEL? |

---

## Guiding Principles

- **Vendor neutral** — all examples use OTEL-standard config, backend choices are illustrative
- **Scale-aware** — every section covers small-app and large-system variants
- **Diagrams first** — written docs reinforce the visuals, not the other way around
- **Opinionated where it helps** — where there's a clear best practice, we say so

---

> **Maintained by:** Architecture team
> **Scope:** OTEL instrumentation, collection, and observability pipeline design
