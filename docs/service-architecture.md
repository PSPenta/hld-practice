# Service Architecture

How you split the app: monolith vs microservices, request-path **services** vs async **workers**, and DB connection habits.

← [README](../README.md) · [Docs index](./README.md) · Related: [Building blocks](./building-blocks.md) · [Messaging](./messaging-and-pipelines.md) · [Data stores](./data-stores.md)

---

## Index

- [Monolith vs microservices](#monolith-vs-microservices)
- [Services vs workers](#services-vs-workers)
- [Where to use services vs workers](#where-to-use-services-vs-workers)
- [API contracts other teams depend on](#api-contracts-other-teams-depend-on)
  - [Evolving without breaking them](#evolving-without-breaking-them)
- [Beyond services & workers](#beyond-services-workers)
- [Cron / scheduler vs Temporal](#cron-scheduler-vs-temporal)
- [DB topology & connections](#db-topology-connections)
- [Untrusted / CPU-heavy work: Node vs Go](#untrusted--cpu-heavy-work-node-vs-go)
- [Blocked Node event loop — detect / fix](#blocked-node-event-loop--detect--fix)

---

## Monolith vs microservices

| | **Monolith** | **Microservices** |
|--|--------------|-------------------|
| Ship | One deployable unit | Many services, own deploys |
| Data | Usually one (or few) DBs | DB per service (ideal); avoid shared DB soup |
| Complexity | Code coupling inside one repo | Network, versioning, observability, sagas |
| Scale | Scale the whole app | Scale hot services alone |
| Best when | Small team, unclear boundaries, early product | Clear domains, independent scale/release, large orgs |

**Interview default:** start **modular monolith** (clear modules, one deploy) → extract a service when you have a real reason (scale, team ownership, failure isolation) — not because “microservices” sounds Staff-level.

**Split when:** different SLOs, different scale, separate release cadence, or blast-radius needs.  
**Don’t split when:** you can’t define an API/data boundary without distributed transactions on every request.

---

## Services vs workers

| | **Service (API / request path)** | **Worker (async consumer)** |
|--|----------------------------------|-----------------------------|
| Trigger | Sync HTTP/gRPC/WebSocket from client or peer | Queue/stream message, cron, CDC event |
| Latency | User waits — target p99 | Can take seconds–minutes |
| Failure | Return error / retry client-side | Retry + DLQ; don’t block the user |
| Scale | On RPS / concurrency | On **queue lag** / message rate |
| Examples | Login, get feed, create order API | Send email, resize image, rebuild index, reconcile payment |

```text
Client → API Service → DB
              ↓ enqueue
           Worker(s) → side effects / DB / 3rd party
```

---

## Where to use services vs workers

| Put in a **service** | Put in a **worker** |
|----------------------|---------------------|
| Needs immediate response | User doesn’t need result in that request |
| Auth, reads, small writes | Email/SMS, webhooks, fan-out, ETL, search index |
| Strong consistency UX (book seat) | Heavy CPU (encode, ML), bursty load |
| Orchestrate “accept” then async | Poison-prone / flaky 3rd parties |

**Pattern:** service does **fast, correct accept** (validate + persist + enqueue) → worker does **slow / unreliable** work. Payments: authorize/idempotent write in service path; notify merchant / reconcile in workers.

---

## API contracts other teams depend on

**Interview snapshot**
- **What:** Shared APIs start with resource model, errors, auth, idempotency, SLOs — then evolve safely.
- **Why:** Other teams build on you; breaks are expensive.
- **Trade-off:** `/v2` for breaks keeps `/v1` alive (cost) vs forcing all clients to upgrade.
- **Example:** Add optional field in v1; rename required field → new version.

**Decide first (platform / shared API)**

1. **Resource model + operations** (what exists; not every field yet)  
2. **Request/response shape** — required vs optional; types; error model  
3. **Authn/z**, idempotency for creates/money, **pagination** (offset vs cursor — see [Cursor vs offset](./algorithms-and-indexes.md#cursor-vs-offset-paginated-queries)), rate limits  
4. **SLOs** consumers will depend on (latency / availability)

Fields matter, but **compatibility and failure behavior** matter more for other teams.

### Evolving without breaking them

| Change | How |
|--------|-----|
| **Additive** (new optional field, new endpoint) | Ship in same version; old clients ignore unknown fields |
| **Semantic break** (rename/remove/required change, meaning change) | **New version** (`/v2` or version header); keep `/v1` until sunset date |
| **Soft deprecate** | Document; dual-write/read if needed; monitor old-version traffic |

**Interview line:** “Contract + compatibility rules first. Breaking changes go to a new version; additive changes don’t force upgrades.”

---

## Beyond services & workers

Service vs worker is the main split — other shapes still show up:

| Shape | Trigger | Use when |
|-------|---------|----------|
| **Cron / scheduler** (Airflow, Cloud Scheduler) | Time | Periodic reconcile, reports, cleanup, downsampling |
| **Serverless function** (Lambda) | HTTP, queue, schedule | Spiky / rare work; pay-per-invoke; watch cold start + DB connections |
| **Stream processor** (Flink, Spark Streaming) | Continuous log | Aggregations, fraud, CDC fan-out — stateful pipelines |
| **Batch job** | Nightly / on-demand | Large backfills, warehouse ETL (Lambda architecture batch layer) |
| **BFF / edge** | Client request | Aggregate APIs for one UI; edge auth/cache — still a “service,” thinner |
| **Workflow engine** (Temporal, Step Functions) | Long-running saga | Multi-step business processes with retries/state |

Most of these are still either **request-path** (like a service) or **async** (like a worker). Cron/batch/stream = async family; BFF/edge = service family.

---

## Cron / scheduler vs Temporal

| | **Cron / scheduler** | **Temporal** (workflow engine) |
|--|----------------------|--------------------------------|
| Model | “Run this job at time T / every N min” | “Run this **long-running process** with steps + state” |
| State | Usually none (job is stateless each run) | Durable workflow history; resumes after crash |
| Retries | Per-run success/fail; you code the rest | Built-in retries, timeouts, heartbeats per activity |
| Fit | Cleanup, reports, nightly ETL, “reconcile last hour” | Onboarding, payouts, KYC, multi-step booking, sagas |
| Fan-out | One schedule → many rows inside the job | One workflow **per entity** (e.g. per payment) |
| Overkill when | Simple periodic task | — |
| Weak when | Multi-day saga, partial progress, human waits | Simple “run vacuum at 3am” |

**Rule:** time-triggered + idempotent batch → **cron**. Multi-step business process that must survive failures mid-flight → **Temporal** (or Step Functions). Often combined: cron *starts* workflows, or cron handles coarse batch while Temporal handles per-entity flows.

---

## DB topology & connections

### Do service and workers share one DB?

**Usually yes (early / same bounded context):** one primary DB; API services + workers all talk to it (via pooler). Microservices ideal is still **DB per service**, not “DB per worker type.”

| Setup | When |
|-------|------|
| **Same primary** | Same domain data; workers finish work the API started (outbox, status updates) |
| **Workers → read replica** | Heavy read/report/index build; keep load off primary user path |
| **Separate DB / store** | Different domain (e.g. analytics), or worker write load would starve OLTP — then CDC/events sync |
| **Not** “1 service = 1 DB server, all workers → one other DB” as a rule | Workers aren’t a second product by default; split by **domain & load**, not by process type alone |

**Your intuition (protect user-facing DB):** right *goal*. Prefer: pooler + short txs + **replica for heavy worker reads** + move non-OLTP to another store — full separate primary only when worker write load or blast radius demands it.

```text
                    ┌─ API services ──┐
Clients ──► API ───┤                  ├──► Pooler ──► Primary DB
                    └─ Workers (writes)┘         └──► Read replica(s) ◄── Workers (heavy reads)
```

### Connection pools

Budget ≈ `(api_replicas × pool) + (worker_concurrency × pool) + admin`.

| | **API services** | **Workers** |
|--|------------------|-------------|
| Sizing | Concurrent requests | Cap concurrency / prefetch first |
| Risk | Too many pods × pool | Scaling consumers exhaust DB faster |
| Holding | Request-scoped only | Don’t hold conn during 3rd-party I/O |

**Rule:** global connection budget + **PgBouncer/RDS Proxy**; Lambda/stream shapes same idea — bounded parallelism, not one conn per event.

---

## Untrusted / CPU-heavy work: Node vs Go

**Interview snapshot**
- **What:** Where to run untrusted user code or CPU-bound work when the API is Node vs Go.
- **Why:** Node’s single-threaded event loop stalls on CPU; untrusted code can crash or steal the process.
- **Trade-off:** Offloading to workers/sidecars adds latency and ops; keeping it in-process is simpler until it isn’t.
- **Example:** Image resize / PDF render / sandbox eval → don’t do it on the Node request thread; queue to a Go/worker pool or isolate.

| Work | Prefer | Why |
|------|--------|-----|
| **CPU-heavy** (encode, zip, crypto on big payloads) | Go / Rust worker, or Node **worker_threads** / separate process pool | Node event loop can’t serve others while busy |
| **Untrusted** (user plugins, eval, scrape) | Sandbox (container/VM/gVisor), separate service, timeouts, no shared secrets | Blast radius; Node `vm` is not a security boundary |
| **I/O-bound API** | Node or Go both fine | Async I/O shines; language less critical |
| **Tight p99 + heavy CPU on path** | Go (or native) on that path | Fewer GC pauses / better multi-core defaults |

**Interview line:** “Node for I/O APIs; push CPU and untrusted work to isolated workers — never block the event loop or share the API process with hostile code.”

---

## Blocked Node event loop — detect / fix

**Interview snapshot**
- **What:** Main thread stuck in sync CPU or sync I/O → all requests freeze (latency ↑, health checks fail).
- **Why:** One bad handler starves the whole process.
- **Trade-off:** Worker threads / separate processes cost complexity; sync APIs are easier to write wrong.
- **Example:** `JSON.parse` on a 50MB body or sync `fs.readFileSync` in a hot route → event-loop lag spikes.

**Detect**

- Metric: **event loop delay / lag** (`perf_hooks.monitorEventLoopDelay`, clinic/doctor, APM “event loop” charts)
- Symptom: p99 ↑ across **all** routes on that pod; CPU high or “stuck”; readiness may still pass until timeout

**Fix**

1. Find sync hot spots (CPU profiles, `async_hooks` / APM spans).  
2. Move CPU to `worker_threads`, child process, or **async worker service**.  
3. Replace sync FS/crypto with async; stream large payloads.  
4. Bound body size; reject huge inputs early.  
5. Fail readiness if loop lag exceeds budget (drain traffic).

**Interview line:** “Watch event-loop lag; treat a blocked loop like a saturated thread pool — offload CPU, never sync I/O on the hot path.”
