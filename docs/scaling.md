# Scaling, Partitioning & Sharding

How you grow past one box — and how you split data without creating hot spots.

← [README](../README.md) · [Docs index](./README.md)

---

## Horizontal scaling: what are you scaling against?

“Add more machines” only helps if you know the **bottleneck**.

| Pressure | Symptom | Scale / fix |
|----------|---------|-------------|
| **Requests/sec (RPS)** | High QPS, CPU OK, queues at LB | More app replicas behind LB; cache; async |
| **CPU** | High compute (encode, crypto, JSON) | More replicas **or** bigger CPUs; move work async; optimize |
| **Memory** | Cache/heap growth, OOM | More nodes with sharded cache; reduce payload; streaming |
| **Disk IOPS / space** | DB slow writes, full disks | Shard DB; object store for blobs; provisioned IOPS |
| **Network** | Bandwidth saturated | CDN; compress; regionalize; fewer chattiness hops |
| **Connections** | Too many sockets (chat) | Dedicated connection tier; pool; HTTP/2 |

**Interview habit:** measure/state the limiting resource, then scale **that** dimension. Blindly “add Kubernetes pods” won’t fix a single-primary DB write bottleneck.

Vertical scaling (bigger machine) is fine early; horizontal wins for HA and linear-ish growth of **stateless** tiers.

---

## Partitioning vs sharding

People use them interchangeably; be precise when it matters.

| Term | Usual meaning |
|------|----------------|
| **Partitioning** | Split data into pieces by a key (in one DB product or logically) — e.g. Postgres partitions by date |
| **Sharding** | Partitions spread across **separate database instances / clusters** (shared-nothing) |

All shards are partitions; not all partitions are separate shards.

### Common partition/shard keys

- `userId`, `tenantId`, `deviceId`, `orderId`
- Time buckets for logs/metrics (range partitioning)

### Strategies

| Strategy | Idea | Trade-off |
|----------|------|-----------|
| **Hash** | `hash(key) % N` | Even load; range queries hard |
| **Range** | `A–M` / `N–Z`, dates | Range scans easy; hot ranges possible |
| **Directory / lookup** | Map key → shard in a service | Flexible moves; lookup is critical path |
| **Consistent hashing** | Minimal remaps when nodes add/remove | Used in caches, some DBs |

### Hot partitions

Celebrity `userId`, “today” time bucket, popular `showId` → one shard melts.

Mitigate: salt keys, separate hot path, fan-out on read, dedicated pools.

---

## Consistent hashing (quick)

Nodes on a hash ring; keys map to successor node. Adding/removing a node moves only nearby keys (with **virtual nodes** for balance). Classic for distributed caches.

---

## Read replicas vs sharding

- **Replicas** — scale **reads**, HA; writes still hit primary; lag possible
- **Sharding** — scale **writes and storage**; cross-shard joins/transactions hurt

Order of escalation in many designs: indexes → cache → read replicas → shard.

---

## Stateless vs stateful horizontal scale

- **Stateless app** — clone freely behind LB
- **Stateful** (DB, Kafka broker, Redis with data) — needs partitioning, rebalancing, replication protocols

Chat connection servers are “sticky” state (socket maps) — scale with **pools + routing**, not naive round-robin alone. See [WhatsApp HLD](../diagrams/whatsapp-hld/whatsapp-hld.excalidraw).
