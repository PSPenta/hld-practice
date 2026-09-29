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
- [Eviction vs invalidation](#eviction-vs-invalidation)
- [What must never be cached](#what-must-never-be-cached)
- [Hit rate crashed overnight (first 10 minutes)](#hit-rate-crashed-overnight-first-10-minutes)
- [Other cache topics worth knowing](#other-cache-topics-worth-knowing)

---

## Quick recap of patterns

**Interview snapshot**
- **What:** Aside = app fills on miss; write-through = every write updates cache + DB together.
- **Why:** Wrong pattern either wastes write latency or serves stale reads.
- **Trade-off:** Aside is simpler/faster writes but brief stale; write-through fresher reads, slower writes.
- **Example:** Product catalog reads → cache-aside; rarely “many readers ⇒ write-through.”

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

**Interview snapshot**
- **What:** Hot key expires → many concurrent misses → all hit DB.
- **Why:** One popular key can melt the database.
- **Trade-off:** Singleflight/lock adds complexity; soft TTL may serve briefly stale data.
- **Example:** Homepage config key TTL hits zero during peak → thundering herd.

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

## Eviction vs invalidation

**Interview snapshot**
- **What:** Invalidation = you drop a key because data changed; eviction = cache frees memory/TTL.
- **Why:** Mixing them up misdiagnoses incidents (bug vs capacity).
- **Trade-off:** Aggressive invalidation = fresher data, more DB load; lazy TTL = cheaper, more stale.
- **Example:** Profile update → `DEL user:42` (invalidation); Redis LRU drops cold keys under `maxmemory` (eviction).

| | **Invalidation** | **Eviction** |
|--|------------------|--------------|
| **Trigger** | **Your write** (or explicit purge) — data changed / must not be served | **Memory / TTL pressure** — redis needs space or key expired |
| **Intent** | Correctness / freshness | Capacity |
| **Example** | `DEL user:42` after profile update | LRU drops cold keys; TTL expires session |

**Interview line:** “Invalidation is product logic (we chose to drop a key). Eviction is the cache protecting itself under memory/TTL.”

---

## What must never be cached

**Interview snapshot**
- **What:** Data that is wrong or dangerous if served stale — even for milliseconds.
- **Why:** Cache hits can skip authz checks, double-spend money, or sell held inventory.
- **Trade-off:** Always hitting DB/source of truth costs latency; worth it on money/security paths.
- **Example:** Don’t cache “seat hold for user X” across pods with a long TTL — hold must live in the booking store with expiry.

| Don’t cache (or only with extreme care) | Why |
|-----------------------------------------|-----|
| **Authn / Authz decisions** (session validity, permissions, API keys) | Stale “allowed” after revoke → security hole |
| **Money / ledger balances**, payment auth outcomes | Stale balance → overdraft / double charge |
| **Inventory holds / seats / coupons** while reserved | Stale “available” → oversell |
| **One-time tokens** (OTP, magic link, nonce) | Replay after consume |
| **PII that policy forbids** in Redis/CDN | Compliance / breach blast radius |

**OK to cache with short TTL + invalidation:** public catalog, profile display fields, feature flags (with careful revoke path).

**Interview line:** “If staleness can lose money or grant access, it isn’t a cache problem — it’s source-of-truth + short-lived locks/holds.”

---

## Hit rate crashed overnight (first 10 minutes)

**Interview snapshot**
- **What:** Sudden drop in cache effectiveness overnight.
- **Why:** Miss storm hits the DB; can take down the primary path.
- **Trade-off:** Raise memory / fail-open to Redis quickly vs find root cause (eviction vs key change vs expiry).
- **Example:** Overnight job sets identical TTLs → mass expiry at 3am → avalanche.

**Prompt:** cache hit rate 95% → 60% overnight. What do you do in the first 10 minutes?

1. **Confirm the metric** — which cache cluster/keyspace? Hit rate vs miss rate vs `evicted_keys` / `expired_keys` / OOM.
2. **Traffic shape** — QPS spike? New traffic mix (bot crawl, deploy of new endpoints)? Compare miss QPS to DB QPS.
3. **Eviction storm?** — `used_memory` near max; `evicted_keys` climbing → raise memory / fix big keys / tune `maxmemory-policy`; find who filled Redis.
4. **Mass expiry?** — many keys created with the **same TTL** overnight job → avalanche; jitter TTLs; stagger warm.
5. **Bad deploy / key change?** — new key schema (`user:42` → `user:v2:42`) → cold cache; rollback or dual-read warm.
6. **Invalidation bug** — too-aggressive `FLUSH` / broad `DEL` pattern / pub-sub wipe.
7. **Dependency** — Redis failover emptied node; clients pounding DB → shed load / serve stale if safe.
8. **Stabilize** — rate-limit miss path to DB; warm hottest keys; page owners of overnight jobs.

**Interview line:** “Separate eviction vs expiry vs key-schema miss vs intentional invalidation — metrics tell which in minutes.”

---

## Other cache topics worth knowing

- **Hot key** — one key huge QPS → local cache, split key, read replica of cache
- **Large value** — don’t stuff multi-MB blobs in Redis; use object store + cache URL
- **Serialization** — CPU cost of JSON vs binary; pipeline/batch GETs
- **Aside vs gateway cache** (CDN / HTTP cache-control) — different layer, same invalidation pain
- **Read replica lag** vs cache — don’t treat replica as a coherent cache without care
