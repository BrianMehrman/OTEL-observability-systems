# Distributed System OTEL Migration Guide

> **Your Stack**: Ruby on Rails · Sidekiq Enterprise · Java Microservices · Node.js APIs ·
> RabbitMQ · Kafka · Redis · AWS SNS/SQS · EC2/VM + Kubernetes

---

## Current Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            USER REQUESTS (HTTPS)                            │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Rails Web Server (Ruby)                                       EC2 / K8s    │
│  ├── Puma + Rails 7                                                         │
│  ├── ──► Redis (session cache)                                              │
│  └── ──► RabbitMQ (event publishing)                                        │
└────────────────────────────┬────────────────────────────────────────────────┘
                             │ HTTP/REST
          ┌──────────────────┼─────────────────────────┐
          ▼                  ▼                          ▼
┌──────────────────┐  ┌────────────────────┐  ┌─────────────────────────────┐
│  API Gateway     │  │  Sidekiq Enterprise│  │  AWS SNS / SQS              │
│  (EC2 / K8s)     │  │  (Ruby, EC2/K8s)   │  │  (Managed)                  │
└────────┬─────────┘  │  └── Redis queue   │  └──────────────┬──────────────┘
         │            └────────────────────┘                 │
         │ HTTP/gRPC                                         │
  ┌──────┼──────────────────┐                               │
  ▼      ▼                  ▼                               ▼
┌──────────┐  ┌──────────┐  ┌──────────┐       ┌─────────────────────────────┐
│  Java    │  │  Java    │  │  Java    │       │  Other Services / Lambdas   │
│ Service  │  │ Service  │  │ Snowflake│       └─────────────────────────────┘
│ (K8s)   │  │ (K8s)   │  │ API (K8s)│
└──────────┘  └──────────┘  └────┬─────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │   Snowflake     │
                         │  Data Lake      │
                         └─────────────────┘
                                  ▲
                         ┌────────┴────────┐
                         │  Node.js        │
                         │  Snowflake API  │
                         │  (K8s / EC2)   │
                         └─────────────────┘

Data Pipeline:
Rails / Services ──► Kafka ──► Data Warehouse / Snowflake (CDC / replication)

Current Observability:
All services ──► Datadog agent ──► Datadog (logs + partial APM)
Some traces are DD-inferred (log correlation), not explicitly instrumented
```

---

## Observability Inventory

| Service | Language | Platform | Explicit Tracing | Log Routing | Priority |
|---------|----------|----------|-----------------|-------------|----------|
| Rails Web | Ruby | EC2 / K8s | ❌ DD-inferred | Datadog | 🔴 High |
| Sidekiq Jobs | Ruby | EC2 / K8s | ❌ DD-inferred | Datadog | 🔴 High |
| API Gateway | varies | EC2 / K8s | ⚠️ Partial | Datadog | 🟡 Medium |
| Java Service(s) | Java | Kubernetes | ⚠️ Partial DD APM | Datadog | 🟡 Medium |
| Java Snowflake API | Java | K8s / EC2 | ❌ None | Datadog | 🟡 Medium |
| Node.js Snowflake API | Node.js | K8s / EC2 | ❌ None | Datadog | 🟡 Medium |
| RabbitMQ | — | EC2 | n/a (broker) | Datadog | Propagation only |
| Kafka | — | EC2 / MSK | n/a (broker) | Datadog | Propagation only |
| Redis | — | Managed | n/a | Datadog | n/a |
| SNS / SQS | — | AWS Managed | n/a | AWS | Propagation only |

---

## What Datadog's Inference Does (and What You Must Replace)

When services have no explicit instrumentation, Datadog fills in the service map using:

| Inference Method | How It Works | What You Lose Without It |
|-----------------|--------------|--------------------------|
| **Log correlation** | Matches `dd.trace_id` / `dd.span_id` injected by DD agents | Trace-log linking |
| **APM header passthrough** | Uninstrumented services transparently pass `x-datadog-*` headers | Request continuity |
| **Network Performance Monitoring** | Tracks TCP connections between hosts to infer service topology | Service map edges |
| **Service Entry Inference** | DD APM infers service entry points from network + log data | Automatic service discovery |

**Critical risk**: Services with no OTEL instrumentation become completely invisible in Grafana,
SigNoz, and Jaeger. The service map only shows services that emit spans.

**Mitigation**: Never decommission Datadog for a service until that service has been
explicitly instrumented with OTEL. Use the phased migration below.

---

## Two Goals, One Strategy

```
Goal 1: Cut costs — stop sending non-prod logs/traces to Datadog
Goal 2: Improve quality — replace inferred traces with explicit OTEL spans

