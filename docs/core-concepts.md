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

After repeated failures to a dependency, **stop calling** for a cool-down; fail fast or use fallback; half-open probe to recover.

Stops cascading failures across microservices.

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
