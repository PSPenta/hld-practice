# Deployment & Operations

How you ship and run systems: containers, orchestration, release strategies, and monitoring.

← [README](../README.md) · [Docs index](./README.md) · Diagram: [Logging and Monitoring](../diagrams/logging-and-monitoring-system/logging-and-monitoring-system.excalidraw)

---

## Index

- [Docker](#docker)
- [Kubernetes (K8s)](#kubernetes-k8s)
- [Deployment strategies](#deployment-strategies)
- [Monitoring: Prometheus, Grafana & ELK](#monitoring-prometheus-grafana-elk)
- [SLIs, SLOs, SLAs (ops vocabulary)](#slis-slos-slas-ops-vocabulary)
- [Other ops prerequisites](#other-ops-prerequisites)

---

## Docker

Packages app + dependencies into an **image**; runs as a **container**.

**Why it matters in HLD**

- Same artifact from laptop → CI → prod
- Horizontal scale = run more containers
- Resource limits (CPU/memory) per container

**Interview depth:** images, registries, env config, healthchecks — not Dockerfile golf.

---

## Kubernetes (K8s)

Orchestrates containers across machines: schedule, restart, scale, network, config.

Core objects to know by name:

| Object | Role |
|--------|------|
| **Pod** | Smallest run unit (one or more containers) |
| **Deployment** | Declarative replica set + rolling updates |
| **Service** | Stable virtual IP / DNS to pods |
| **Ingress** | HTTP routing from outside |
| **HPA** | Autoscale replicas on CPU/RPS/custom metrics |
| **ConfigMap / Secret** | Config and sensitive config |

**When to mention:** multi-service systems needing rolling deploys, health-based restarts, autoscaling.  
**When not to:** don’t start a URL shortener design with “first we need K8s” — it’s an implementation detail under “stateless app tier.”

Related: **service mesh** (Istio/Linkerd) for mTLS, retries, traffic split — optional deep dive if interview goes there.

---

## Deployment strategies

| Strategy | How it works | Pros | Cons |
|----------|--------------|------|------|
| **Recreate** | Kill old, start new | Simple | Downtime |
| **Rolling** | Replace instances gradually | No downtime; K8s default | Mixed versions briefly |
| **Blue/Green** (= **Red/Black**) | Two full envs; flip all traffic | Instant rollback | ~2× resources |
| **Canary** | Small % → new version, then ramp | Low blast radius | Needs good metrics |
| **Shadow / dark** | Mirror traffic to new (users stay on old) | Safe validation | Extra cost; won’t catch all UX bugs |
| **Feature toggle** | Ship code dark; flag on/off per user/cohort | Deploy ≠ release | Flag debt / cleanup |

**Where traffic flips (not only DNS):** LB / Ingress / service mesh / API gateway weight rules (usual) · **DNS weighted records** (coarse; TTL slows rollback) · feature flag inside the app (no traffic split needed).

Tie to NFRs: payments → canary + fast rollback; internal tools → rolling is fine.

---

## Monitoring: Prometheus, Grafana & ELK

### Prometheus

- **Pull-based** metrics scraper (apps expose `/metrics`)
- Time-series DB with labels (`service`, `route`, `status`)
- **PromQL** for queries; Alertmanager for alerts
- Great for: QPS, latency histograms, error rate, saturation (USE/RED)

**RED:** Rate, Errors, Duration  
**USE:** Utilization, Saturation, Errors (resources)

### Grafana

- Dashboards & visualization over Prometheus (and Loki, Tempo, …)
- On-call views: SLOs, burn rates, dependency health
- Can plot **metrics** well; not a replacement for deep **log search** (that’s Kibana / similar)

### ELK stack

**ELK** = **E**lasticsearch + **L**ogstash + **K**ibana (often + Beats/Fluent Bit as shippers). OpenSearch + Data Prepper/Logstash-style pipelines is the common fork.

| Piece | Role |
|-------|------|
| **Beats / agents** | Ship logs/metrics from hosts/pods |
| **Logstash** (or equivalent) | Parse, enrich, buffer, route |
| **Elasticsearch** | Index & search log documents (inverted index) |
| **Kibana** | Explore logs, dashboards, saved searches |

Typical HLD path: `App → stdout/file → agent → (Kafka optional) → Logstash → ES → Kibana`.

**Good at:** “What happened for `requestId=…`?”, audit trails, error text search, security/compliance log retention.  
**Watch:** storage cost, cardinality of fields, PII in logs, cluster ops.

Managed cousins: Elastic Cloud, OpenSearch Service, Datadog Logs, CloudWatch Logs — same *role* (log platform).

### ELK vs Prometheus & Grafana

They solve **different signals**. Don’t pick one as “the monitoring tool.”

| | **ELK (or Loki + UI)** | **Prometheus + Grafana** |
|--|------------------------|---------------------------|
| Primary signal | **Logs** (events, text) | **Metrics** (numbers over time) |
| Query style | Full-text / structured log search | PromQL (rate, histogram_quantile) |
| Alerting | Possible, but heavier; often secondary | First-class (Alertmanager, SLO burn) |
| Cost at scale | Dominated by log volume & retention | Dominated by series cardinality |
| Debug story | Reconstruct a single request’s trail | See fleet-wide latency/error **rates** |
| HLD box | “Log pipeline + search” | “Metrics + dashboards + alerts” |

| Question in the interview | Prefer |
|---------------------------|--------|
| p99 latency, error rate, CPU, saturation | **Prometheus → Grafana** (+ Alertmanager) |
| Find exception stack / user id / audit line | **ELK** (or Loki) |
| On-call “is payment SLO burning?” | **Metrics** first |
| On-call “why did this charge fail?” | **Logs** (+ traces) |

**Loki** (with Grafana): log aggregation with label-based indexing — lighter/cheaper than classic ELK for many K8s shops; weaker arbitrary full-text than ES. Mention as ELK alternative.

**Interview line:** “Prometheus/Grafana for **metrics and SLO alerts**; ELK for **log search**. Grafana doesn’t replace Kibana for deep log forensics. Draw metrics + logs + traces.”

### Full observability trio

| Signal | Examples | Tools (typical) |
|--------|----------|-----------------|
| Metrics | QPS, p99, CPU | Prometheus + Grafana |
| Logs | Errors, audit | **ELK** / OpenSearch / Loki |
| Traces | Request across services | Jaeger / Tempo / Zipkin |

HLD: draw **metrics + logs + traces + alerts**. See also [logging diagram](../diagrams/logging-and-monitoring-system/logging-and-monitoring-system.excalidraw).

---

## SLIs, SLOs, SLAs (ops vocabulary)

- **SLI** — measured indicator (e.g. availability = successful / total)
- **SLO** — target (99.9% monthly)
- **SLA** — contractual promise (often looser than SLO)

Error budgets justify canaries and release freezes.

---

## Other ops prerequisites

- **CI/CD** — build → test → deploy pipeline
- **Infrastructure as Code** — Terraform / CloudFormation (mention lightly)
- **Secrets management** — KMS, Vault; rotate credentials
- **Chaos / game days** — optional senior signal
- **Runbooks** — what on-call does when alert fires
- **Multi-AZ / multi-region DR** — RPO/RTO stated explicitly
