# 08 — Kubernetes: OTEL Operator, DaemonSet & Auto-Instrumentation

**Visual guide:** `otel-kubernetes.html`
**Covers:** The OTEL Operator and its CRDs, Collector deployment modes (DaemonSet / Deployment / Sidecar), cert-manager setup, auto-instrumentation injection via pod annotations, production Helm configuration, and scaling considerations.

---

## Why Kubernetes Changes the Picture

Running OTEL in Kubernetes introduces concerns that don't exist in a simple VM or container deployment:

- **Pods are ephemeral** — you need telemetry infrastructure that survives pod restarts
- **Many pods per node** — per-node buffering (DaemonSet) reduces traffic to the gateway
- **Pod metadata** — traces and metrics need Kubernetes context (namespace, pod name, deployment)
- **Credential management** — API keys and TLS certs need Kubernetes-native handling
- **Scale** — Collector replicas need to be managed like any other workload

The **OTEL Operator** is the Kubernetes-native way to manage all of this.

---

## The OTEL Operator

The OTEL Operator is a Kubernetes operator that manages two custom resource types:

| CRD | Purpose |
|-----|---------|
| `OpenTelemetryCollector` | Declares a Collector deployment (replaces writing Deployments/DaemonSets by hand) |
| `Instrumentation` | Declares auto-instrumentation config; injected into pods via annotations |

### Install: cert-manager (prerequisite)

The OTEL Operator uses webhooks, which require TLS certificates. cert-manager handles certificate provisioning automatically.

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Wait for cert-manager to be ready
kubectl wait --for=condition=Available deployment --all -n cert-manager --timeout=120s
```

### Install: OTEL Operator

```bash
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```

Verify:
```bash
kubectl get pods -n opentelemetry-operator-system
# NAME                                      READY   STATUS    RESTARTS
# opentelemetry-operator-xxxxxxxxx-xxxxx    2/2     Running   0
```

---

## Collector Deployment Modes

The Operator supports three deployment modes, chosen via `.spec.mode` in the `OpenTelemetryCollector` resource.

### DaemonSet (Agent Tier)

One Collector pod per node. Receives telemetry from all pods on that node, attaches Kubernetes metadata, and forwards to the gateway.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-agent
  namespace: observability
spec:
  mode: daemonset

  resources:
    limits:
      cpu: 200m
      memory: 256Mi
    requests:
      cpu: 50m
      memory: 64Mi

  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      hostmetrics:
        collection_interval: 30s
        scrapers:
          cpu:
          memory:
          disk:
          network:

    processors:
      memory_limiter:
        check_interval: 1s
        limit_mib: 200
        spike_limit_mib: 50
      k8sattributes:
        extract:
          metadata:
            - k8s.pod.name
            - k8s.pod.uid
            - k8s.namespace.name
            - k8s.deployment.name
            - k8s.node.name
        pod_association:
          - sources:
              - from: resource_attribute
                name: k8s.pod.ip
          - sources:
              - from: connection
      resourcedetection:
        detectors: [env, k8snode, system]
      batch:
        send_batch_size: 500
        timeout: 2s

    exporters:
      otlp:
        endpoint: otel-gateway-collector:4317
        tls:
          insecure: true

    service:
      pipelines:
        traces:
          receivers:  [otlp]
          processors: [memory_limiter, k8sattributes, resourcedetection, batch]
          exporters:  [otlp]
        metrics:
          receivers:  [otlp, hostmetrics]
          processors: [memory_limiter, k8sattributes, resourcedetection, batch]
          exporters:  [otlp]
        logs:
          receivers:  [otlp]
          processors: [memory_limiter, k8sattributes, resourcedetection, batch]
          exporters:  [otlp]
```

### Deployment (Gateway Tier)

Runs as a scaled Deployment. Handles tail sampling, PII scrubbing, and routing to final backends.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-gateway
  namespace: observability
