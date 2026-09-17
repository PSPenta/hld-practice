# Core Concepts

Ideas interviewers expect you to **apply** when comparing designs. Prefer concrete trade-offs over buzzwords.

← [README](../README.md) · [Docs index](./README.md)

---

## Index

- [CAP and PACELC (practical view)](#cap-and-pacelc-practical-view)
- [Consistency models](#consistency-models)
- [ACID vs BASE](#acid-vs-base)
- [CRDT (Conflict-free Replicated Data Type)](#crdt-conflict-free-replicated-data-type)
- [Idempotency](#idempotency)
- [Optimistic locking & versioning](#optimistic-locking-versioning)
- [Latency vs throughput](#latency-vs-throughput)
- [Latency & metrics vocabulary](#latency-metrics-vocabulary)
- [Availability & failure modes](#availability-failure-modes)
- [Partitioning & hot keys](#partitioning-hot-keys)
- [Rate limiting](#rate-limiting)
- [Backpressure](#backpressure)
- [Exponential backoff](#exponential-backoff)
- [Circuit breaker](#circuit-breaker)
- [Security basics (HLD depth)](#security-basics-hld-depth)
- [Fan-out on write vs read](#fan-out-on-write-vs-read)
- [Sync vs async](#sync-vs-async)
- [How to talk about trade-offs](#how-to-talk-about-trade-offs)

---

## CAP and PACELC (practical view)

**CAP:** in a partition, you choose between **consistency** and **availability** (you always want partition tolerance in distributed systems).

**PACELC:** even without a partition — if you replicate, you trade **latency** vs **consistency**.

Interview translation:

- Banking ledger / seat booking → lean **consistent** (accept refusals or higher latency)
- Social feed / likes count → often **available + eventual**
- Always say *which* data needs which model — systems are rarely one setting globally

## Consistency models

| Model | Meaning | Example |
|-------|---------|---------|
| **Strong** | Readers see latest committed write | Inventory decrement, payments |
| **Read-your-writes** | You see your own updates | Profile edit |
| **Eventual** | Replicas converge; temporary staleness OK | Follower counts, search index |
| **Causal** | Respect cause→effect order | Comment threads (sometimes) |

Don’t claim “eventual consistency” without saying **how** you reconcile (last-write-wins, CRDTs, version vectors, business merge).

## ACID vs BASE

| | **ACID** | **BASE** |
|--|----------|----------|
| Focus | Strong transactional guarantees | Availability + eventual consistency |
| Fit | Ledger, booking, inventory | Feeds, caches, presence, analytics ingest |

Full comparison and when to mix both: [Data stores — ACID vs BASE](./data-stores.md#acid-vs-base).

## CRDT (Conflict-free Replicated Data Type)

Data structures that **merge concurrent updates** without coordination (or with minimal coordination) so replicas converge.

| Example CRDT | Use |
|--------------|-----|
| G-Counter / PN-Counter | Distributed counters (likes, with limits) |
| G-Set / OR-Set | Grow-only or add/remove sets |
| LWW-Register | Last-write-wins fields (needs timestamps/causality care) |
| RGA / text CRDTs | Collaborative editors (Google Docs–style) |

**Pros:** multi-region writes, offline-first apps.  
**Cons:** not every business invariant is CRDT-friendly (money transfer ≠ naïve counter); metadata overhead; “merge” may not match user intent without extra rules.

**Vs operational transforms (OT):** both enable collab editing; CRDT often easier to reason about multi-replica; OT historically used in some doc systems.

When you say “eventual consistency,” CRDTs are one *concrete* merge story — better than hand-waving.

## Idempotency

Same request applied twice → same effect as once.

Critical for:

- At-least-once queues
- Client retries / flaky networks
- Payment and booking APIs

Patterns:

- **Idempotency key** (client-generated UUID) stored with result
- Natural keys (`orderId + seatId`)
- Conditional writes (`UPDATE … WHERE status = 'pending'`)

## Optimistic locking & versioning

Avoid long DB locks under contention:

- Store `version` / `etag` on the row
- Update only if version matches; on conflict → retry or fail to client

Useful for profiles, carts, concurrent editors (with more advanced CRDT/OT for Google Docs–style).

## Latency vs throughput

| | **Latency** | **Throughput** |
|--|-------------|----------------|
| Meaning | Time for **one** request | Work per second (QPS / bytes/s) |
| Quote | **p50 / p95 / p99** (not only avg) | Sustained and peak QPS |
| Trade-off | Batching ↑ throughput, often ↑ latency | More parallelism can ↑ both until a bottleneck |

**Also talk about (latency family)**

- **Tail latency (p99/p999)** — what users feel under load; outliers matter  
- **Cold vs warm** — cold cache / cold start / cold connection ≫ warm path  
- **Queueing delay** — when utilization is high, wait time dominates (shed load / scale)  
- **Fan-out** — parallel calls ≈ slowest dependency; serial calls add up  
- **Bandwidth ≠ latency** — fat pipe can still be high RTT  
- **Consistency vs latency** — sync replication / cross-region adds RTT ([PACELC](#cap-and-pacelc-practical-view))

State targets early: e.g. search p99 &lt; 200ms; autocomplete p99 &lt; 50ms.

## Latency & metrics vocabulary

The notation everyone uses without defining it.

| Term | Meaning |
|------|---------|
| **p50 (median)** | Half of requests are faster than this |
| **p95 / p99** | 95% / 99% of requests finish under this value |
| **p99.9 / p99.99** | The far tail; needs high traffic to measure meaningfully |
| **Tail latency** | The slow end (p99+) — what users complain about |
| **QPS / RPS / TPS** | Queries / requests / transactions per second |
| **Goodput** | Useful throughput, excluding retries and errors |

**Why not the average**

99 requests at 10ms + 1 request at 10s → mean ≈ 110ms. Looks healthy; one user waited 10 seconds. Quote percentiles, never only the mean.

**Tail at scale (the important one)**

If a request fans out to 100 dependencies, each with p99 = 100ms, then P(all fast) = 0.99¹⁰⁰ ≈ **37%**. So ~63% of requests hit at least one slow call — your p99 becomes the median user's experience.

Mitigations: hedged requests, aggressive timeouts, partial responses, fewer serial hops.

**Percentiles don't add or average**

- Can't average p99 across servers to get fleet p99
- Can't add p99 of two services to get end-to-end p99
- Need **histograms** (why Prometheus stores buckets + `histogram_quantile`)

**Little's Law** — `concurrency = arrival rate × latency`

5,000 rps × 200ms = **1,000 in-flight requests**. Use it to size thread pools, connection pools, and worker counts on the whiteboard.

**Utilization vs latency**

Queueing delay grows ~`ρ/(1−ρ)`, so latency explodes non-linearly near saturation — 70% → 90% busy roughly triples wait time. Keep steady-state utilization ~70–80%. "Spare CPU" ≠ "spare latency budget".

**Availability nines**

| Target | Downtime / year |
|--------|-----------------|
| 99.9% | ~8.8 hours |
| 99.95% | ~4.4 hours |
| 99.99% | ~53 minutes |

**Incident timing** — **MTTD** (detect), **MTTR** (recover), **MTBF** (between failures); plus **RPO / RTO** for DR ([Reliability](./reliability-and-slos.md)).

## Availability & failure modes

- **Replicas** — survive node loss; define failover
- **Multi-AZ** — survive data-center zone loss
- **Health checks** — remove bad instances from LB
- **Graceful degradation** — feed without recommendations, checkout without promo engine
- **Timeouts, retries with jitter, circuit breakers** — don’t amplify outages

Ask: “What does the user see when X is down?”

## Partitioning & hot keys

Split data/load by a **partition key** (userId, tenantId, orderId).

Bad keys:

- Low cardinality (`country`, `status`) → uneven shards
- Celebrity userId → one hot partition

Mitigations: salt/hash buckets, separate “hot” path, local caches, fan-out on read for celebs (see social feed designs).

## Rate limiting

Protects your system (and downstream) from abuse and stampedes.

Algorithms (know one well):

- **Token bucket** — burst + sustained rate
- **Leaky bucket** — smooth outflow
- **Fixed / sliding window** — simple counters (Redis)

Place at gateway and/or per-tenant / per-IP / per-API key. Return `429` with clear retry guidance.

## Backpressure

When consumers are slow, don’t let unbounded queues melt memory.

- Limit queue depth; shed load; slow producers
- Autoscale workers from lag metrics
- Prefer fail-fast over silent infinite buffering

## Exponential backoff

Retry transient failures with growing delay: `min(cap, base × 2^n) + jitter`.

- Stops synchronized retry storms; always **cap attempts**
- Only on **idempotent** operations (or with idempotency keys)
- Full write-up: [Reliability — Exponential backoff](./reliability-and-slos.md#exponential-backoff)

## Circuit breaker

### Problem it solves

Retries + timeouts alone can **amplify** an outage: every app pod keeps hammering a dead Redis/DB/PSP → thread pools fill, latency spreads to healthy paths, cascade.

A **circuit breaker** watches calls to one dependency. After enough failures it **stops calling** for a cool-down (**fail fast** or **fallback**), then probes lightly to see if recovery started.

```text
Closed ──(failures exceed threshold)──► Open ──(cool-down elapsed)──► Half-open
  ▲                                      │                              │
  │                                      │ success probe                │ fail probe
  └──────────────(success)───────────────┴──────────────────────────────┘
```

| State | Behavior |
|-------|----------|
| **Closed** | Normal: calls go through; failures counted in a window |
| **Open** | Short-circuit: **do not** call dependency; return error or fallback immediately |
| **Half-open** | Allow a **small** number of probe calls; success → Closed, failure → Open again |

Without half-open you’d either stay open forever or flap. Without open you’d keep retrying into a black hole.

### Trip criteria (strategies)

| Strategy | Trip when… | Notes |
|----------|------------|-------|
| **Consecutive failures** | N failures in a row | Simple; noisy on rare blips |
| **Failure rate in window** | e.g. ≥50% errors in last 20s / 100 calls | Better for high QPS |
| **Slow-call rate** | e.g. ≥80% calls slower than 1s | Dependency “up” but useless |
| **Hybrid** | Error rate **or** slow-call rate | Common in Resilience4j-style configs |

Also configure: **minimum calls** before evaluating rate (don’t open on 1/1 failure), **open duration** (cool-down), **half-open max probes**.

### Scope: what one breaker protects

Prefer **one breaker per dependency (and often per operation)**, not one global switch for the whole process:

- `redis-cache` vs `postgres-primary` vs `payment-psp` — separate  
- Optional: per-shard / per-downstream-host so one bad host doesn’t open the whole pool (**bulkhead** + breaker)

Global “everything open” hides which dependency died and blocks healthy paths.

---

### Use cases: Redis, DB, both

Assume a typical read path: **try Redis → on miss hit DB**.

#### 1) Redis failure only

| Without breaker | With breaker |
|-----------------|--------------|
| Every request times out on Redis (~100–200ms×retries) then hits DB | After trip: **skip Redis**, go DB (or serve stale if you have it) |
| DB suddenly gets 100% traffic **plus** timeout delay on every call | DB gets more load, but **no** Redis timeout tax; p99 stays bounded |

**Fallback when open:** treat as cache miss (read-through DB), or serve **stale** local/previous value if soft dependency. **Do not** infinite-retry Redis in the request path.

**Staff note:** Redis is usually a **soft** dependency for reads — breaker open → degrade, don’t 500 the user if DB is healthy.

#### 2) DB failure only

| Path | Behavior |
|------|----------|
| Cache **hit** | Can still serve from Redis while DB breaker is open (stale-OK reads) |
| Cache **miss** / writes | Fail fast or queue; don’t pile connections on a dead primary |

**Fallback when open:** read-only stale from cache; for writes → `503` + retry-later, or outbox/queue if the product allows async accept. Payments/bookings: **fail closed** (don’t pretend success).

#### 3) Both Redis and DB failing

Breakers on **both** open → request path must **fail fast** (503) or serve a hard-coded/minimal degraded page. No point retrying either.

Alert: this is an incident, not a “retry harder” situation. Shed load at gateway.

#### 4) Other common HLD placements

| Dependency | When open |
|------------|-----------|
| **PSP / SMS vendor** | Queue intent; show “pending”; don’t block all checkouts on sync retries |
| **Search / recommendations** | Omit widget; core browse/checkout continues |
| **Downstream microservice** | Fallback cached response or default; isolate with bulkhead thread pool |

---

### How to implement

**Where it lives:** in the **caller** (app / client SDK), not inside Redis/DB. Libraries wrap the outbound call.

**Typical loop (conceptual):**

1. If state **Open** and cool-down not finished → return fallback / error (no I/O).  
2. If **Half-open** → allow limited probes only.  
3. Execute call with a **strict timeout**.  
4. Record success/failure/slow into the breaker metrics.  
5. Transition state per thresholds above.

**In process (common in interviews):**

| Stack | Examples |
|-------|----------|
| Java | Resilience4j, Sentinel, (legacy Hystrix ideas) |
| Go | `sony/gobreaker`, custom middleware |
| Node | `opossum`, cockatiel |
| Sidecar | Envoy outlier detection (eject bad hosts) — related idea at mesh layer |

**Distributed breaker (optional Staff depth):** share open/closed state in Redis so 100 pods don’t each need N failures to learn. Trade-off: Redis is also a dependency — often **local breaker per pod** is enough; pods trip within one window anyway under load.

**Pair with (never alone):**

| Control | Role |
|---------|------|
| **Timeouts** | Bound each attempt |
| **Retries + jitter** | Only while breaker **Closed**; **stop** when **Open** |
| **Bulkhead** | Separate thread/connection pools so Redis timeouts don’t starve DB calls |
| **Load shedding** | 503 at edge if error budget burning |

See [Exponential backoff](./reliability-and-slos.md#exponential-backoff) and [Caching — avalanche](./caching.md#cache-avalanche).

### Fallbacks (pick per dependency)

| Dependency type | Sensible fallback |
|-----------------|-------------------|
| Cache (Redis) | Skip cache → DB; or stale |
| Read replica | Fail over to primary (watch load) or stale cache |
| Primary DB writes | Fail closed / accept async with clear UX |
| Recommendations | Empty list |
| Idempotent side-effect (SMS) | Queue + retry later |

### Interview pitfalls

- Opening one breaker for “all I/O” — can’t degrade partially  
- Retrying **while open** — defeats the breaker  
- No timeout — breaker never sees “failure,” only hung threads  
- Treating DB like Redis (soft) on payment/ledger paths — wrong; fail closed  
- No metrics/alert on open transitions — silent degradation  

**Interview line:** “Per dependency circuit breaker: closed → open on error/slow rate → half-open probes. Redis open: skip cache. DB open: stale reads or fail writes closed. Never retry into an open circuit; bulkhead pools so one dependency can’t exhaust the process.”

## Security basics (HLD depth)

Enough to place boxes and call out risks:

| Topic | Interview expectation |
|-------|------------------------|
| **Authn** | Who are you? (session, JWT, OAuth) |
| **Authz** | What can you do? (RBAC, resource checks) |
| **Transport** | TLS everywhere external; often internal too |
| **Secrets** | Not in code; vault / KMS / env from secret store |
| **Least privilege** | Services get only needed IAM / DB grants |
| **PII / PCI** | Minimize storage; tokenize cards; audit access |

Payment designs should mention **idempotency + PCI scope reduction** (never store raw PANs if avoidable). See [Payment Gateway](../diagrams/payment-gateway-system/payment-gateway-system.excalidraw).

## Fan-out on write vs read

| | **Fan-out on write** | **Fan-out on read** |
|--|----------------------|---------------------|
| Idea | Precompute timelines on post | Compute feed when user opens app |
| Pros | Fast reads | Simple writes; OK for celebs |
| Cons | Expensive for huge follower counts | Heavier reads; cache carefully |
| Hybrid | Normal users write-fanout; celebs read-fanout | Common in Twitter-like systems |

See [Social Media App](../diagrams/social-media-app/social-media-app.excalidraw).

## Sync vs async

- **Sync** — user waits; needs low latency and clear errors (book seat, pay)
- **Async** — accept job, process later (email, image resize, search index)

Use async when the user doesn’t need the result in the same request — and expose status/webhooks when they do.

---

## How to talk about trade-offs

Template that scores well:

> “Option A gives us X (e.g. strong consistency) at the cost of Y (higher latency / lower availability). Given NFR Z, I’ll pick A for the booking path and B for the feed.”

Always tie the choice to a **stated requirement**.

---

## See also

- [Scaling](./scaling.md) — RPS vs CPU vs memory; partitioning vs sharding
- [Distributed coordination](./distributed-coordination.md) — races, 2PC/3PC, Saga, Raft
- [Data stores](./data-stores.md) — ACID/BASE, SQL/NoSQL, CRDT companions, Vector DB
- [Caching deep dive](./caching.md) — invalidation and failure modes
- [Algorithms & indexes](./algorithms-and-indexes.md) — hashing vs encryption, Bloom filters
