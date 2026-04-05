# 07 — Language SDKs: Python, Node.js, Go & Java

**Visual guide:** `otel-sdks.html`
**Covers:** How OTEL SDKs are structured, zero-code auto-instrumentation, manual span creation, resource configuration, and complete setup guides for Python, Node.js, Go, and Java.

---

## SDK Architecture (Shared Concepts)

Every OTEL SDK — regardless of language — implements the same architecture:

```
[ Your Code ]
     │
     ▼
[ Tracer / Meter / Logger ]    ← API layer (stable, instrumentable)
     │
     ▼
[ SDK ]                         ← implementation: samplers, processors, exporters
     │
     ▼
[ Exporter ]                    ← sends OTLP to Collector or backend
```

The **API** and **SDK** are separate packages. Libraries instrument against the API (stable). Your application wires up the SDK (the implementation). This means a library can emit spans even if no SDK is configured — the calls are no-ops.

### Key Configuration Concepts

All SDKs share these configuration points, many of which can be set via environment variables:

| Concept | Purpose | Environment variable |
|---------|---------|---------------------|
| **Resource** | Describes the service (name, version, environment) | `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES` |
| **Exporter** | Where to send data | `OTEL_EXPORTER_OTLP_ENDPOINT` |
| **Sampler** | What percentage of traces to record | `OTEL_TRACES_SAMPLER`, `OTEL_TRACES_SAMPLER_ARG` |
| **Propagator** | How context crosses process boundaries | `OTEL_PROPAGATORS` |

### Recommended environment variables (all languages)

```bash
OTEL_SERVICE_NAME=my-service
OTEL_SERVICE_VERSION=1.4.2
OTEL_DEPLOYMENT_ENVIRONMENT=production
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
OTEL_PROPAGATORS=tracecontext,baggage
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1
```

Setting these via environment means you can change sampling rate, endpoint, or service version without touching code — useful for Kubernetes deployments and CI/CD pipelines.

---

## Python

### Install

```bash
pip install opentelemetry-sdk \
            opentelemetry-exporter-otlp-proto-grpc \
            opentelemetry-instrumentation-fastapi \
            opentelemetry-instrumentation-sqlalchemy \
            opentelemetry-instrumentation-httpx
```

### Zero-Code Auto-Instrumentation

The `opentelemetry-instrument` command wraps any Python process and instruments it automatically — no code changes required.

```bash
pip install opentelemetry-distro
opentelemetry-bootstrap --action=install    # installs instrumentation for detected packages

opentelemetry-instrument \
  --service-name my-service \
  --exporter-otlp-endpoint http://localhost:4317 \
  python app.py
```

Auto-instrumented frameworks include: FastAPI, Flask, Django, SQLAlchemy, Redis, Celery, httpx, requests, gRPC, and more.

### SDK Setup (code)

Use this when you need control over configuration, or when running in environments where the CLI wrapper is not practical.

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource, SERVICE_NAME
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

# Resource identifies this service
resource = Resource.create({
    SERVICE_NAME: "order-service",
    "service.version": "2.1.0",
    "deployment.environment": "production",
})

# Wire up the provider, processor, and exporter
provider = TracerProvider(resource=resource)
exporter = OTLPSpanExporter(endpoint="http://localhost:4317", insecure=True)
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)

# Acquire a tracer — one per module is the standard pattern
tracer = trace.get_tracer(__name__)
```

### Manual Spans

```python
with tracer.start_as_current_span("process-order") as span:
    span.set_attribute("order.id", order_id)
    span.set_attribute("order.total_usd", total)

    try:
        result = process(order)
        span.set_attribute("order.status", result.status)
    except PaymentError as e:
        span.set_status(StatusCode.ERROR, str(e))
        span.record_exception(e)    # captures stack trace as a span event
        raise
```

### Metrics

```python
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter

reader = PeriodicExportingMetricReader(OTLPMetricExporter(endpoint="http://localhost:4317"))
metrics.set_meter_provider(MeterProvider(metric_readers=[reader], resource=resource))
meter = metrics.get_meter(__name__)