spec:
  mode: deployment
  replicas: 3

  resources:
    limits:
      cpu: 1000m
      memory: 2Gi
    requests:
      cpu: 250m
      memory: 512Mi

  # Tail sampling requires all spans for a trace to reach the same replica.
  # The loadbalancingexporter in the agent tier handles this; the gateway
  # itself can be placed behind a standard service.

  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317

    processors:
      memory_limiter:
        check_interval: 1s
        limit_mib: 1800
        spike_limit_mib: 400
      tail_sampling:
        decision_wait: 10s
        num_traces: 100000
        policies:
          - name: errors-always
            type: status_code
            status_code: {status_codes: [ERROR]}
          - name: slow-always
            type: latency
            latency: {threshold_ms: 2000}
          - name: baseline
            type: probabilistic
            probabilistic: {sampling_percentage: 10}
      attributes:
        actions:
          - key: user.email
            action: delete
          - key: http.request.header.authorization
            action: delete
      batch:
        send_batch_size: 2000
        timeout: 10s

    exporters:
      otlp/tempo:
        endpoint: tempo:4317
        tls:
          insecure: true
      prometheusremotewrite:
        endpoint: http://prometheus:9090/api/v1/write
      loki:
        endpoint: http://loki:3100/loki/api/v1/push

    service:
      pipelines:
        traces:
          receivers:  [otlp]
          processors: [memory_limiter, tail_sampling, attributes, batch]
          exporters:  [otlp/tempo]
        metrics:
          receivers:  [otlp]
          processors: [memory_limiter, attributes, batch]
          exporters:  [prometheusremotewrite]
        logs:
          receivers:  [otlp]
          processors: [memory_limiter, attributes, batch]
          exporters:  [loki]
```

### Sidecar

Injects a Collector container into each pod. Useful when pods need pod-local buffering or cannot reach the DaemonSet Collector directly.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-sidecar
  namespace: my-app
spec:
  mode: sidecar
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
    processors:
      batch: {}
    exporters:
      otlp:
        endpoint: otel-agent-collector:4317
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers:  [otlp]
          processors: [batch]
          exporters:  [otlp]
```

Annotate pods to receive the sidecar:

```yaml
metadata:
  annotations:
    sidecar.opentelemetry.io/inject: "otel-sidecar"
```

---

## Auto-Instrumentation Injection

The `Instrumentation` CRD defines language-specific auto-instrumentation config. The Operator injects it into matching pods via annotations — no Dockerfile changes required.

### Instrumentation Resource

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: otel-instrumentation
  namespace: my-app
spec:
  exporter:
    endpoint: http://otel-agent-collector:4317

  propagators:
    - tracecontext
    - baggage

  sampler:
    type: parentbased_traceidratio
    argument: "0.1"

  python:
    env:
      - name: OTEL_LOGS_EXPORTER
        value: otlp

  nodejs:
    env:
      - name: OTEL_LOGS_EXPORTER
        value: otlp

  java:
    env:
      - name: OTEL_LOGS_EXPORTER
        value: otlp

  go:
    env:
      - name: OTEL_GO_AUTO_TARGET_EXE
        value: /app/server    # path to the Go binary inside the container
```

### Pod Annotations

Add one annotation to a pod (or Deployment template) to enable auto-instrumentation for that language:

```yaml
# Python
annotations:
  instrumentation.opentelemetry.io/inject-python: "my-app/otel-instrumentation"

# Node.js
annotations:
  instrumentation.opentelemetry.io/inject-nodejs: "my-app/otel-instrumentation"

# Java
annotations:
  instrumentation.opentelemetry.io/inject-java: "my-app/otel-instrumentation"

# Go (uses eBPF — requires privileged pod and Linux kernel ≥ 5.4)
annotations:
  instrumentation.opentelemetry.io/inject-go: "my-app/otel-instrumentation"
```

### Full Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        instrumentation.opentelemetry.io/inject-python: "my-app/otel-instrumentation"
    spec:
      containers:
        - name: order-service
          image: myregistry/order-service:2.1.0
          ports:
            - containerPort: 8080
          env:
            - name: OTEL_SERVICE_NAME
              value: order-service
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "service.version=2.1.0,deployment.environment=production"
```

The Operator's webhook intercepts the pod creation and injects the appropriate init container, environment variables, and volume mounts automatically.

---

## Deployment Topology

