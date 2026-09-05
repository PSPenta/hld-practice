# Deployment & Operations

How you ship and run systems: containers, orchestration, release strategies, and monitoring.

← [README](../README.md) · [Docs index](./README.md) · Diagram: [Logging and Monitoring](../diagrams/logging-and-monitoring-system/logging-and-monitoring-system.excalidraw)

---

## Index

- [Docker](#docker)
- [Kubernetes (K8s)](#kubernetes-k8s)
- [Deployment strategies](#deployment-strategies)
- [Monitoring: Prometheus & Grafana](#monitoring-prometheus-grafana)
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

## Monitoring: Prometheus & Grafana

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

### Full observability trio

| Signal | Examples | Tools (typical) |
|--------|----------|-----------------|
| Metrics | QPS, p99, CPU | Prometheus + Grafana |
| Logs | Errors, audit | Loki / ELK |
| Traces | Request across services | Jaeger / Tempo / Zipkin |

HLD: draw **metrics + logs + traces + alerts**; name Prometheus/Grafana when asked for concrete stack. See also [logging diagram](../diagrams/logging-and-monitoring-system/logging-and-monitoring-system.excalidraw).

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