# Counter
request_counter = meter.create_counter("http.server.request.count")
request_counter.add(1, {"http.method": "POST", "http.status_code": 200})

# Histogram
latency = meter.create_histogram("http.server.request.duration", unit="ms")
latency.record(42.5, {"http.route": "/checkout"})
```

---

## Node.js

### Install

```bash
npm install @opentelemetry/sdk-node \
            @opentelemetry/auto-instrumentations-node \
            @opentelemetry/exporter-trace-otlp-grpc \
            @opentelemetry/exporter-metrics-otlp-grpc
```

### Zero-Code Auto-Instrumentation

Node.js auto-instrumentation works by requiring a setup file before your application code loads. This must run first — before any other imports.

```bash
# Start with env-based config (no code changes)
OTEL_SERVICE_NAME=my-service \
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
node --require @opentelemetry/auto-instrumentations-node/register app.js
```

Auto-instrumented libraries include: Express, Fastify, Koa, http/https, gRPC, MySQL, PostgreSQL, MongoDB, Redis, AWS SDK, and more.

### SDK Setup (CommonJS)

Create `tracing.js` and require it first — before any other imports in your entry point.

```javascript
// tracing.js
'use strict';

const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { OTLPMetricExporter } = require('@opentelemetry/exporter-metrics-otlp-grpc');
const { PeriodicExportingMetricReader } = require('@opentelemetry/sdk-metrics');
const { Resource } = require('@opentelemetry/resources');
const { ATTR_SERVICE_NAME, ATTR_SERVICE_VERSION } = require('@opentelemetry/semantic-conventions');