```
┌─────────────────────────────────────────┐
│  Kubernetes Cluster                     │
│                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐│
│  │  Node 1 │  │  Node 2 │  │  Node 3 ││
│  │         │  │         │  │         ││
│  │ [Pod A] │  │ [Pod C] │  │ [Pod E] ││
│  │ [Pod B] │  │ [Pod D] │  │ [Pod F] ││
│  │         │  │         │  │         ││
│  │[DaemonSet│  │[DaemonSet│  │[DaemonSet││
│  │ Agent]  │  │ Agent]  │  │ Agent]  ││
│  └────┬────┘  └────┬────┘  └────┬────┘│
│       │             │             │     │
│       └─────────────┴─────────────┘     │
│                     │ OTLP              │
│                     ▼                   │
│          ┌──────────────────┐           │
│          │ Gateway Collectors│           │
│          │   (Deployment ×3) │           │
│          └─────────┬────────┘           │
└────────────────────┼────────────────────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
       [Tempo]  [Prometheus] [Loki/S3]
```

---

## RBAC for k8sattributes

The `k8sattributes` processor needs permission to read pod metadata from the Kubernetes API:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector
rules:
  - apiGroups: [""]
    resources: ["pods", "namespaces", "nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["replicasets"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: otel-collector
subjects:
  - kind: ServiceAccount
    name: otel-agent-collector
    namespace: observability
```

The OTEL Operator creates the ServiceAccount automatically; you provide the ClusterRoleBinding.

---

## Helm Deployment

For production, the community Helm chart is the recommended installation method.

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# Install the Operator
helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
  --namespace opentelemetry-operator-system \
  --create-namespace \
  --set manager.collectorImage.repository=otel/opentelemetry-collector-k8s

# Install a Collector (agent + gateway) via values file
helm install otel-collector open-telemetry/opentelemetry-collector \
  --namespace observability \
  --create-namespace \
  -f otel-collector-values.yaml
```

Example `otel-collector-values.yaml`:

```yaml
mode: daemonset

presets:
  hostMetrics:
    enabled: true
  kubernetesAttributes:
    enabled: true
  logsCollection:
    enabled: true

config:
  exporters:
    otlp:
      endpoint: otel-gateway:4317
      tls:
        insecure: true
  service:
    pipelines:
      traces:
        exporters: [otlp]
      metrics:
        exporters: [otlp]
      logs:
        exporters: [otlp]
```

---

## Resource Limits & Scaling

### Sizing guidelines

| Tier | CPU request | CPU limit | Memory request | Memory limit |
|------|------------|-----------|----------------|-------------|
| DaemonSet agent | 50m | 200m | 64Mi | 256Mi |
| Gateway (per replica) | 250m | 1000m | 512Mi | 2Gi |
| Sidecar | 25m | 100m | 32Mi | 128Mi |

### Horizontal Pod Autoscaler (gateway)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: otel-gateway-hpa
  namespace: observability
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: otel-gateway-collector
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75
```

### Tail sampling constraint

When using tail-based sampling in the gateway, all spans for a given trace must arrive at the **same gateway replica**. Use the `loadbalancingexporter` in the agent tier to route by `trace_id`:

```yaml
# In the agent Collector config
exporters:
  loadbalancing:
    protocol:
      otlp:
        tls:
          insecure: true
    resolver:
      k8s:
        service: otel-gateway-collector     # headless service
        ports: [4317]
```

---

## Deployment Checklist

| Step | Done? |
|------|-------|
| cert-manager installed and Ready | ☐ |
| OTEL Operator installed | ☐ |
| `observability` namespace created | ☐ |
| DaemonSet `OpenTelemetryCollector` applied | ☐ |
| Gateway `OpenTelemetryCollector` applied | ☐ |
| RBAC ClusterRole + ClusterRoleBinding applied | ☐ |
| `Instrumentation` resource created per namespace | ☐ |
| Pod annotations added for each language | ☐ |
| Spans appearing in backend | ☐ |
| Kubernetes metadata present on spans | ☐ |

---

## What's Next

- **[06 — Ecosystem](otel-ecosystem.md)** — Backend and SaaS platform choices that work with this topology
- **[04 — Sampling](otel-sampling.md)** — Tail sampling policy design for the gateway tier
- **[03 — Collector](otel-collector.md)** — Full processor and exporter reference

---

[← Back to Guide Index](otel-overview.md)
