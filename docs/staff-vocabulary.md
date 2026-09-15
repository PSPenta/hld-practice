# Staff HLD Vocabulary

Crisp definitions for terms Staff interviewers expect you to use **correctly** — especially dependency direction, capacity, and overload behavior.

← [README](../README.md) · [Docs index](./README.md) · Related: [Soft skills](./soft-skills.md) · [Estimation](./estimation-fluency.md) · [Reliability & SLOs](./reliability-and-slos.md) · [Scaling](./scaling.md)

---

## Index

- [Upstream vs downstream (in your HLD)](#upstream-vs-downstream-in-your-hld)
- [Dependency chain on the board](#dependency-chain-on-the-board)
- [Traffic & capacity](#traffic-capacity)
- [Latency & overload](#latency-overload)
- [Isolation & failure spread](#isolation-failure-spread)
- [Queues & async paths](#queues-async-paths)
- [How to use this in a round](#how-to-use-this-in-a-round)

---

## Upstream vs downstream (in your HLD)

Direction follows **the request or data flow**, not org chart or “importance.”

| Term | Meaning | Example (Notification Service) |
|------|---------|--------------------------------|
| **Upstream** | Sends traffic **into** your service | Client → API Gateway → **Notification Service** (Gateway is upstream of Notification) |
| **Downstream** | Your service calls **out to** it | Notification Service → Kafka → **SMS Worker** → Exotel (Worker and Exotel are downstream of Notification) |
| **Your service** | The box you’re designing | Notification Service |

**Common mistake:** calling the database “upstream” because “data comes from DB.” In read paths the DB is downstream of the app (the app calls the DB). In CDC paths the DB change stream is upstream of the indexer (events flow DB → Kafka → search).

**Staff habit:** when you draw an arrow, say which dependency it is:

> “Payment Service is **upstream** of Ledger Worker; Stripe is **downstream** of Payment Service.”

---

## Dependency chain on the board

| Term | Staff meaning |
|------|----------------|
| **Critical path** | Longest serial chain on the user-facing request. Latency ≈ sum of hops (plus tail effects). Optimize here first |
| **Fan-out** | One request triggers N parallel downstream calls (e.g. timeline → 100 microservices). Tail latency dominates — see [Latency & metrics](./core-concepts.md#latency-metrics-vocabulary) |
| **Fan-in** | Many producers → one consumer (aggregators, batch indexers). Watch hot partitions and write amplification |
| **Sync dependency** | Caller waits. Failure/timeout blocks the user path |
| **Async dependency** | Hand off to queue/stream; user gets 202 or eventual result. Decouples latency but adds consistency work |
| **Hard dependency** | Cannot serve without it (auth, primary DB for write). Fail closed or queue |
| **Soft dependency** | Nice-to-have (recommendations, analytics). Degrade without failing core flow |
| **Dependency depth** | Number of hops. Staff designs **shallow critical paths** and push depth async |

**Example sentence:** “Checkout’s critical path is Gateway → Order → Payment → PSP; recommendations are a **soft downstream** — we timeout at 50ms and skip.”

---

## Traffic & capacity

| Term | Meaning | Staff usage |
|------|---------|-------------|
| **QPS / RPS / TPS** | Requests or transactions per second | “Peak write QPS drives shard count” |
| **Throughput** | Work completed per second (requests, bytes, messages) | Distinct from latency — high throughput can still have bad p99 |
| **Steady state vs peak** | Average load vs campaign / festival / launch spike | Design for **peak** (often 5–10× average); state both |
| **Headroom** | Spare capacity below saturation | “Run app tier ~70% CPU at peak so we have headroom for failover and deploys” |
| **Saturation** | Resource at limit (CPU, connections, disk IOPS, broker lag) | Past saturation, latency explodes (queueing) — name **which** resource saturates first |
| **Bottleneck** | The resource that caps the system | RPS vs CPU vs memory vs DB connections vs network — pick one and scale **that** |
| **Connection budget** | Max DB/cache sockets across all pods | Workers × pool size can exhaust Postgres — use pooler, separate pools per tier |
| **Ingress vs egress** | Traffic in vs out of a service | Rate-limit **ingress**; throttle **egress** to fragile downstream (PSP, SMS vendor) |
| **Goodput** | Useful throughput excluding retries/errors | Retry storms inflate raw QPS but not goodput |
| **Capacity plan** | Servers × per-box capacity ≈ peak + headroom | Tie to [Estimation fluency](./estimation-fluency.md) |

**Little’s Law** (Staff should know): `concurrency ≈ arrival_rate × latency`. At 5k rps and 200ms, ~1k in-flight requests — sizes thread pools and connection limits.

---

## Latency & overload

| Term | Meaning | Staff usage |
|------|---------|-------------|
| **p50 / p99 / tail latency** | Percentile latency — see [core-concepts](./core-concepts.md#latency-metrics-vocabulary) | Quote SLOs on p99, not average |
| **Timeout budget** | Max time each downstream hop may use | Sum of child timeouts must fit inside parent SLA |
| **Backpressure** | Slow consumer signals upstream to stop or slow producing | Bounded queues, 429, shed load — not unbounded memory |
| **Load shedding** | Drop or reject low-priority work under overload | Protect critical path (OTP > promo) |
| **Rate limiting** | Cap requests at edge (per IP, tenant, API key) | Protects **your** service and **downstream** |
| **Throttling** | Slow down accepted work (token bucket) | Smoother than hard reject; used at vendor APIs |
| **Circuit breaker** | Stop calling a failing downstream for a cooldown | Prevents cascading failure and retry amplification |
| **Graceful degradation** | Reduced feature set under stress | “Search stale OK; payments must stay strong” |
| **Cold vs warm path** | First request (cache miss, JIT, new pod) vs steady | Cold paths set tail latency — mention warmup and caches |
| **SLI / SLO / error budget** | What you measure, target, allowed bad window | See [Reliability & SLOs](./reliability-and-slos.md) |

**Overload sentence:** “When error budget burns, we shed promotional sends first and keep transactional p99 under 5s.”

---

## Isolation & failure spread

| Term | Meaning | Staff usage |
|------|---------|-------------|
| **Blast radius** | How much breaks when one component fails | “Shard by merchant so one bad deploy doesn’t take all payments” |
| **Bulkhead** | Separate pools (threads, connections, queues) per tenant or feature | Noisy neighbor isolation |
| **Cell / shard isolation** | Failure or load contained to one partition | Multi-tenant platforms (Razorpay-class) |
| **Cascading failure** | Upstream retry storm or timeout pile-up takes down healthy nodes | Break with timeouts, jittered backoff, breakers, bulkheads |
| **Fail open vs fail closed** | Degrade permissively vs reject | Config cache: fail open (stale OK). Payments: fail closed |
| **Poison message** | One bad message blocks or crashes consumers | DLQ, skip with metric, don’t infinite-retry |
| **RPO / RTO** | Data loss window / time to restore service | Multi-region justification — [Reliability](./reliability-and-slos.md) |

---

## Queues & async paths

| Term | Meaning | Staff usage |
|------|---------|-------------|
| **At-least-once / at-most-once / exactly-once** | Delivery semantics | Interview default: at-least-once + idempotent consumers |
| **Consumer lag** | Backlog depth on stream/queue | Scale workers on lag, not CPU alone |
| **DLQ** | Parking lot for poison / exhausted retries | Must have replay runbook |
| **Visibility timeout** | SQS-style in-flight hide period | Retry delay mechanism |
| **Head-of-line blocking** | One slow message blocks the rest of ordered stream | Split lanes (transactional vs bulk), separate partitions |
| **Dual write** | Two stores updated without atomicity | Outbox or CDC — [Messaging](./messaging-and-pipelines.md) |

---

## How to use this in a round

1. **Label the diagram once:** “Upstream: Client. Downstream: Kafka, DB, PSP.”
2. **Name the bottleneck resource** after estimates: “DB connections, not CPU.”
3. **Walk one failure:** “If downstream PSP is slow, breaker opens; we queue and alert; user sees pending, not double charge.”
4. **State overload policy:** what you shed, what you never shed.
5. Tie to **SLO + error budget** — vocabulary without numbers is weak Staff signal.

Cross-check: [README — SDE3 / Staff expected extras](../README.md#what-does-the-interviewer-evaluate) · [Soft skills — communication bar](./soft-skills.md#sde3-staff-communication-bar)
