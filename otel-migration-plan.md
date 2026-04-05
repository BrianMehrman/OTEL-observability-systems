# Migration Plan: Logstash → Elasticsearch → Datadog to OTEL-Native

**Visual guide:** `otel-migration-plan.html`
**Context:** Step-by-step migration from the current Rails log pipeline to one of three OTEL-native options. The plan uses a parallel-run strategy — no hard cutover, no downtime.

---

## The Problem with a Hard Cutover

Switching pipelines all at once is risky:
- Log gaps if the new stack has a configuration error
- No baseline to compare new stack against old
- Developers are unfamiliar with the new UI before it's the only option

The strategy below runs old and new pipelines simultaneously across three phases before decommissioning anything.

---

## Phase 0 — Current State

```
[ Rails App ] ──writes──▶ [ Log File ] ──tails──▶ [ Logstash ]
                                                        │
                                                        ▼
                                               [ Elasticsearch ]
                                                        │
                                                        ▼
                                                  [ Datadog ]
```

**Signal coverage:** Logs only. No traces, no application metrics.
**Pipeline RAM:** 5–11 GB (Logstash + Elasticsearch alone).
**Components running:** Rails, Log File, Logstash, Elasticsearch, Datadog.

---

## Phase 1 — Install OTEL Collector (No Rails Changes Required)

Add the OTEL Collector alongside the existing pipeline. Configure its `filelog` receiver to tail the same log file Logstash is already reading. The old pipeline continues running unchanged.

```
[ Rails App ] ──writes──▶ [ Log File ] ──tails──▶ [ Logstash ]   ← still running
                               │                         │
                               │ (also tailed by)        ▼
                               ▼                  [ Elasticsearch ]
                        [ OTEL Collector ]  NEW         │
                               │                        ▼
                               ▼                  [ Datadog ]      ← still running
                  [ New Backend ] NEW
                  (Loki+Grafana, SigNoz, or Datadog via OTEL Collector)
```

**Goal:** Validate that logs arrive correctly in the new stack before touching anything else.
**Risk:** Zero — the old pipeline is entirely untouched.
**Duration:** 1–2 days to deploy and verify.

### OTEL Collector filelog receiver config (Phase 1)

```yaml
receivers:
  filelog:
    include: [/var/log/rails/production.log]
    operators:
      - type: json_parser
        timestamp:
          parse_from: attributes.time
          layout: "%Y-%m-%dT%H:%M:%S.%LZ"
```

---

## Phase 2 — Add Rails OTEL Instrumentation

Add the `opentelemetry-ruby` gems to Rails. Rails now sends OTLP telemetry (traces + metrics + logs) directly to the OTEL Collector. The old file-based log pipeline continues running in parallel for comparison.

```
[ Rails App ] ──OTLP──▶ [ OTEL Collector ]   ← NEW path (traces + metrics + logs)
      │                         │
      │ (still writes)          ▼
      ▼                 [ New Backend ]
[ Log File ] ──tails──▶ [ Logstash ]          ← still running (comparison baseline)
                               │
                               ▼
                        [ Elasticsearch ]
                               │
                               ▼
                          [ Datadog ]
```

**Goal:** Verify traces and metrics appear for the first time. Compare logs in old vs new backend.
**What's new:** Full request traces, SQL query spans, Redis spans, outbound HTTP spans. Application metrics (request rate, error rate, latency histograms).
**Duration:** 1 week minimum — enough time to see normal traffic patterns.

### Gemfile additions

```ruby
gem 'opentelemetry-sdk'
gem 'opentelemetry-exporter-otlp'
gem 'opentelemetry-instrumentation-all'
```

---

## Phase 3 — Decommission (Choose Your Path)

After 30 days of parity, shut down the legacy components. Which components you remove depends on which option you chose.

---

### Path A/B — Full Replacement (Grafana OSS or SigNoz)

Remove Logstash, Elasticsearch, and Datadog entirely. The OTEL Collector is the only pipeline.

```
[ Rails App ] ──OTLP──▶ [ OTEL Collector ]
                                │
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
             [ Loki ]    [ Prometheus ]    [ Tempo ]    ← Option A
                └───────────────┴───────────────┘
                                │
                                ▼
                          [ Grafana ]

          — OR —

[ Rails App ] ──OTLP──▶ [ SigNoz Collector ] ──▶ [ ClickHouse ] ──▶ [ SigNoz UI ]   ← Option B

REMOVED: Logstash, Elasticsearch, Datadog Agent
RECOVERED: 5–11 GB RAM, Datadog subscription cost
```

---

### Path C — Hybrid (Datadog for Production, Self-hosted for Non-prod)

Remove Logstash and Elasticsearch from all environments. Keep Datadog for production but replace it with a self-hosted stack in dev/staging.

```
PRODUCTION:
[ Rails App ] ──OTLP──▶ [ OTEL Collector ] ──datadog exporter──▶ [ Datadog ]

STAGING / DEV:
[ Rails App ] ──OTLP──▶ [ OTEL Collector ] ──otlp exporter──▶ [ SigNoz ]

REMOVED: Logstash, Elasticsearch (all environments)
         Datadog (non-production environments only)
RECOVERED: 5–11 GB RAM, 2–3 Datadog host fees per month
```

---

## Phase-by-Phase Component Status Summary

| Component | Phase 0 | Phase 1 | Phase 2 | Phase 3 (A/B) | Phase 3 (C) |
|-----------|---------|---------|---------|----------------|-------------|
| Rails App | ✅ Running | ✅ Running | ✅ Running + OTEL gems | ✅ Running | ✅ Running |
| Log File | ✅ Active | ✅ Active | ✅ Active | ❌ Removed (optional) | ❌ Removed |
| Logstash | ✅ Running | ✅ Running | ⚠️ Parallel (retiring) | ❌ Removed | ❌ Removed |
| Elasticsearch | ✅ Running | ✅ Running | ⚠️ Parallel (retiring) | ❌ Removed | ❌ Removed |
| Datadog | ✅ Running | ✅ Running | ⚠️ Parallel (retiring) | ❌ Removed | ✅ Production only |
| OTEL Collector | — | 🆕 New | 🆕 New | ✅ Running | ✅ Running |
| New Backend | — | 🆕 New | 🆕 New (gains traces) | ✅ Primary | ✅ Non-prod primary |

---

## Rollback Plan

At each phase, rollback is safe:

- **Phase 1 rollback:** Remove OTEL Collector containers. Old pipeline never changed.
- **Phase 2 rollback:** Remove OTEL gems from Gemfile, redeploy. Logstash still running.
- **Phase 3 rollback:** Restart Logstash and Elasticsearch containers. OTEL Collector continues.

There is no point of no return until you cancel the Datadog subscription or delete the Elasticsearch volume.

---

## Timeline Estimate

| Phase | Duration | Work required |
|-------|----------|---------------|
| Phase 1 | 1–3 days | Deploy OTEL Collector + filelog config |
| Phase 2 | 1 week | Add 3 gems, write initializer, validate traces |
| Phase 3 (parity check) | 30 days | Compare old vs new, build dashboards and alerts |
| Phase 3 (decommission) | 1 day | Remove containers, cancel subscriptions |
| **Total** | **~6 weeks** | Safe, parallel migration |
