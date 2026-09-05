# Estimation Fluency

Back-of-the-envelope math so your design fits the scale you claimed. Interviewers want **order-of-magnitude** correctness and clear assumptions — not calculator precision.

← [README](../README.md) · [Docs index](./README.md)

---

## Index

- [Why estimate?](#why-estimate)
- [Useful constants (memorize these)](#useful-constants-memorize-these)
- [Traffic: DAU → QPS](#traffic-dau-qps)
- [Storage growth](#storage-growth)
- [Bandwidth](#bandwidth)
- [Cache sizing](#cache-sizing)
- [Servers / capacity (rough)](#servers-capacity-rough)
- [Time to fill / drain queues](#time-to-fill-drain-queues)
- [Worked mini-examples](#worked-mini-examples)
- [How to present estimates in a round](#how-to-present-estimates-in-a-round)

---

## Why estimate?

Estimates justify:

- Single DB vs sharded
- Cache size
- Queue throughput
- Number of servers
- Multi-region or not

State assumptions out loud: DAU, requests per user per day, payload size, read/write ratio, retention.

---

## Useful constants (memorize these)

| Quantity | Approx |
|----------|--------|
| Seconds per day | ~10⁵ (86,400) |
| Seconds per month | ~2.5 × 10⁶ |
| Requests/day → QPS | ÷ 10⁵ (rough) |
| 1 KB | 10³ bytes |
| 1 MB | 10⁶ bytes |
| 1 GB | 10⁹ bytes |
| 1 TB | 10¹² bytes |
| Days per year | ~365 ≈ 4 × 10² |

Peak QPS is often **2–5×** average (say so).

---

## Traffic: DAU → QPS

**Average QPS** ≈ `(DAU × actions_per_user_per_day) / 86,400`

Example — social app:

- 100M DAU
- Each user opens feed 10 times/day → 1B feed reads/day
- Avg QPS ≈ `1e9 / 1e5` = **10,000 QPS**
- Peak ≈ **30,000 QPS** (3×)

Split **read QPS** and **write QPS**; they drive different bottlenecks.

---

## Storage growth

**Storage/day** ≈ `writes_per_day × bytes_per_write`  
**Retention** multiplies (e.g. 5 years of logs).

Example — posts:

- 10M new posts/day
- Avg post metadata 1 KB → ~10 GB/day metadata
- Media: 20% of posts have 2 MB image → `0.2 × 10M × 2MB` ≈ **4 TB/day** media → object store + CDN, not the primary DB

Always put **blobs in object storage**; DB holds pointers.

---

## Bandwidth

**Egress** ≈ `QPS × response_size` (and similarly ingress).

Example: 10K QPS × 50 KB JSON ≈ 500 MB/s ≈ **4 Gbps** — may need caching/CDN long before DB dies.

---

## Cache sizing

Estimate **working set**, not full DB.

Example:

- 20% of keys get 80% of traffic
- 50M active keys × 2 KB = **100 GB** → fits a Redis cluster
- Aim for high hit rate on that hot set; state expected hit ratio (e.g. 90%)

Also estimate **QPS the cache must absorb** after hit rate:

`DB_QPS ≈ total_read_QPS × (1 - hit_ratio)`

---

## Servers / capacity (rough)

If one app instance handles **1K QPS** at your p99 budget:

`instances ≈ peak_QPS / 1000` (+ headroom for deploys and failure, often **+50–100%**)

Storage nodes: size by **disk, IOPS, and network**, not only GB.

---

## Time to fill / drain queues

If producers write **P msg/s** and consumers handle **C msg/s**:

- Steady state needs `C ≥ P` (with lag budget)
- Spike of S extra messages drains in `S / (C - P)` seconds if `C > P`

Use this when arguing for autoscaling consumers or buffering with Kafka.

---

## Worked mini-examples

### URL shortener

- 100M new URLs/day → write QPS ≈ `1e8/1e5` ≈ **1K QPS**
- 10:1 read:write → **10K read QPS**
- 8-byte id + metadata ~100 bytes → ~10 GB/day → fine on SQL/NoSQL with cache for hot redirects

### Ticket booking (show night)

- Average low, **peak** huge when sale opens
- Design for peak + **queue / waiting room**; don’t size only on daily average

### Chat

- Connections may dominate: 10M online × 1 connection = **10M concurrent sockets** → dedicated connection tier, not one API box

---

## How to present estimates in a round

1. Write assumptions on the board
2. Compute avg QPS, peak QPS, storage/day, 5-year storage
3. Point at the first bottleneck those numbers imply
4. Offer to refine if the interviewer changes DAU

If you’re unsure of a constant, **say the assumption** and continue — freezing on exact math is worse than a labeled guess.
