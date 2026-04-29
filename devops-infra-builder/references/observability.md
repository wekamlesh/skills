# Observability Reference

Use this file when implementing metrics, dashboards, logging, or distributed tracing.

## Table of Contents
1. kube-prometheus-stack (Prometheus + Grafana bundle)
2. VictoriaMetrics (Prometheus-compatible, resource-efficient)
3. Loki + Promtail (log aggregation)
4. OpenTelemetry Collector (unified telemetry pipeline)
5. Tempo (distributed tracing)
6. Jaeger (distributed tracing)
7. Grafana dashboard patterns
8. Observability stack comparison

---

## 1. kube-prometheus-stack

The standard all-in-one: Prometheus Operator, Prometheus, Alertmanager, Grafana, and default k8s dashboards.

### Install
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  -f infra/platform/observability/prometheus-values.yaml
```

`infra/platform/observability/prometheus-values.yaml`:
```yaml
grafana:
  enabled: true
  adminPassword: "changeme"   # use a secret in prod
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts:
      - grafana.example.com
    tls:
      - secretName: grafana-tls
        hosts:
          - grafana.example.com
  persistence:
    enabled: true
    size: 5Gi

prometheus:
  prometheusSpec:
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 20Gi

alertmanager:
  enabled: true

nodeExporter:
  enabled: true

kubeStateMetrics:
  enabled: true
```

### Add a ServiceMonitor (scrape your app)
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: default
  labels:
    release: kube-prometheus-stack    # must match Prometheus selector
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

Your app must expose `/metrics` in Prometheus format. For Node.js use `prom-client`, for Go use `prometheus/client_golang`.

### Verify
```bash
kubectl get pods -n monitoring
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
# Open http://localhost:3000, login admin/changeme
```

---

## 2. VictoriaMetrics

Drop-in Prometheus replacement. Uses 5–10x less RAM and disk. Recommended for Raspberry Pi and resource-constrained nodes.

**Experimental note**: VictoriaMetrics is production-stable but less common than Prometheus. The operator pattern mirrors Prometheus Operator closely.

### Install single-node (simplest)
```bash
helm repo add vm https://victoriametrics.github.io/helm-charts
helm repo update

helm upgrade --install victoria-metrics vm/victoria-metrics-single \
  --namespace monitoring --create-namespace \
  -f infra/platform/observability/vm-values.yaml
```

`infra/platform/observability/vm-values.yaml`:
```yaml
server:
  persistentVolume:
    enabled: true
    size: 20Gi
  retentionPeriod: "30d"
  scrape:
    enabled: true   # built-in scrape config (no Prometheus Operator needed)

# Keep Grafana from kube-prometheus-stack or install separately
```

Point Grafana datasource to VictoriaMetrics:
```yaml
# Grafana datasource
url: http://victoria-metrics-single-server:8428
```

VictoriaMetrics is fully PromQL-compatible — all Grafana dashboards built for Prometheus work unchanged.

---

## 3. Loki + Promtail (Log Aggregation)

Loki stores logs indexed only by labels (not full-text), keeping storage costs low. Promtail ships logs from each node.

### Install
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=10Gi
```

### Add Loki datasource to Grafana
In Grafana > Data Sources > Add Loki:
- URL: `http://loki:3100`

Or via Helm values:
```yaml
grafana:
  additionalDataSources:
    - name: Loki
      type: loki
      url: http://loki:3100
      access: proxy
```

### Query logs in Grafana (LogQL)
```logql
{namespace="default", app="my-app"} |= "error"
{namespace="default"} | json | level="error"
```

### Verify
```bash
kubectl get pods -n monitoring -l app=loki
kubectl get pods -n monitoring -l app=promtail
```

---

## 4. OpenTelemetry Collector

The OTel Collector is a vendor-neutral pipeline: receives traces/metrics/logs from apps, processes, and exports to any backend (Tempo, Jaeger, Prometheus, Loki, Datadog, etc.).