const sdk = new NodeSDK({
  resource: new Resource({
    [ATTR_SERVICE_NAME]: 'order-service',
    [ATTR_SERVICE_VERSION]: '2.1.0',
  }),
  traceExporter: new OTLPTraceExporter({ url: 'http://localhost:4317' }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({ url: 'http://localhost:4317' }),
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();

process.on('SIGTERM', () => {
  sdk.shutdown().then(() => process.exit(0));
});
```

```javascript
// app.js — require tracing first
require('./tracing');
const express = require('express');
// ... rest of app
```

### SDK Setup (ESM)

```javascript
// tracing.mjs
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-grpc';

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({ url: 'http://localhost:4317' }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

```bash
# ESM: use --import instead of --require
node --import ./tracing.mjs app.mjs
```

### Manual Spans

```javascript
const { trace, SpanStatusCode } = require('@opentelemetry/api');

const tracer = trace.getTracer('order-service');

async function processOrder(orderId) {
  return tracer.startActiveSpan('process-order', async (span) => {
    span.setAttribute('order.id', orderId);
    try {
      const result = await doWork(orderId);
      span.setAttribute('order.status', result.status);
      return result;
    } catch (err) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
      span.recordException(err);
      throw err;
    } finally {
      span.end();  // always end the span
    }
  });
}
```

---

## Go

### Important: Go Has No Auto-Instrumentation

Unlike Python and Node.js, Go does not support zero-code instrumentation. All spans are added manually or via instrumentation libraries for specific packages.

### Install

```bash
go get go.opentelemetry.io/otel \
       go.opentelemetry.io/otel/sdk \
       go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc \
       go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp
```

### SDK Setup

```go
package main

import (
    "context"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
)

func initTracer(ctx context.Context) (*sdktrace.TracerProvider, error) {
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("localhost:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }

    res := resource.NewWithAttributes(
        semconv.SchemaURL,
        semconv.ServiceName("order-service"),
        semconv.ServiceVersion("2.1.0"),
    )

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1))),
    )
    otel.SetTracerProvider(tp)
    return tp, nil
}

func main() {
    ctx := context.Background()
    tp, err := initTracer(ctx)
    if err != nil {
        panic(err)
    }
    defer tp.Shutdown(ctx)   // flush remaining spans on exit
    // ... rest of app
}
```

### Manual Spans

```go
tracer := otel.Tracer("order-service")

func processOrder(ctx context.Context, orderID string) error {
    ctx, span := tracer.Start(ctx, "process-order")
    defer span.End()

    span.SetAttributes(
        attribute.String("order.id", orderID),
    )

    if err := doWork(ctx, orderID); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return err
    }
    return nil
}
```

### HTTP Middleware

```go
import "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

// Wrap your HTTP handler — every request gets a span automatically
http.Handle("/checkout", otelhttp.NewHandler(checkoutHandler, "checkout"))
```

Instrumentation packages exist for: `net/http`, `database/sql`, `gRPC`, `gin`, `echo`, `chi`, `aws-sdk-go-v2`, and more under `go.opentelemetry.io/contrib/`.

---

## Java

### Auto-Instrumentation via Java Agent

Java's auto-instrumentation is the most powerful: a single `-javaagent` flag instruments the entire JVM process with no code changes.

```bash
# Download the agent
curl -L https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar \
     -o opentelemetry-javaagent.jar

# Run with agent
java \
  -javaagent:opentelemetry-javaagent.jar \
  -DOTEL_SERVICE_NAME=order-service \
  -DOTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
  -DOTEL_TRACES_SAMPLER=parentbased_traceidratio \
  -DOTEL_TRACES_SAMPLER_ARG=0.1 \
  -jar myapp.jar
```

The agent instruments: Spring (MVC, Boot, WebFlux), JDBC, Hibernate, Kafka, gRPC, Netty, OkHttp, AWS SDK, and 100+ more libraries automatically.

### Spring Boot Configuration

With the Java agent, Spring Boot apps need no code changes. Config via `application.properties`:

```properties
# application.properties
otel.service.name=order-service
otel.exporter.otlp.endpoint=http://localhost:4317
otel.traces.sampler=parentbased_traceidratio
otel.traces.sampler.arg=0.1
otel.propagators=tracecontext,baggage
```

Or via environment:

```yaml
# docker-compose.yml
environment:
  OTEL_SERVICE_NAME: order-service
  OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
  OTEL_TRACES_SAMPLER: parentbased_traceidratio
  OTEL_TRACES_SAMPLER_ARG: "0.1"
```

### Manual Spans (Java SDK)

For custom business logic spans when using the SDK directly (without the agent):

```xml
<!-- pom.xml -->
<dependency>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-api</artifactId>
  <version>1.40.0</version>
</dependency>
```

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.api.trace.Tracer;

Tracer tracer = GlobalOpenTelemetry.getTracer("order-service");

Span span = tracer.spanBuilder("processOrder")
    .setAttribute("order.id", orderId)
    .startSpan();

try (var scope = span.makeCurrent()) {
    processOrder(orderId);
} catch (Exception e) {
    span.recordException(e);
    span.setStatus(StatusCode.ERROR, e.getMessage());
    throw e;
} finally {
    span.end();
}
```

---

## Language Comparison

| | Python | Node.js | Go | Java |
|-|--------|---------|-----|------|
| **Auto-instrumentation** | ✅ CLI wrapper | ✅ `--require` flag | ❌ Manual only | ✅ `-javaagent` flag |
| **Manual spans** | ✅ Context manager | ✅ `startActiveSpan` | ✅ `tracer.Start` | ✅ `spanBuilder` |
| **Setup complexity** | Low | Low | Medium | Low (with agent) |
| **Async support** | asyncio native | Promises / async-await | Goroutines (context) | CompletableFuture |
| **Library coverage** | Broad | Broad | Selective | Very broad |

---

## What's Next

- **[08 — Kubernetes](otel-kubernetes.md)** — Deploying the OTEL Operator, auto-injecting instrumentation via pod annotations, and scaling the Collector in a cluster
- **[09 — Tools Reference](otel-tools.md)** — Instrumentation library and backend comparison tables

---

[← Back to Guide Index](otel-overview.md)
