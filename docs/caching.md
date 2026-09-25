# Caching (Deep Dive)

Caching is easy to draw and easy to get wrong. Interviewers probe **failures and invalidation**, not only Redis.

← [README](../README.md) · [Docs index](./README.md) · Diagram: [Distributed Caching System](../diagrams/distributed-caching-system/distributed-caching-system.excalidraw)

---

## Index

- [Quick recap of patterns](#quick-recap-of-patterns)
- [Cache stampede (a.k.a. dogpile / thundering herd)](#cache-stampede-aka-dogpile-thundering-herd)
- [Cache avalanche](#cache-avalanche)
- [Cache penetration](#cache-penetration)
- [Cache invalidation](#cache-invalidation)
- [Eviction policies](#eviction-policies)
- [Other cache topics worth knowing](#other-cache-topics-worth-knowing)

---

## Quick recap of patterns

| Pattern | Idea | When |
|---------|------|------|
| **Cache-aside** | App reads cache → miss → DB → fill cache; app invalidates/updates on write | **Default.** Reads dominate; brief stale OK; you control fill/invalidation |
| **Read-through** | Cache library loads DB on miss | Same as aside, but fill logic lives in cache layer |
| **Write-through** | Every write updates **cache + DB** together | Stronger freshness on read-after-write; **higher write latency**; still need failure handling if one succeeds and one fails |
| **Write-behind** | Write cache first; async flush to DB | Rare — speed over durability; risk data loss on crash |

**Do not pick write-through because “many users read the same key.”** Shared hot reads → cache-aside (or read-through) + stampede protection. Write-through is about **write-path consistency**, not read fan-out.

**Interview line:** “Cache-aside for most APIs. Write-through only if product needs read-your-writes from cache and we accept slower writes.”

---

## Cache stampede (a.k.a. dogpile / thundering herd)

**What:** Hot key expires → many concurrent requests miss → all hit DB → overload.

**Mitigations**

- **Singleflight / request coalescing** — one filler, others wait (best in-process)
- **Probabilistic early expiration** — refresh before TTL hits zero
- **Short lock on miss** — Redis `SET key NX EX` (or Redlock only if multi-region lock truly needed — usually overkill)
- **Soft TTL** — serve stale while one request refreshes
- Never set identical TTL on millions of keys created together (jitter TTLs)

**Not a stampede fix:** “switch to write-through.”

---

## Cache avalanche

**What:** Many keys expire **at the same time** (or cache cluster dies) → sudden flood to DB.

**Mitigations**

- Jitter TTLs (`TTL + random`)
- High availability cache (replica / cluster); multi-AZ
- Soft dependency: degrade with stale data or shed load
- Circuit breaker + bulkheads so DB isn’t destroyed
- Warm critical keys after failover

Stampede = one hot key; avalanche = **many keys / whole tier** failing together.

---

## Cache penetration

**What:** Requests for **data that doesn’t exist** (or malicious IDs) → always miss → always hit DB.

**Mitigations**

- Cache **negative entries** (“not found”) with short TTL
- **Bloom filter** in front — reject definitely-absent keys cheaply (see [Algorithms & indexes](./algorithms-and-indexes.md))
- Auth / rate limit / captcha for abusive patterns
- Validate IDs before DB (format, existence service)

---

## Cache invalidation

Hardest part of caching. Strategies:

| Strategy | When |
|----------|------|
| **TTL only** | Mild staleness OK (public profiles, counters) |
| **Delete on write** | Update/delete entity → `DEL` cache key |
| **Update on write** | Write new value to cache (write-through) |
| **Versioned keys** | `user:42:v7` — bump version on change |
| **Pub/Sub invalidate** | One writer notifies all cache nodes |

Be explicit about **staleness budget** (“feed can be 30s stale”).

Related failure: **cache inconsistency** across nodes after partial invalidation — prefer delete + lazy refill over dual updates when unsure.

---

## Eviction policies

When memory is full (or TTL fires):

| Policy | Evicts | Good for |
|--------|--------|----------|
| **TTL** | Expired keys | Session, OTP, short-lived locks |
| **LRU** | Least recently used | General hot-key caches |
| **LFU** | Least frequently used | Stable popularity skew |
| **FIFO / random** | Simple / cheap | Rarely ideal alone |
| **Volatile-LRU** | LRU among keys **with** TTL (Redis) | Protect persistent keys |

Interview Redis knobs: `maxmemory-policy` (`allkeys-lru`, `volatile-lru`, `noeviction`, …).

Often combine: **TTL for freshness** + **LRU for memory pressure**. See eviction notes in the [caching diagram](../diagrams/distributed-caching-system/distributed-caching-system.excalidraw).

---

## Other cache topics worth knowing

- **Hot key** — one key huge QPS → local cache, split key, read replica of cache
- **Large value** — don’t stuff multi-MB blobs in Redis; use object store + cache URL
- **Serialization** — CPU cost of JSON vs binary; pipeline/batch GETs
- **Aside vs gateway cache** (CDN / HTTP cache-control) — different layer, same invalidation pain
- **Read replica lag** vs cache — don’t treat replica as a coherent cache without care
