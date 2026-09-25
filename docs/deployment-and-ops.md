# Deployment & Operations

How you ship and run systems: containers, orchestration, release strategies, and monitoring.

← [README](../README.md) · [Docs index](./README.md) · Diagram: [Logging and Monitoring](../diagrams/logging-and-monitoring-system/logging-and-monitoring-system.excalidraw)

---

## Index

- [Docker](#docker)
- [Kubernetes (K8s)](#kubernetes-k8s)
  - [Cluster, Node, Pod](#cluster-node-pod)
  - [Kubelet & Kube-proxy](#kubelet--kube-proxy)
  - [Workloads](#workloads)
  - [Service types & Ingress](#service-types--ingress)
  - [ConfigMap & Secret](#configmap--secret)
  - [Volumes, PV & PVC](#volumes-pv--pvc)
  - [Namespace](#namespace)
  - [Horizontal Pod Autoscaler (HPA)](#horizontal-pod-autoscaler-hpa)
  - [Probes, resources & Staff gotchas](#probes-resources--staff-gotchas)
  - [When to use what (cheat sheet)](#when-to-use-what-cheat-sheet)
- [Kubernetes vs Amazon ECS](#kubernetes-vs-amazon-ecs)
- [Deployment strategies](#deployment-strategies)
- [Monitoring: Prometheus, Grafana & ELK](#monitoring-prometheus-grafana-elk)
- [Prometheus & Grafana setup (detailed)](./prometheus-grafana-setup.md) — `/metrics` samples, Compose vs K8s, Grafana datasource
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

Orchestrates containers across machines: **schedule**, **restart**, **scale**, **network**, **config**, **storage**.

**Mental model**

```
Cluster
  └── Node (VM / bare metal) × N
        └── Pod × M
              └── Container × 1..k   ← your app + sidecars
```

Control plane (API server, scheduler, controller-manager, **etcd**) decides *desired state*.  
Each Node’s **Kubelet** makes reality match; **Kube-proxy** (or a CNI dataplane) implements Service networking.

**When to mention:** multi-service systems needing rolling deploys, health-based restarts, autoscaling, multi-tenant isolation.  
**When not to:** don’t open a URL-shortener HLD with “first we need K8s” — fold it under “stateless app tier.”

Related: **service mesh** (Istio/Linkerd) for mTLS, retries, traffic split — optional deep dive if the interview goes there.

---

### Cluster, Node, Pod

| Concept | What it is | Interview line |
|---------|------------|----------------|
| **Cluster** | One control plane + worker Nodes sharing a network/API | “One K8s cluster per env/region; multi-cluster for blast-radius / compliance” |
| **Node** | A machine that runs Pods (kubelet + container runtime + kube-proxy/CNI) | Capacity = sum of Node CPU/mem; cordon/drain for maintenance |
| **Pod** | Smallest deployable unit: 1+ containers sharing **network namespace** (same localhost) and optional volumes | Prefer **one main container** per Pod; sidecars for proxy/log/agent |

**Pod facts Staff expect**

- Pods are **ephemeral** — IP changes on reschedule; never hardcode Pod IPs (use **Service** DNS).
- Containers in a Pod start together, share `localhost`, and die together.
- Scheduling: scheduler picks a Node based on **requests**, affinity/taints, topology.
- You rarely create naked Pods in prod — controllers (**Deployment**, **StatefulSet**, …) own them.

---

### Kubelet & Kube-proxy

| Agent | Runs on | Job |
|-------|---------|-----|
| **Kubelet** | Every Node | Talks to API server; starts/stops containers via runtime (containerd); reports Node/Pod status; runs **liveness/readiness** probes; enforces cgroup limits |
| **Kube-proxy** | Every Node (classic) | Programs iptables/IPVS (or equivalent) so **Service** ClusterIPs/NodePorts reach Pod endpoints |

**Staff nuance**

- Many modern clusters replace kube-proxy’s dataplane with **CNI** features (Cilium, Calico eBPF) — role is the same: **Service → Pod** routing + load spread.
- If Kubelet is unhealthy → Node **NotReady** → Pods drain / stop scheduling there.
- Control plane pieces you name lightly: **API server** (all clients), **scheduler**, **controller-manager** (Deployment/ReplicaSet loops), **etcd** (cluster state). Don’t over-draw them in HLD unless asked.

---

### Workloads

Controllers reconcile *desired replica count / Pod template* → actual Pods.

#### Deployment & ReplicaSet

| Object | Role |
|--------|------|
| **ReplicaSet** | Ensures *N* identical Pods matching a selector/template. Low-level; you rarely manage it by hand |
| **Deployment** | Owns ReplicaSets; does **rolling updates**, rollbacks, scaling. Default for **stateless** APIs/workers |

**How a rolling update works:** Deployment creates a new ReplicaSet, ramps new Pods up / old Pods down (`maxSurge` / `maxUnavailable`). Old RS kept for rollback.

**Use Deployment when:** REST APIs, consumers, most microservices — any Pod interchangeable.

#### StatefulSet

Ordered, sticky identity for **stateful** workloads.

| Guarantee | Why it matters |
|-----------|----------------|
| Stable network identity | `pod-0`, `pod-1`, … + headless Service DNS |
| Stable storage | Each ordinal gets its own **PVC** (survives Pod reschedule) |
| Ordered deploy/scale/delete | Useful for quorum (ZK/etcd-ish), primary election helpers |

**Use StatefulSet when:** Kafka/ZooKeeper/etcd-style members, Redis with persistence per member, DB operators’ data Pods.  
**Not for:** normal stateless API — Deployment is simpler and rolls faster.

#### DaemonSet

Runs **exactly one Pod per (eligible) Node** (or per Node matching a selector).

**Use when:** log/metrics agents (Fluent Bit, node-exporter), CNI agents, security/runtime sensors, node-local caches.  
**Not for:** app replicas — that’s Deployment/HPA.

#### Job & CronJob

| Object | Role |
|--------|------|
| **Job** | Run Pod(s) to **completion** (batch). Retries until `completions` succeed or backoff limit |
| **CronJob** | Creates Jobs on a **cron schedule** (`0 * * * *`) |

**Use Job when:** migrations, one-shot ETL, report generation, reindex.  
**Use CronJob when:** periodic cleanup, digest emails, reconciliation ticks (prefer app-level schedulers / Temporal for complex workflows — see [service architecture](./service-architecture.md#cron-scheduler-vs-temporal)).

**Interview pitfall:** long-running APIs are **not** Jobs; crashing batch work should be a Job, not a Deployment that “exits.”

---

### Service types & Ingress

**Service** = stable virtual IP + DNS name in front of a **set of Pods** (label selector). Endpoints update as Pods come and go.

| Type | How clients reach Pods | Typical use |
|------|------------------------|-------------|
| **ClusterIP** (default) | Internal VIP only inside the cluster | Service-to-service (`http://payments.default.svc`) |
| **NodePort** | Opens a high port on **every Node**; forwards to ClusterIP | Lab / bare-metal; rarely preferred in cloud |
| **LoadBalancer** | Cloud provider provisions an external LB → NodePort/ClusterIP | Public or internal cloud LB for one Service |
| **Headless** (`clusterIP: None`) | DNS returns **Pod IPs** directly | StatefulSet peers, client-side LB |

**Ingress** = L7 HTTP/HTTPS routing **into** the cluster (host/path rules → Services). Needs an **Ingress controller** (NGINX, ALB, Traefik, …).

| Layer | Box on the HLD board |
|-------|----------------------|
| External users | DNS → cloud LB / Ingress controller |
| Ingress | TLS terminate, path `/api` → `api-svc`, `/admin` → `admin-svc` |
| Service (ClusterIP) | Stable name for Pods of one Deployment |
| Pods | Actual containers |

**Staff lines**

- East-west (service ↔ service): **ClusterIP** (+ mesh optional).  
- North-south (internet → app): **Ingress** or **LoadBalancer**; Ingress usually wins when many host/path rules share one LB.  
- gRPC/WebSocket: confirm Ingress/controller support (timeouts, sticky sessions if needed).  
- Don’t confuse **Ingress** (K8s object) with “ingress traffic” vocabulary in [staff-vocabulary](./staff-vocabulary.md#traffic--capacity).

---

### ConfigMap & Secret

| Object | Holds | Mounted as |
|--------|-------|------------|
| **ConfigMap** | Non-sensitive config (feature flags, hostnames, file templates) | Env vars or volumes |
| **Secret** | Sensitive data (tokens, passwords, TLS keys) — base64 in etcd by default | Env vars or volumes |

**Staff expectations**

- Prefer **volume mounts** for secrets (less likely to leak via process listing / crash logs than env).
- Default Secret encryption-at-rest is weak unless **encryption providers** / external **Vault / KMS / cloud secret stores** (External Secrets Operator pattern).
- Changing a ConfigMap does **not** always restart Pods — use checksum annotations or restart Reloader; treat config rollout like a deploy.
- Never bake prod secrets into images.

---

### Volumes, PV & PVC

| Concept | Role |
|---------|------|
| **Volume** | Storage attached to a **Pod** (emptyDir, configMap, secret, PVC, hostPath, …). Lifetime often tied to Pod |
| **PersistentVolume (PV)** | Cluster-level piece of real storage (EBS, PD, NFS, …) provisioned by admin or dynamically |
| **PersistentVolumeClaim (PVC)** | Pod/StatefulSet **request** for storage (“10Gi, ReadWriteOnce”) — bound to a PV |

**Binding flow:** Pod references PVC → PVC binds PV → Node attaches disk → volume mounted into container.

| Access mode | Meaning |
|-------------|---------|
| **ReadWriteOnce (RWO)** | One Node at a time (typical cloud block disk) |
| **ReadOnlyMany (ROX)** | Many Nodes read-only |
| **ReadWriteMany (RWX)** | Many Nodes read-write (NFS / shared FS) |

**Interview lines**

- Stateless API: **no PVC** — local disk is ephemeral; state in DB/Redis/S3.  
- StatefulSet: `volumeClaimTemplates` → one PVC per ordinal.  
- `emptyDir`: scratch space; **gone when Pod is removed**.  
- Losing a PVC ≠ losing a Deployment replica — call out **RPO** for disk-backed state.

---

### Namespace

Virtual cluster slice for **isolation of names** (and often RBAC, quotas, network policies).

| Use | Example |
|-----|---------|
| Environments | `dev`, `staging`, `prod` (or separate clusters for prod) |
| Teams / domains | `payments`, `identity`, `data` |
| Soft multi-tenancy | Quotas + NetworkPolicies per namespace |

DNS form: `service.namespace.svc.cluster.local`.  
**Not** a hard security boundary by itself — pair with **RBAC**, **NetworkPolicy**, and preferably separate clusters for hostile tenants.

---

### Horizontal Pod Autoscaler (HPA)

Automatically sets Deployment/ReplicaSet/StatefulSet **replica count** from metrics.

| Signal | Common |
|--------|--------|
| CPU / memory | Resource metrics (default) |
| Custom | RPS, queue depth, lag (custom/external metrics APIs) |

**Staff gotchas**

- HPA scales **Pods**, not the database — scaling app replicas into a saturated primary makes things worse ([scaling](./scaling.md)).
- Needs correct **resource requests** (HPA CPU % is vs requests).
- Pair with **Cluster Autoscaler** / node pools when Nodes are full (HPA alone can’t create Nodes).
- Cool-downs / flapping: scale-up fast, scale-down slow.
- Vertical Pod Autoscaler (VPA) adjusts **requests/limits** — mention only if asked; don’t run naive HPA+VPA on the same metrics without care.

---

### Probes, resources & Staff gotchas

| Probe | Purpose |
|-------|---------|
| **Liveness** | Restart container if deadlocked / wedged (**process health**) |
| **Readiness** | Remove from Service endpoints until warm — deps up, migrations done, cache loaded (**ready for traffic**) |
| **Startup** | Slow-start apps; delay liveness until boot finishes |

**Readiness ≠ “is the process alive.”** A live Pod can fail readiness → **no new traffic**, but not restarted. Liveness fail → **restart**.

| Resource field | Effect |
|----------------|--------|
| **requests** | Scheduler + HPA baseline; guarantees |
| **limits** | Cap; CPU throttle / OOM kill if exceeded |

**PodDisruptionBudget (PDB):** limit voluntary disruptions (drains, upgrades) so you keep `minAvailable` healthy Pods — Staff signal for HA APIs.

---

### When to use what (cheat sheet)

| Need | Prefer |
|------|--------|
| Stateless API / worker | **Deployment** + **ClusterIP** (+ **Ingress** if public HTTP) |
| One agent per Node | **DaemonSet** |
| Disk + stable identity (brokers, DB members) | **StatefulSet** + PVC + headless Service |
| One-shot batch | **Job** |
| Periodic batch | **CronJob** |
| Scale on load | **HPA** (+ node autoscaler if needed) |
| Config / passwords | **ConfigMap** / **Secret** (or external secret store) |
| Shared cluster isolation | **Namespace** (+ RBAC / NetworkPolicy) |
| Node agent unhealthy | Think **Kubelet**; Service routing → **Kube-proxy**/CNI |

**HLD drawing order:** Clients → Ingress/LB → Service → Deployment Pods → data stores. Name StatefulSet/DaemonSet/Job only when the workload type forces it.

---

## Kubernetes vs Amazon ECS

Both **schedule containers across a cluster** (tasks/Pods on nodes/instances). K8s is **not** “pods inside one VM while ECS manages whole machines.”

| | **Amazon ECS** | **Kubernetes** |
|--|----------------|----------------|
| Scope | AWS-native orchestrator (EC2 or **Fargate**) | Portable control plane (cloud or on-prem) |
| Abstraction | Task / Service / Cluster | Pod / Deployment / Service / Ingress + CRDs |
| Networking / probes | AWS wiring; health checks | Rich Service/Ingress model; **liveness vs readiness** first-class |
| Extensibility | AWS features | Huge ecosystem (operators, mesh, custom resources) |
| **What you gain with K8s** | — | Portability, finer scheduling, community patterns, multi-cloud option |
| **What it costs you** | Less DIY on AWS | Control-plane ops (etcd, upgrades), networking complexity, steeper learning |

**When ECS is enough:** AWS-only team, want managed simplicity (often + Fargate).  
**When K8s is worth it:** multi-cloud/on-prem, need custom controllers, or org standardizes on K8s.

**Interview line:** “ECS and K8s both place containers on a fleet. K8s buys portability and primitives; you pay operational complexity. Readiness controls **traffic**; liveness controls **restarts**.”

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

HLD-level roles and when to pick metrics vs logs. **Wiring, `/metrics` samples, Compose vs Kubernetes, Grafana datasource:** [Prometheus & Grafana setup (detailed reference)](./prometheus-grafana-setup.md).

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

Setup detail (scrape config, ServiceMonitor, sample export): [Prometheus & Grafana setup](./prometheus-grafana-setup.md).

### Full observability trio

| Signal | Examples | Tools (typical) |
|--------|----------|-----------------|
| Metrics | QPS, p99, CPU | Prometheus + Grafana |
| Logs | Errors, audit | **ELK** / OpenSearch / Loki |
| Traces | Request across services | Jaeger / Tempo / Zipkin |

HLD: draw **metrics + logs + traces + alerts**. See also [logging diagram](../diagrams/logging-and-monitoring-system/logging-and-monitoring-system.excalidraw).

---

## SLIs, SLOs, SLAs (ops vocabulary)

- **SLI** — measured indicator (e.g. successful / total — availability is *one* SLI, not the word’s definition)
- **SLO** — target on that SLI (99.9% monthly)
- **SLA** — contractual promise (often looser than SLO)
- **Error budget** — `1 − SLO`; when exhausted → **freeze risky deploys**, fix reliability (rollback is a tool, not the definition)

Full write-up: [Reliability & SLOs](./reliability-and-slos.md).

---

## Other ops prerequisites

- **CI/CD** — build → test → deploy pipeline
- **Infrastructure as Code** — Terraform / CloudFormation (mention lightly)
- **Secrets management** — KMS, Vault; rotate credentials
- **Chaos / game days** — optional senior signal
- **Runbooks** — what on-call does when alert fires
- **Multi-AZ / multi-region DR** — RPO/RTO stated explicitly
