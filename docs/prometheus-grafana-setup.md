# Prometheus & Grafana setup (detailed reference)

How metrics actually wire up: `/metrics` payload, scrape config, Grafana datasources, **with and without Kubernetes**. HLD interview narrative stays in [Deployment & ops — Monitoring](./deployment-and-ops.md#monitoring-prometheus-grafana-elk); this doc is the **ops deep dive**.

← [README](../README.md) · [Docs index](./README.md) · HLD view: [Deployment & ops](./deployment-and-ops.md#monitoring-prometheus-grafana-elk) · Diagram: [Logging and Monitoring](../diagrams/logging-and-monitoring-system/logging-and-monitoring-system.excalidraw)

---

## Index

- [End-to-end flow](#end-to-end-flow)
- [What `/metrics` returns](#what-metrics-returns)
- [Sample `/metrics` export](#sample-metrics-export)
- [Backend vs frontend](#backend-vs-frontend)
- [Without Kubernetes](#without-kubernetes)
- [With Kubernetes](#with-kubernetes)
- [Configuring Grafana](#configuring-grafana)
- [Alertmanager (optional)](#alertmanager-optional)
- [Other common tools](#other-common-tools)
- [Interview vs ops depth](#interview-vs-ops-depth)

---

## End-to-end flow

```
App (instrumented)
  └── GET /metrics   ← Prometheus exposition text
        ↑ scrape every 15–30s
Prometheus (TSDB + PromQL + optional Alertmanager)
        ↑ Grafana datasource query
Grafana (dashboards / panels)
```

- **Prometheus pulls** the app (default). Apps rarely push to Prometheus unless you use **remote_write** / Pushgateway (batch jobs).
- **Grafana does not scrape** `/metrics`; it queries Prometheus (or Loki, Tempo, …).

---

## What `/metrics` returns

| Property | Value |
|----------|--------|
| Path | Usually `/metrics` (configurable) |
| Method | `GET` |
| Content-Type | `text/plain; version=0.0.4` (classic) or OpenMetrics |
| Body | **Not JSON** — line-oriented Prometheus text |

**Metric types**

| Type | Meaning | Example use |
|------|---------|-------------|
| **counter** | Only goes up (resets on restart) | `http_requests_total` |
| **gauge** | Up or down | `db_pool_in_use`, temperature |
| **histogram** | Observations in buckets (+ `_sum`, `_count`) | Latency, payload size |
| **summary** | Client-side quantiles (less common than histogram) | Legacy latency |

Lines starting with `# HELP` / `# TYPE` are comments. Labels are `{key="value"}` → **cardinality** (too many unique label values = expensive).

**Libraries (typical):** `prometheus/client_golang`, `prom-client` (Node), `prometheus_client` (Python), Micrometer, OpenTelemetry → Prometheus exporter.

---

## Sample `/metrics` export

What a scrape might receive from a Go/Node API (truncated):

```text
# HELP http_requests_total Total HTTP requests processed
# TYPE http_requests_total counter
http_requests_total{method="GET",route="/v1/orders",code="200"} 12840
http_requests_total{method="GET",route="/v1/orders",code="500"} 12
http_requests_total{method="POST",route="/v1/orders",code="201"} 902

# HELP http_request_duration_seconds HTTP request latency
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="GET",route="/v1/orders",le="0.005"} 4200
http_request_duration_seconds_bucket{method="GET",route="/v1/orders",le="0.01"} 8100
http_request_duration_seconds_bucket{method="GET",route="/v1/orders",le="0.05"} 12000
http_request_duration_seconds_bucket{method="GET",route="/v1/orders",le="0.1"} 12600
http_request_duration_seconds_bucket{method="GET",route="/v1/orders",le="0.5"} 12800
http_request_duration_seconds_bucket{method="GET",route="/v1/orders",le="+Inf"} 12840
http_request_duration_seconds_sum{method="GET",route="/v1/orders"} 312.4
http_request_duration_seconds_count{method="GET",route="/v1/orders"} 12840

# HELP db_pool_in_use Open DB connections in pool
# TYPE db_pool_in_use gauge
db_pool_in_use{pool="primary"} 14

# HELP go_goroutines Number of goroutines (runtime)
# TYPE go_goroutines gauge
go_goroutines 87

# HELP process_cpu_seconds_total Total user and system CPU time
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 441.2
```

**PromQL you’d use later in Grafana**

```promql
# QPS
sum(rate(http_requests_total[5m])) by (route)

# Error rate
sum(rate(http_requests_total{code=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# p99 latency
histogram_quantile(
  0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route)
)
```

**Security:** bind `/metrics` to internal network / ClusterIP only; don’t expose it on the public Ingress. Prefer a separate listen port (e.g. `:9090/metrics`) if the public API is `:8080`.

---

## Backend vs frontend

| | **Backend / API / worker** | **Browser SPA** | **FE Node (SSR / BFF)** |
|--|---------------------------|-----------------|-------------------------|
| Prometheus scrapes app? | **Yes** — `/metrics` on pods/VMs | **No** | Yes, if the Node process exposes it |
| What you measure | RED/USE, queues, DB, deps | Web Vitals, JS errors, client API latency | SSR latency + same as BE |
| Path to Prometheus | Direct scrape (or mesh sidecar) | RUM SDK → **collector/ingest API** → that service exports metrics or remote_writes | Scrape BFF like BE |
| HLD box | App → Prometheus → Grafana | Browser → telemetry collector → metrics backend | Same as BE |

**Frontend pattern:** `web-vitals` / OpenTelemetry browser SDK → your `telemetry-ingest` service (or Datadog/New Relic) → metrics. Prometheus never dials the user’s laptop.

---

## Without Kubernetes

Minimal local / VM / Docker Compose layout.

### 1. App

Expose metrics on `0.0.0.0:8080/metrics` (or `:9090`).

### 2. `prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]   # optional

rule_files:
  - /etc/prometheus/rules/*.yml            # optional recording/alert rules

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]        # Prometheus scrapes itself

  - job_name: payments-api
    metrics_path: /metrics
    scrape_interval: 15s
    static_configs:
      - targets: ["payments-api:8080"]
        labels:
          env: "dev"
          service: "payments-api"

  - job_name: node
    static_configs:
      - targets: ["node-exporter:9100"]    # host CPU/disk (optional)
```

Run Prometheus with this file mounted, e.g. `prom/prometheus --config.file=/etc/prometheus/prometheus.yml`.

**Verify:** open Prometheus UI → **Status → Targets** (all UP) → **Graph** and paste a PromQL expression.

### 3. Docker Compose sketch

```yaml
services:
  payments-api:
    image: myorg/payments-api:latest
    ports: ["8080:8080"]

  prometheus:
    image: prom/prometheus:v2.54.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports: ["9090:9090"]
    depends_on: [payments-api]

  grafana:
    image: grafana/grafana:11.2.0
    ports: ["3000:3000"]
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    depends_on: [prometheus]
```

Network DNS names (`payments-api`, `prometheus`) must match `targets` in `prometheus.yml`.

### 4. File-based service discovery (many VMs)

```yaml
scrape_configs:
  - job_name: api
    file_sd_configs:
      - files: ["/etc/prometheus/targets/*.json"]
        refresh_interval: 1m
```

`targets/payments.json`:

```json
[
  {
    "targets": ["10.0.1.11:8080", "10.0.1.12:8080"],
    "labels": { "service": "payments-api", "env": "prod" }
  }
]
```

---

## With Kubernetes

Same pull model; **targets are discovered** from the API server instead of static IPs.

### Option A — Pod / Service annotations (simple)

Pod template:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
spec:
  containers:
    - name: api
      ports:
        - name: http
          containerPort: 8080
```

Works if your Prometheus chart is configured to honor these annotations (common community setup). Prefer Operator CRDs in serious clusters (below).

### Option B — Prometheus Operator `ServiceMonitor` (usual prod)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payments-api
  labels:
    app: payments-api
spec:
  selector:
    app: payments-api
  ports:
    - name: http
      port: 8080
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payments-api
  labels:
    release: kube-prometheus-stack   # must match Prometheus Operator selector
spec:
  selector:
    matchLabels:
      app: payments-api
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

**kube-prometheus-stack** (Helm): installs Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics in one chart. Your app only needs metrics + ServiceMonitor.

### Option C — Sidecar / mesh

Istio/Linkerd can emit golden metrics (request rate/error/latency) without app changes; still often keep **app business metrics** on `/metrics`.

### Network notes (K8s)

| Concern | Practice |
|---------|----------|
| Reachability | Prometheus scrapes **Pod IP** or Service; same cluster NetworkPolicy must allow it |
| Auth | Optional bearer token / mTLS for scrape; don’t put metrics on public Ingress |
| Multi-cluster | Per-cluster Prometheus + **Thanos** / Mimir / Cortex for global query (Staff depth) |

---

## Configuring Grafana

### 1. Add Prometheus as a data source

**UI:** Connections → Data sources → Add → **Prometheus**

| Field | Example |
|-------|---------|
| URL | `http://prometheus:9090` (Compose) or `http://kube-prometheus-stack-prometheus.monitoring.svc:9090` (K8s) |
| Access | Server (default) — Grafana backend queries Prometheus |
| Auth | Usually none inside the cluster network |

**Click Save & test** → “Data source is working”.

### 2. Provisioning (IaC / Compose)

`grafana/provisioning/datasources/datasource.yml`:

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

### 3. Dashboard panel

1. Create dashboard → Add visualization → select **Prometheus**.
2. Query (PromQL), e.g. `sum(rate(http_requests_total[5m])) by (route)`.
3. Legend, unit (reqps / seconds), thresholds for on-call views.

Import community dashboards (Node Exporter Full, Go runtime, etc.) via **Dashboards → Import → ID**.

### 4. Grafana alerts (optional)

Grafana can alert from panel rules, **or** keep alerts in Prometheus/Alertmanager (classic SRE style). For HLD, say: “Prometheus rules → Alertmanager → PagerDuty/Slack; Grafana for human dashboards.”

---

## Alertmanager (optional)

Prometheus evaluates rules → fires to **Alertmanager** → groups, dedupes, routes to Slack/PagerDuty.

`rules/payments.yml` (loaded by Prometheus `rule_files`):

```yaml
groups:
  - name: payments
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{code=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: page
        annotations:
          summary: "Payments 5xx ratio > 5%"
```

---

## Other common tools

| Tool | Role vs Prom/Grafana |
|------|----------------------|
| **node-exporter** | Host CPU, disk, network → scraped like any target |
| **kube-state-metrics** | K8s object gauges (deployments, pods desired/ready) |
| **blackbox-exporter** | Probe HTTP/TCP/ICMP from outside the app |
| **Pushgateway** | Short-lived batch Jobs push metrics (use sparingly) |
| **Alertmanager** | Alert routing (above) |
| **Loki** | Logs; Grafana can add Loki as a **second** datasource |
| **Tempo / Jaeger** | Traces; Grafana explore traces ↔ metrics links |
| **OTel Collector** | Receive OTLP from apps → export to Prometheus remote_write / Tempo / Loki |
| **Mimir / Thanos / Cortex** | Long-term / HA / multi-tenant Prometheus storage |
| **Datadog / CloudWatch** | Managed substitutes; same *roles* (metrics UI + alerts) |

**OpenTelemetry path (modern):** app → OTLP → Collector → Prometheus remote_write → Grafana. Exposition may be OTLP instead of hand-rolled `/metrics`; Prometheus-compatible endpoints remain common.

---

## Interview vs ops depth

| In a 45-min HLD | Use this doc for |
|----------------|------------------|
| “Metrics: Prometheus scrape `/metrics` → Grafana dashboards; Alertmanager for pages” | Wiring, sample payload, Compose vs K8s ServiceMonitor |
| RED/USE + SLO burn | Exact PromQL and dashboard JSON |
| Don’t lead with Compose YAML | When interviewer asks “how does scrape discovery work?” |

**See also:** [Deployment & ops — Monitoring](./deployment-and-ops.md#monitoring-prometheus-grafana-elk) · [Reliability & SLOs](./reliability-and-slos.md) · [Building blocks — Observability](./building-blocks.md#observability-stack)
