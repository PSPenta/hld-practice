# Reliability & SLOs

Staff-level designs talk about **what “good” means**, how you fail, and how far damage spreads — not only which database you picked.

← [README](../README.md) · [Docs index](./README.md) · Related: [Deployment & ops](./deployment-and-ops.md) · [Distributed coordination](./distributed-coordination.md)

---

## Index

- [SLI, SLO, SLA](#sli-slo-sla)
- [Error budgets](#error-budgets)
- [Failure modes checklist](#failure-modes-checklist)
- [Blast radius](#blast-radius)
- [Load shedding & graceful degradation](#load-shedding-graceful-degradation)
- [Multi-AZ vs multi-region](#multi-az-vs-multi-region)
- [Exponential backoff](#exponential-backoff)
- [Backpressure](#backpressure)
- [Reliability in fintech (Razorpay-class)](#reliability-in-fintech-razorpay-class)

---

## SLI, SLO, SLA

| Term | Meaning | Example |
|------|---------|---------|
| **SLI** | Metric you measure | Successful charges / total charge attempts |
| **SLO** | Internal target on an SLI | 99.95% success over 30 days |
| **SLA** | Customer/contract promise | Often looser than SLO (credits if breached) |

**RED (requests):** Rate, Errors, Duration (latency)  
**USE (resources):** Utilization, Saturation, Errors  

In an interview, propose 1–2 SLOs early (“payment authorize p99 &lt; 300ms”, “availability 99.95%”) — they justify caching, multi-AZ, and load shedding.

---

## Error budgets

If SLO is 99.9% monthly ≈ 43 minutes downtime budget.

- Budget healthy → ship features, canaries OK  
- Budget burned → freeze risky deploys, fix reliability  

Staff signal: connect **canary / feature flags** to error budget, not vibes.

---

## Failure modes checklist

Walk at least these when deep-diving:

| Failure | Design response |
|---------|-----------------|
| Instance death | LB health checks, replicas |
| AZ loss | Multi-AZ; no single-AZ stateful dependency |
| Dependency slow/down | Timeouts, retries with jitter, circuit breaker, fallback |
| Poison message | Retry limit → DLQ → alert |
| Hot key / thundering herd | Jitter, singleflight, shed load |
| Bad deploy | Canary, instant rollback, feature flag off |
| Data corruption | Backups, PITR, checksums, restore drill (RPO/RTO) |

Always answer: **What does the user see?** (degraded read-only, queue for tickets, fail closed on pay).

---

## Blast radius

How much of the system one fault can take down.

**Reduce blast radius**

- Bulkheads / separate pools per dependency  
- Per-tenant or per-shard isolation  
- Cell-based / pod-based architecture (many small failure domains)  
- Careful shared caches and global rate-limiters  

Staff interviewers listen for “this failure stays inside cell X.”

---

## Load shedding & graceful degradation

When overloaded, **deliberately drop or simplify** work:

- Return 503 on non-critical paths; protect checkout/pay  
- Serve stale cache; hide recommendations  
- Disable expensive fan-out; switch celebs to fan-out-on-read  
- Waiting room / queue for flash sales (BookMyShow-style)

Prefer controlled degradation over uncontrolled collapse.

---

## Multi-AZ vs multi-region

| | **Multi-AZ** | **Multi-region** |
|--|--------------|------------------|
| Protects against | Data-center zone loss | Region / disaster |
| Latency | Same-region | Cross-region cost |
| Consistency | Easier to stay strongly consistent | Often eventual or regional primary |
| Cost / complexity | Baseline for serious prod | Justify with RPO/RTO / user geo |

**Active-passive:** primary region + failover (simpler, RTO minutes).  
**Active-active:** write locally, conflict rules / CRDTs / regional keys (harder).

State **RPO** (data loss tolerance) and **RTO** (time to recover) when DR comes up.

---

## Exponential backoff

Retry transient failures with **increasing delay** so a sick dependency isn’t hammered by synchronized clients.

### Formula (typical)

```text
delay = min(cap, base * 2^attempt) + random_jitter
```

Example: base 100ms → 100, 200, 400, 800… capped at e.g. 30s, with **full or equal jitter**.

| Idea | Why |
|------|-----|
| **Exponential** | Space out retries as failure persists |
| **Cap (max delay)** | Bound worst-case wait |
| **Max attempts** | Then fail / DLQ / alert — don’t retry forever |
| **Jitter** | Break thundering herds when many clients retry together |
| **Idempotent ops only** | Retries on non-idempotent charges → double pay |

### When to use

- HTTP 429 / 503, timeouts, connection resets  
- Queue consumers after transient broker/DB errors  
- Webhook delivery to merchants  

### When **not** to blind-retry

- **4xx** (except 408/429) — fix the request  
- Unknown payment outcome without idempotency key — reconcile, don’t spam authorize  
- After circuit breaker **open** — wait for half-open, don’t keep backing off into a black hole  

### Pair with

- **Timeouts** (fail fast per attempt)  
- **Circuit breaker** (stop calling when error rate is high) — see [Core concepts](./core-concepts.md#circuit-breaker)  
- **DLQ** after N failures — see [Messaging](./messaging-and-pipelines.md)  

**Interview line:** “Retries with exponential backoff **and jitter**; idempotent; capped attempts; then DLQ.”

---

## Backpressure

Propagate “slow down” upstream: bounded queues, HTTP 429/503, Kafka consumer lag alerts, autoscale on lag — not infinite buffers that OOM at 3 a.m.

---

## Reliability in fintech (Razorpay-class)

- Fail **closed** on uncertain payment state; reconcile async  
- Exactly-once *effect* via idempotency keys + ledger, not magic middleware  
- Dual writes (DB + Kafka) are a reliability smell → outbox  
- Reconciliation workers as first-class design (see payment diagram)

---

## See also

- [Security & compliance](./security-and-compliance.md)  
- [Caching](./caching.md) — stampede/avalanche as reliability bugs  
- [Messaging & pipelines](./messaging-and-pipelines.md) — DLQ, at-least-once  
