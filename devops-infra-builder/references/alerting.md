# Alerting Reference

Use this file when implementing alert routing, notification channels, or on-call workflows.

## Table of Contents
1. Alertmanager (Prometheus-native)
2. PrometheusRule — defining alerts
3. PagerDuty integration
4. Slack integration
5. ntfy (free, self-hosted push notifications)
6. Grafana OnCall (free tier)
7. Dead man's switch (watchdog alert)
8. Alerting tool comparison

---

## 1. Alertmanager

Alertmanager receives alerts from Prometheus, deduplicates, groups, and routes to receivers.

When using `kube-prometheus-stack`, Alertmanager is already deployed. Configure via values:

`infra/platform/alerting/alertmanager-values.yaml`:
```yaml
alertmanager:
  config:
    global:
      resolve_timeout: 5m

    route:
      group_by: ["namespace", "alertname"]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: "default"
      routes:
        - match:
            severity: critical
          receiver: pagerduty
        - match:
            severity: warning
          receiver: slack

    receivers:
      - name: "default"
        slack_configs:
          - api_url: "https://hooks.slack.com/services/XXX/YYY/ZZZ"
            channel: "#alerts"
            title: "{{ .GroupLabels.alertname }}"
            text: "{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}"

      - name: "pagerduty"
        pagerduty_configs:
          - routing_key: "<PAGERDUTY_INTEGRATION_KEY>"

      - name: "slack"
        slack_configs:
          - api_url: "https://hooks.slack.com/services/XXX/YYY/ZZZ"
            channel: "#alerts-warning"
```

Apply as a Helm override:
```bash
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring -f infra/platform/alerting/alertmanager-values.yaml
```

Or create an `AlertmanagerConfig` CRD for namespace-scoped routing:
```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: my-app-alerts
  namespace: default
spec:
  route:
    receiver: slack-myapp
    matchers:
      - name: namespace
        value: default
  receivers:
    - name: slack-myapp
      slackConfigs:
        - apiURL:
            name: slack-webhook
            key: url
          channel: "#my-app-alerts"
```

---

## 2. PrometheusRule — Defining Alerts

```yaml
# infra/platform/alerting/rules-my-app.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-rules
  namespace: default
  labels:
    release: kube-prometheus-stack   # must match Prometheus ruleSelector
spec:
  groups:
    - name: my-app
      interval: 1m
      rules:
        - alert: HighErrorRate
          expr: |
            rate(http_requests_total{job="my-app", status=~"5.."}[5m])
            /
            rate(http_requests_total{job="my-app"}[5m]) > 0.05
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "High error rate on my-app"
            description: "Error rate is {{ $value | humanizePercentage }} over the last 5m"

        - alert: PodCrashLooping
          expr: kube_pod_container_status_restarts_total{namespace="default"} > 3
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"

        - alert: HighMemoryUsage
          expr: |
            container_memory_usage_bytes{namespace="default"}
            / container_spec_memory_limit_bytes{namespace="default"} > 0.9
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Container {{ $labels.container }} using >90% memory limit"
```

### Verify rules loaded
```bash
kubectl get prometheusrule -A
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
# Open http://localhost:9090/alerts
```

---

## 3. PagerDuty Integration

### Create a PagerDuty integration
1. In PagerDuty: Services > your service > Integrations > Add Integration > Prometheus
2. Copy the **Integration Key**

### Alertmanager config
```yaml
receivers:
  - name: pagerduty
    pagerduty_configs:
      - routing_key: "<INTEGRATION_KEY>"
        severity: "{{ .CommonLabels.severity }}"
        description: "{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}"
        details:
          namespace: "{{ .GroupLabels.namespace }}"
          runbook: "{{ .CommonAnnotations.runbook_url }}"
```

Store the key as a Kubernetes Secret and reference it:
```yaml
pagerduty_configs:
  - routing_key_secret:
      name: pagerduty-secret
      key: routing-key
```

---

## 4. Slack Integration

### Create a Slack Incoming Webhook
Slack App > Incoming Webhooks > Add New Webhook > Copy URL.

### Alertmanager config
```yaml
receivers:
  - name: slack
    slack_configs:
      - api_url: "https://hooks.slack.com/services/T.../B.../XXX"
        channel: "#alerts"
        send_resolved: true
        title: |-
          [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.alertname }}
        text: >-
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Severity:* {{ .Labels.severity }}
          *Namespace:* {{ .Labels.namespace }}
          {{ end }}
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
```

---

## 5. ntfy (Free Self-Hosted Push Notifications)

ntfy is a free, open-source push notification service. Send alerts to your phone without PagerDuty costs.

**Experimental note**: ntfy is not natively supported by Alertmanager. Use a webhook receiver + a small relay or the `ntfy-alertmanager` sidecar.

### Run ntfy in your cluster
```yaml
# infra/platform/alerting/ntfy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ntfy
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ntfy
  template:
    metadata:
      labels:
        app: ntfy
    spec:
      containers:
        - name: ntfy
          image: binwiederhier/ntfy:latest
          args: ["serve"]
          ports:
            - containerPort: 80
          volumeMounts:
            - name: ntfy-data
              mountPath: /var/cache/ntfy
      volumes:
        - name: ntfy-data
          persistentVolumeClaim:
            claimName: ntfy-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: ntfy
  namespace: monitoring
spec:
  selector:
    app: ntfy
  ports:
    - port: 80
      targetPort: 80
```

### Alertmanager webhook to ntfy
```yaml
receivers:
  - name: ntfy
    webhook_configs:
      - url: "http://ntfy/alerts"
        send_resolved: true
```

Subscribe on your phone: install the ntfy app, subscribe to `http://your-ntfy-host/alerts`.

---

## 6. Grafana OnCall (Free Tier)

Grafana OnCall is an on-call management tool with a generous free tier (up to 5 users).

**Experimental note**: Grafana OnCall self-hosted requires Grafana 9.1+ and a Redis sidecar. The cloud version (grafana.com) is simpler to set up.

### Cloud setup (recommended to start)
1. Go to grafana.com > OnCall
2. Connect your Grafana instance
3. Create an integration: Alertmanager > Copy the webhook URL
4. Add as Alertmanager receiver:
```yaml
receivers:
  - name: grafana-oncall
    webhook_configs:
      - url: "https://oncall-prod-us-central-0.grafana.net/integrations/v1/alertmanager/XXX/"
        send_resolved: true
```

---

## 7. Dead Man's Switch (Watchdog Alert)

Always add a watchdog alert — an alert that fires when Prometheus is healthy. If it stops firing, your alerting pipeline is broken.

```yaml
- alert: Watchdog
  expr: vector(1)
  labels:
    severity: none
  annotations:
    summary: "Alerting pipeline is functional"
    description: "This alert is always firing to confirm the pipeline works end-to-end."
```

Route the watchdog to a separate receiver (e.g., a dedicated Slack channel or ntfy topic) so silence = problem.

---

## 8. Alerting Tool Comparison

| | Alertmanager | PagerDuty | Slack | ntfy | Grafana OnCall |
|---|---|---|---|---|---|
| Cost | Free | Paid (~$20/user/mo) | Free (webhook) | Free (self-hosted) | Free (≤5 users) |
| On-call scheduling | No | Yes | No | No | Yes |
| Escalation policies | No | Yes | No | No | Yes |
| Mobile push | No | Yes | Yes | Yes | Yes |
| Setup complexity | Low | Low | Low | Medium | Medium |
| Best for | Routing hub | Production on-call | Team notifications | Personal/homelab | Small teams |