Both goals share the same infrastructure:
  App code        → same everywhere (OTEL SDK)
  OTEL Collector  → different config per environment
  Backend         → Datadog (prod) / SigNoz (non-prod)
```

---

## Phase 1 — Immediate Cost Win: Log Routing (No Code Changes)

Before touching any application code, you can reroute non-prod logs away from Datadog
using an OTEL Collector with a filelog receiver. This is the fastest ROI.

Deploy an OTEL Collector alongside each non-prod service using this config:

```yaml
# otel-collector-nonprod.yaml
receivers:
  filelog:
    include:
      - /var/log/app/*.log
      - /var/log/rails/*.log
    start_at: beginning
    operators:
      - type: json_parser
        timestamp:
          parse_from: attributes.timestamp
          layout: '%Y-%m-%dT%H:%M:%S.%fZ'

  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 1000

  resource:
    attributes:
      - key: deployment.environment
        value: "${DEPLOY_ENV}"           # staging / integration / test
        action: upsert
      - key: service.name
        value: "${SERVICE_NAME}"
        action: upsert

exporters:
  # Non-prod: route to SigNoz
  otlp/signoz:
    endpoint: "http://signoz-otel-collector:4317"
    tls:
      insecure: true

  # Keep prod going to Datadog during migration
  datadog:
    api:
      key: "${DD_API_KEY}"
      site: datadoghq.com

service:
  pipelines:
    logs:
      receivers: [filelog, otlp]
      processors: [batch, resource]
      exporters: [otlp/signoz]      # swap to [datadog] for prod
    traces:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [otlp/signoz]
    metrics:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [otlp/signoz]
```

**Environment variable routing** — set in your deploy config:

```bash
# Integration / Staging
DEPLOY_ENV=staging
SERVICE_NAME=rails-web
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318

# Production
DEPLOY_ENV=production
DD_API_KEY=<your-key>
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
```

---

## Phase 2 — Ruby Services: Rails + Sidekiq

### Gemfile

```ruby
# Gemfile
gem 'opentelemetry-sdk'
gem 'opentelemetry-exporter-otlp'

# Auto-instrumentation gems (pick what you use)
gem 'opentelemetry-instrumentation-rails'
gem 'opentelemetry-instrumentation-active_record'
gem 'opentelemetry-instrumentation-rack'
gem 'opentelemetry-instrumentation-action_pack'
gem 'opentelemetry-instrumentation-action_view'
gem 'opentelemetry-instrumentation-sidekiq'    # traces enqueue + process
gem 'opentelemetry-instrumentation-redis'      # Redis calls
gem 'opentelemetry-instrumentation-bunny'      # RabbitMQ via Bunny
gem 'opentelemetry-instrumentation-faraday'    # outbound HTTP (Faraday)
gem 'opentelemetry-instrumentation-http'       # outbound HTTP (Net::HTTP)
```

### Initializer

```ruby
# config/initializers/opentelemetry.rb
require 'opentelemetry/sdk'
require 'opentelemetry/exporter/otlp'
require 'opentelemetry/instrumentation/all'

OpenTelemetry::SDK.configure do |c|
  c.service_name    = ENV.fetch('OTEL_SERVICE_NAME', 'rails-web')
  c.service_version = ENV.fetch('APP_VERSION', 'unknown')

  c.resource = OpenTelemetry::SDK::Resources::Resource.create({
    'deployment.environment' => ENV.fetch('DEPLOY_ENV', 'development'),
    'service.namespace'      => 'my-platform',
  })

  # Use all installed instrumentation gems automatically
  c.use_all

  # Or selectively enable:
  # c.use 'OpenTelemetry::Instrumentation::Rails'
  # c.use 'OpenTelemetry::Instrumentation::Sidekiq'
  # c.use 'OpenTelemetry::Instrumentation::Bunny'
end
```

### Environment Variables

```bash
# All environments — points to local OTEL Collector
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_SERVICE_NAME=rails-web
OTEL_TRACES_EXPORTER=otlp
OTEL_METRICS_EXPORTER=otlp
OTEL_LOGS_EXPORTER=otlp

# OTEL Collector routes to Datadog in prod, SigNoz in non-prod
# App code is identical — only collector config differs
```

### Sidekiq-Specific Notes

The `opentelemetry-instrumentation-sidekiq` gem automatically:
- Creates a **producer span** when a job is enqueued (in Rails web request)
- Creates a **consumer span** when Sidekiq processes the job
- Propagates trace context via Sidekiq job metadata (stored in Redis)
- Links consumer spans back to the original web request span

This gives you end-to-end visibility: HTTP request → enqueue → job processing.

### Adding Custom Business Spans

```ruby
# In any model, service object, or controller
tracer = OpenTelemetry.tracer_provider.tracer('my-service', '1.0')

def process_order(order_id)
  tracer.in_span('order.process', attributes: {
    'order.id'     => order_id,
    'order.source' => 'web'
  }) do |span|
    result = do_processing(order_id)
    span.set_attribute('order.status', result.status)
    result
  end
end
```

---

## Phase 3 — Java Services: Zero-Code Agent

The OpenTelemetry Java agent requires **zero code changes**. Drop the agent JAR and set
environment variables. All instrumentation (Spring Boot, JDBC, Kafka, RabbitMQ, HTTP) is
handled automatically.

### Download the Agent

```bash
# Download latest stable agent
curl -L https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar \
  -o /opt/otel/opentelemetry-javaagent.jar

# Pin to a specific version (recommended for stability)
# https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases
```

### JVM Startup

```bash
java \
  -javaagent:/opt/otel/opentelemetry-javaagent.jar \
  -Dotel.service.name=java-gateway \
  -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
  -Dotel.exporter.otlp.protocol=grpc \
  -Dotel.traces.exporter=otlp \
  -Dotel.metrics.exporter=otlp \
  -Dotel.logs.exporter=otlp \
  -Dotel.resource.attributes=deployment.environment=staging,service.namespace=my-platform \
  -jar app.jar
```

### Docker / Kubernetes Environment Variables

```yaml
# docker-compose.yml or Kubernetes Deployment
env:
  - name: JAVA_TOOL_OPTIONS
    value: "-javaagent:/opt/otel/opentelemetry-javaagent.jar"
  - name: OTEL_SERVICE_NAME
    value: "java-snowflake-api"
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector:4317"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "grpc"
  - name: OTEL_TRACES_EXPORTER
    value: "otlp"
  - name: OTEL_METRICS_EXPORTER
    value: "otlp"
  - name: OTEL_LOGS_EXPORTER
    value: "otlp"
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "deployment.environment=staging,service.namespace=my-platform"
```

### Spring Boot Specific

```xml
<!-- pom.xml — add actuator for metric exposure -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# application.yaml
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus
  tracing:
    sampling:
      probability: 1.0       # 100% in non-prod; reduce in prod
```

**What the Java agent instruments automatically:**
- Spring MVC / Spring Boot web endpoints
- JDBC / JPA database queries
- Kafka producer and consumer calls (with header propagation)
- RabbitMQ via Spring AMQP (with header propagation)
- AWS SDK v1 and v2 (SQS, SNS, S3, DynamoDB)
- HTTP clients (RestTemplate, WebClient, OkHttp, Apache HTTP)
- gRPC client and server

---

## Phase 4 — Node.js APIs

### Installation

```bash
npm install \
  @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-grpc \
  @opentelemetry/exporter-metrics-otlp-grpc \
  @opentelemetry/exporter-logs-otlp-grpc
```

### Bootstrap File

```javascript
// tracing.js  — loaded before any other module
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { OTLPMetricExporter } = require('@opentelemetry/exporter-metrics-otlp-grpc');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]:
      process.env.OTEL_SERVICE_NAME || 'node-snowflake-api',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]:
      process.env.DEPLOY_ENV || 'development',
    'service.namespace': 'my-platform',
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://otel-collector:4317',
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-fs': { enabled: false }, // noisy, usually skip
    }),
  ],
});

sdk.start();
process.on('SIGTERM', () => sdk.shutdown());
```

### package.json startup

```json
{
  "scripts": {
    "start": "node --require ./tracing.js app.js"
  }
}
```

**What `auto-instrumentations-node` covers:**
- HTTP / HTTPS outbound calls
- Express / Fastify / Koa / Hapi
- AWS SDK v2 and v3 (SQS, SNS, DynamoDB, S3)
- KafkaJS (Kafka producer + consumer)
- `ioredis` / `redis` clients
- `pg`, `mysql2`, `mongodb` database drivers
- `graphql`

---

## Phase 5 — Context Propagation Through Your Messaging Layer

This is the hardest part of the migration. Without explicit trace context propagation,
your async message flows will appear as disconnected traces — no link between the
producer service and the consumer service.

### RabbitMQ (Bunny gem — Ruby producer, Spring AMQP consumer)

**Ruby producer** — the Bunny instrumentation gem handles this automatically:
```ruby
# opentelemetry-instrumentation-bunny injects W3C TraceContext headers
# into AMQP message headers when you publish:
channel.default_exchange.publish(
  payload.to_json,
  routing_key: queue.name,
  content_type: 'application/json'
  # Trace headers injected automatically by the gem
)
```

**Java consumer** — Spring AMQP + the Java agent extract automatically:
```java
// The Java agent instruments @RabbitListener automatically
@RabbitListener(queues = "my.queue")
public void handleMessage(Message message) {
  // Span is automatically created and linked to producer trace
  // No manual code needed when using the javaagent
}
```

**Verify propagation is working:**
```bash
# Producer span should show child spans in the consuming service
# Look for spans with kind=CONSUMER linked to kind=PRODUCER spans
```

### Kafka (Java producer → Java consumer)

The Java agent instruments `kafka-clients` automatically. Trace context is propagated
via Kafka record headers (`traceparent`, `tracestate`).

```java
// Producer — no changes needed; agent injects headers automatically
ProducerRecord<String, String> record = new ProducerRecord<>("my-topic", key, value);
producer.send(record);  // traceparent header injected automatically

// Consumer — agent extracts context automatically
@KafkaListener(topics = "my-topic")
public void consume(ConsumerRecord<String, String> record) {
  // Span created and linked to producer trace automatically
}
```

**Node.js Kafka (KafkaJS):**
```javascript
// auto-instrumentations-node includes @opentelemetry/instrumentation-kafkajs
// Producer and consumer spans are created and linked automatically

const producer = kafka.producer();
await producer.send({
  topic: 'my-topic',
  messages: [{ key, value }],  // traceparent injected in message headers
});

const consumer = kafka.consumer({ groupId: 'my-group' });
await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    // Span automatically linked to producer trace
  },
});
```

### AWS SNS / SQS

AWS SDK instrumentation is included in both the Java agent and `auto-instrumentations-node`.
Trace context is propagated via SQS **message attributes** (SNS wraps SQS attributes).

**Important limitation**: SQS has a max of 10 message attributes per message. Reserve
one attribute slot for `AWSTraceHeader` (used by X-Ray and OTEL AWS propagator).

```javascript
// Node.js — auto-instrumentations-node handles SQS automatically
// When you send a message, traceparent is injected into MessageAttributes
const command = new SendMessageCommand({
  QueueUrl: queueUrl,
  MessageBody: JSON.stringify(payload),
  // Do NOT set MessageSystemAttributes here — SDK injects AWSTraceHeader
});
await sqsClient.send(command);
```

```java
// Java — the Java agent injects into SQS MessageAttributes automatically
SendMessageRequest req = SendMessageRequest.builder()
    .queueUrl(queueUrl)
    .messageBody(payload)
    .build();
sqsClient.sendMessage(req);  // AWSTraceHeader attribute injected automatically
```

**SNS → SQS flow**: When SNS delivers to an SQS subscription, trace context in SNS
message attributes is preserved in the SQS message. The consumer extracts it from the
`MessageAttributes` in the SNS notification envelope.

### Redis / Sidekiq

Sidekiq stores jobs in Redis as JSON. The `opentelemetry-instrumentation-sidekiq` gem
serializes trace context into the Sidekiq job payload automatically.

```
Web Request span
  └── ActiveRecord span (DB query)
  └── Sidekiq::Client span (enqueue)        ← producer side
        traceparent stored in job JSON
           └── Sidekiq::Server span          ← consumer side (different process)
                 └── ActiveRecord span
                 └── Redis span
```

No manual code is needed — the gem handles both sides.

---

## OTEL Collector Deployment by Platform

### EC2 / VM Services — Docker Sidecar

Run a Collector container alongside each service:

```yaml
# docker-compose.yml (per service)
services:
  app:
    image: my-service:latest
    environment:
      OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector:4318"
      OTEL_EXPORTER_OTLP_PROTOCOL: "http/protobuf"
    depends_on:
      - otel-collector

  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.96.0
    volumes:
      - ./otel-collector.yaml:/etc/otelcol-contrib/config.yaml
    ports:
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP
```

### Kubernetes Services — OTEL Operator + DaemonSet

```yaml
# Install Operator
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml

# Deploy a DaemonSet Collector (one per node)
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-daemonset
spec:
  mode: daemonset
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      batch:
        timeout: 5s
      k8sattributes:
        extract:
          metadata:
            - k8s.namespace.name
            - k8s.pod.name
            - k8s.node.name
            - k8s.deployment.name
    exporters:
      otlp/signoz:
        endpoint: "http://signoz-otel-collector.monitoring:4317"
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch, k8sattributes]
          exporters: [otlp/signoz]
        logs:
          receivers: [otlp]
          processors: [batch, k8sattributes]
          exporters: [otlp/signoz]
        metrics:
          receivers: [otlp]
          processors: [batch, k8sattributes]
          exporters: [otlp/signoz]
```

**Auto-instrumentation injection** (no Dockerfile changes for Java):

```yaml
# Instrumentation resource — patches pods via webhook
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: java-instrumentation
spec:
  exporter:
    endpoint: http://otel-daemonset-collector:4317
  propagators:
    - tracecontext
    - baggage
    - b3
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest

---
# Annotate Kubernetes Deployment to inject the agent
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-gateway
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-java: "true"
```

---

## Environment Tier Strategy

The key principle: **application code is identical across all environments**.
Only the OTEL Collector configuration changes.

```
All services → OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318

Production Collector  → datadog exporter       → Datadog
Staging Collector     → otlp exporter          → SigNoz (persistent)
Integration Collector → otlp exporter          → SigNoz (ephemeral, per-env)
Test Collector        → debug / no-op exporter → nowhere (fast, no cost)
```

### Production — Datadog via OTEL Collector

```yaml
# otel-collector-prod.yaml
exporters:
  datadog:
    api:
      key: "${DD_API_KEY}"
      site: datadoghq.com
    traces:
      span_name_as_resource_name: true
    host_metadata:
      enabled: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [datadog]
    logs:
      receivers: [otlp, filelog]
      processors: [batch, resource]
      exporters: [datadog]
    metrics:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [datadog]
```

### Staging — SigNoz (Persistent)

```yaml
# deploy SigNoz to your staging cluster once
helm repo add signoz https://charts.signoz.io
helm install signoz signoz/signoz \
  --namespace monitoring \
  --create-namespace \
  --set frontend.service.type=ClusterIP
```

```yaml
# otel-collector-staging.yaml
exporters:
  otlp/signoz:
    endpoint: "http://signoz-otel-collector.monitoring:4317"
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [otlp/signoz]
    logs:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [otlp/signoz]
    metrics:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [otlp/signoz]
```

### Integration — Ephemeral SigNoz per Environment

Each isolated integration environment gets its own short-lived SigNoz instance.
Since it's ephemeral (torn down after E2E tests), you can use minimal resource settings.

```yaml
# docker-compose.integration.yaml — spin up with each isolated env
services:
  signoz:
    image: signoz/signoz:latest
    environment:
      - SIGNOZ_LOCAL_DB_PATH=/var/lib/signoz/signoz.db
    ports:
      - "3301:3301"    # SigNoz UI
    volumes:
      - signoz-data:/var/lib/signoz

  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.96.0
    volumes:
      - ./otel-collector-integration.yaml:/etc/otelcol-contrib/config.yaml
    environment:
      - SIGNOZ_ENDPOINT=http://signoz:4317
```

**Cost impact**: Integration environments previously sent all telemetry to Datadog.
With ephemeral SigNoz, that cost drops to **$0** — only compute for the SigNoz container.

### Test / CI — No-Op Exporter

For unit tests and fast CI runs, disable OTEL entirely to avoid network overhead:

```bash
# .env.test
OTEL_TRACES_EXPORTER=none
OTEL_METRICS_EXPORTER=none
OTEL_LOGS_EXPORTER=none
```

```ruby
# Or in spec_helper.rb / test_helper.rb
ENV['OTEL_TRACES_EXPORTER'] = 'none' if Rails.env.test?
```

```java
// Or pass as JVM arg in test runner
-Dotel.traces.exporter=none
-Dotel.metrics.exporter=none
```

---

## Migration Roadmap

| Phase | Scope | Code Changes? | Cost Impact | Risk |
|-------|-------|--------------|-------------|------|
| **1** | OTEL Collector + log re-routing | None | High (immediate) | Low |
| **2** | Ruby: Rails + Sidekiq | Gemfile + initializer | None yet | Low |
| **3** | Java services (javaagent) | Dockerfile only | Per-service | Low |
| **4** | Node.js APIs | package.json + tracing.js | Per-service | Low |
| **5** | Async message propagation | Verify headers flow | None (auto) | Medium |
| **6** | Decommission Datadog per service | Remove DD agent | Per-service | Low |

### Phase 1: Log Router (Days 1–7)

1. Deploy OTEL Collector to each non-prod environment (staging, integration, test)
2. Configure filelog receiver pointing at existing log paths
3. Route to SigNoz in staging; ephemeral SigNoz in integration
4. Set `OTEL_TRACES_EXPORTER=none` in test environments
5. **Verify**: Logs appear in SigNoz, Datadog log volume drops for non-prod
6. **Rollback**: Remove collector, logs revert to file-only (Datadog agent still running)

### Phase 2: Ruby Services (Weeks 2–3)

1. Add gems to Gemfile, bundle install
2. Add `config/initializers/opentelemetry.rb`
3. Set OTEL env vars in docker-compose / K8s manifests
4. Deploy to staging → verify traces appear in SigNoz
5. Check Sidekiq job traces link back to originating web request
6. Check RabbitMQ publish spans appear
7. **Parallel run**: Keep Datadog agent running for 1 week to compare
8. Remove Datadog agent from staging Rails/Sidekiq after validation

### Phase 3: Java Services (Weeks 3–4)

1. Add javaagent JAR to Docker image or K8s init container
2. Set `JAVA_TOOL_OPTIONS`, `OTEL_*` env vars
3. For K8s services: annotate with `inject-java: "true"` (OTEL Operator)
4. Deploy to staging → verify traces in SigNoz
5. Verify Kafka spans link producer ↔ consumer
6. Verify Spring AMQP (RabbitMQ) spans appear
7. Remove Datadog APM agent from staging Java services after validation

### Phase 4: Node.js APIs (Weeks 4–5)

1. `npm install` the OTEL packages
2. Create `tracing.js`, update `package.json` start command
3. Set env vars, deploy to staging
4. Verify Snowflake API traces appear and link to caller spans
5. Remove Datadog agent from Node.js staging services after validation

### Phase 5: Close Messaging Gaps (Weeks 5–6)

1. Run end-to-end trace validation: send a request, trace it through all hops
2. Check: Web → Sidekiq → RabbitMQ → Java consumer → Kafka → downstream
3. Check: Rails → SNS → SQS → Lambda/service
4. For any broken links: inspect headers in message broker, verify propagator config
5. If using B3 propagation in some services: configure the collector's propagators list
6. Deploy validated config to production (prod still routes to Datadog)

---

## What You Gain vs. Datadog Inference

| Capability | Datadog Inference | Explicit OTEL |
|------------|------------------|---------------|
| Service visibility | ✅ Automatic (partial) | ✅ All instrumented services |
| Trace depth | ⚠️ Shallow (log correlation) | ✅ Full span tree with timing |
| Kafka trace linking | ❌ No | ✅ Via record headers |
| RabbitMQ trace linking | ❌ No | ✅ Via AMQP headers (Bunny + Spring) |
| SQS/SNS trace linking | ❌ No | ✅ Via message attributes |
| Sidekiq job linking | ⚠️ Partial (DD APM) | ✅ Full enqueue → process chain |
| Custom business spans | ❌ No | ✅ Manual spans anywhere |
| Cross-language consistency | ❌ Varies by DD agent | ✅ Same W3C TraceContext everywhere |
| Non-prod cost | ❌ Full Datadog billing | ✅ $0 (OSS backends) |
| Backend portability | ❌ Locked to Datadog | ✅ Any OTLP-compatible backend |
| Propagation standard | `x-datadog-*` headers | W3C `traceparent` / `tracestate` |

---

## Estimated Cost Impact

| Environment | Current State | After Migration | Savings |
|-------------|---------------|-----------------|---------|
| Production | Datadog | Datadog (unchanged) | $0 |
| Staging | Datadog (per-service) | SigNoz (self-hosted) | ~80–90% |
| Integration | Datadog (full ecosystem) | Ephemeral SigNoz | ~100% |
| Test / CI | Datadog logs | No-op exporter | ~100% |

The largest savings come from integration environments — if you spin up 3–5 isolated
integration environments per sprint, you're currently paying full Datadog ingestion rates
for each one. Ephemeral SigNoz runs on your existing compute for $0 in backend costs.

---

## Related Guides

- [Rails Stack Analysis](otel-rails-stack.md) — Options A/B/C for the Rails pipeline
- [Migration Plan](otel-migration-plan.md) — Phase-by-phase from Logstash → OTEL
- [Non-Prod Options](otel-nonprod-options.md) — Grafana LGTM, SigNoz, Uptrace, Jaeger comparison
- [Distributed Tracing Theory](otel-distributed.md) — W3C TraceContext, B3, baggage
- [Kubernetes Deployment](otel-kubernetes.md) — OTEL Operator, DaemonSet, auto-instrumentation
- [Language SDKs](otel-sdks.md) — SDK setup for Python, Node.js, Go, Java