**Experimental note**: OTel is the industry standard in progress. OTLP protocol is stable; some receiver/exporter combos are still beta.

### Install
```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

helm upgrade --install otel-collector open-telemetry/opentelemetry-collector \
  --namespace monitoring \
  -f infra/platform/observability/otel-values.yaml
```

`infra/platform/observability/otel-values.yaml`:
```yaml
mode: deployment   # or daemonset for node-level collection

config:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

  processors:
    batch: {}
    memory_limiter:
      check_interval: 1s
      limit_mib: 400

  exporters:
    otlp:
      endpoint: tempo:4317
      tls:
        insecure: true
    prometheus:
      endpoint: "0.0.0.0:8889"

  service:
    pipelines:
      traces:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [otlp]
      metrics:
        receivers: [otlp]
        processors: [batch]
        exporters: [prometheus]
```

Apps send traces to `http://otel-collector:4318/v1/traces` (HTTP) or `otel-collector:4317` (gRPC).

---

## 5. Tempo (Distributed Tracing)

Tempo stores traces cheaply using object storage. Integrates natively with Grafana for trace visualization.

### Install
```bash
helm upgrade --install tempo grafana/tempo \
  --namespace monitoring \
  -f infra/platform/observability/tempo-values.yaml
```

`infra/platform/observability/tempo-values.yaml`:
```yaml
tempo:
  storage:
    trace:
      backend: local    # use s3 or gcs in production
      local:
        path: /var/tempo/traces

  receivers:
    otlp:
      protocols:
        grpc: {}
        http: {}
    jaeger:
      protocols:
        thrift_http: {}

persistence:
  enabled: true
  size: 10Gi
```

### Add Tempo datasource to Grafana
```yaml
grafana:
  additionalDataSources:
    - name: Tempo
      type: tempo
      url: http://tempo:3100
      access: proxy
      jsonData:
        tracesToLogsV2:
          datasourceUid: loki-uid   # link traces → logs
```

---

## 6. Jaeger (Distributed Tracing)

Jaeger is the older, battle-tested alternative to Tempo. More complex but very mature.

```bash
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

helm upgrade --install jaeger jaegertracing/jaeger \
  --namespace monitoring \
  --set allInOne.enabled=true \   # single-process for dev
  --set storage.type=memory       # use elasticsearch/cassandra in prod
```

Access the Jaeger UI:
```bash
kubectl port-forward svc/jaeger-query 16686:16686 -n monitoring
# Open http://localhost:16686
```

**Tempo vs Jaeger**: Prefer Tempo for Grafana-native workflows and cheap storage. Use Jaeger if you need its mature UI or existing ecosystem.

---

## 7. Grafana Dashboard Patterns

### Import community dashboards
In Grafana > Dashboards > Import, use dashboard IDs:
- `1860` — Node Exporter Full
- `13332` — Kubernetes cluster overview
- `15141` — Loki logs dashboard
- `17501` — VictoriaMetrics overview

### Provision dashboards via ConfigMap (GitOps-friendly)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-my-app
  namespace: monitoring
  labels:
    grafana_dashboard: "1"   # Grafana sidecar picks this up
data:
  my-app.json: |
    { ... paste dashboard JSON here ... }
```

---

## 8. Observability Stack Comparison

| | kube-prometheus-stack | VictoriaMetrics | Loki | Tempo | Jaeger |
|---|---|---|---|---|---|
| Type | Metrics + Alerts | Metrics + Alerts | Logs | Traces | Traces |
| Resource use | Medium-High | Low | Low | Low | Medium |
| Grafana native | Yes | Yes (PromQL) | Yes | Yes | Partial |
| Best for | Standard k8s | Pi/edge/cost | All envs | Grafana shops | Legacy/mature |
| Storage backend | Local PVC | Local/S3 | Local/S3 | Local/S3/GCS | Memory/ES/Cassandra |
