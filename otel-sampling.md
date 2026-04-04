# 04 — Sampling: Controlling Volume Without Losing Signal

**Visual guide:** `otel-sampling.html`
**Covers:** Why sampling is necessary, head vs. tail approaches, all six tail sampling policy types, a production-ready config, and the three most common pitfalls.

---

## Why Sampling Exists

At 100% collection, trace volume scales linearly with traffic:

| Traffic | Traces/day (100%) |
|---------|------------------|
| 100 req/s | ~8.6M |
| 1,000 req/s | ~86M |
| 10,000 req/s | ~864M |

Beyond a few hundred requests per second, storing every trace becomes expensive and mostly redundant — a healthy 200ms request at 10:00:01 is not meaningfully different from the same request at 10:00:02. Sampling is the strategy for keeping the traces that matter and discarding the ones that don't.

**The goal:** 100% coverage of errors and slow requests. A representative sample of healthy traffic.

---

## Head-Based Sampling

The decision is made at the **start of the request**, before any spans are created.

### How it works

1. The root service SDK generates a `trace_id`
2. A random sampling decision is made based on a configured percentage
3. The decision (sample / do not sample) is encoded in the `tracestate` header
4. All downstream services read this header and honour the same decision
5. Either the whole trace is recorded, or none of it is

### Configuration (in the SDK)

```python
# Python example
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased

sampler = TraceIdRatioBased(0.1)  # 10% of traces
```

```yaml
# Or via OTEL environment variable
OTEL_TRACES_SAMPLER=traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1
```

### Strengths and weaknesses

| | |
|-|-|
| ✓ Zero latency overhead | ✗ Blind to outcomes |
| ✓ No Collector required | ✗ May drop error traces |
| ✓ Stateless | ✗ May drop slow requests |
| ✓ Simple to configure | ✗ No rule-based targeting |

**Use when:** Traffic is low to moderate, simplicity is valued, and missing occasional error traces is acceptable.

---

## Tail-Based Sampling

The decision is made **after the trace completes**, in the Collector.

### How it works

1. All spans flow to the gateway Collector tier
2. The Collector groups spans by `trace_id` and buffers them in memory
3. After `decision_wait` (or when the trace appears complete), policies are evaluated
4. If any policy matches, the trace is kept; otherwise it is dropped

### Why this matters

With tail-based sampling, you can write rules like:
- **Always keep** any trace with an error status
- **Always keep** any trace that took longer than 1 second
- **Always keep** any trace from an enterprise tenant
- **Keep 5%** of everything else

This gives you guaranteed coverage of the traces that matter, with controlled cost on the rest.

### Requirement: stateful Collector

Tail-based sampling requires all spans for a given trace to arrive at the **same Collector instance**. Use consistent hashing on `trace_id` when load-balancing across gateway replicas.

---

## Tail Sampling Policy Types

### status_code
Keep all traces where any span has an error status.
```yaml
- name: errors-always
  type: status_code
  status_code:
    status_codes: [ERROR]
```

### latency
Keep all traces where the root span exceeded a threshold.
```yaml
- name: slow-always
  type: latency
  latency:
    threshold_ms: 1000
```

### probabilistic
Randomly keep a percentage of traces. For baseline visibility into healthy traffic.
```yaml
- name: baseline
  type: probabilistic
  probabilistic:
    sampling_percentage: 5
```

### string_attribute
Keep traces matching a specific attribute value. Useful for business-critical paths or specific tenants.
```yaml
- name: checkout-always
  type: string_attribute
  string_attribute:
    key: http.route
    values: ["/checkout", "/payment"]
```

### rate_limiting
Hard cap on spans per second. Acts as a budget ceiling regardless of other policies.
```yaml
- name: hard-cap
  type: rate_limiting
  rate_limiting:
    spans_per_second: 500
```

### composite
Combines policies with per-policy rate allocations. Most precise control.
```yaml
- name: combined
  type: composite
  composite:
    max_total_spans_per_second: 500
    policy_order:
      - errors-always
      - slow-always
      - baseline
```

---

## Production-Ready Config

A layered policy set for a high-volume service. Policies are evaluated in order — first match wins.

```yaml
tail_sampling:
  decision_wait: 10s       # wait for all spans to arrive
  num_traces: 50000        # max in-memory buffer
  expected_new_traces_per_sec: 1000

  policies:

    # Layer 1: Always keep — no exceptions
    - name: errors-always
      type: status_code
      status_code: {status_codes: [ERROR]}

    - name: slow-always
      type: latency
      latency: {threshold_ms: 2000}

    # Layer 2: Business-critical paths
    - name: checkout-always
      type: string_attribute
      string_attribute:
        key: http.route
        values: ["/checkout", "/payment", "/order"]

    - name: enterprise-always
      type: string_attribute
      string_attribute:
        key: tenant.tier
        values: [enterprise]

    # Layer 3: Baseline sample of healthy traffic
    - name: healthy-sample
      type: probabilistic
      probabilistic: {sampling_percentage: 5}

    # Layer 4: Hard cap — spending protection
    - name: rate-limiter
      type: rate_limiting
      rate_limiting: {spans_per_second: 1000}
```

---

## Three Common Pitfalls

### 1. Spans split across Collector instances

**Problem:** Tail sampling is stateful. Spans from the same trace hitting different gateway replicas means the policy sees an incomplete trace.

**Fix:** Configure your load balancer to route on `trace_id` using consistent hashing. In Kubernetes, the `loadbalancingexporter` in the agent tier can do this automatically.

### 2. decision_wait too short

**Problem:** If your p99 trace duration is 15s but `decision_wait` is 10s, the Collector decides before all spans arrive. It may keep or drop a trace based on incomplete information.

**Fix:** Set `decision_wait` to at least 2× your p99 trace duration. Monitor `otelcol_processor_tail_sampling_late_span_age` to detect this problem.

### 3. Sampling metrics

**Problem:** Applying probabilistic sampling to a metrics pipeline corrupts your aggregates. A counter that only captures 10% of increments will be off by a factor of 10.

**Fix:** Never apply sampling processors to the metrics pipeline. Metrics always flow at 100%.

---

## What's Next

- **[05 — Distributed Tracing]** — How trace context crosses service boundaries, W3C propagation, and correlating traces across languages and frameworks
