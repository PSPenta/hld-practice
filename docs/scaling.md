# Scaling, Partitioning & Sharding

How you grow past one box — and how you split data without creating hot spots.

← [README](../README.md) · [Docs index](./README.md)

---

## Index

- [Horizontal scaling: what are you scaling against?](#horizontal-scaling-what-are-you-scaling-against)
- [Partitioning vs sharding](#partitioning-vs-sharding)
- [Consistent hashing](#consistent-hashing)
- [Read replicas vs sharding](#read-replicas-vs-sharding)
- [Stateless vs stateful horizontal scale](#stateless-vs-stateful-horizontal-scale)

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

## Consistent hashing

`hash(key) % N` reshuffles almost every key when N changes. A **hash ring** does not.

Hash servers and keys with the **same** function onto a circle (`0 … 2³²−1`, wraps). A key is owned by the **first server clockwise**. Every client computes that itself — there is no per-key assignment table.

```text
        s1
   k1 ●     ● s2
              k2
   k3 ●     ● s3
     wrap → 0
```

`k1` lives only on `s1`, `k2` only on `s2`, `k3` only on `s3`. Clockwise picks **that one owner**. The key is not copied onto every server.

**Next server is not “S2 always follows S1”.** `s1` and `s2` are names. Ring order is `hash(serverId)`, not the label. `s1`’s clockwise neighbor is whoever hashed next — `s2` or `s3`. Add a node between them and the neighbor changes. Nobody assigns neighbors; every client sorts the same membership list onto the circle and walks clockwise.

**Replica = next distinct servers clockwise** (optional, replication factor R). Store the key on the first R **physical** servers walking clockwise, skipping extra virtual nodes of a server you already picked. If `s1` dies, the copy already on the next server can serve. That is a preference list (Dynamo-style), not a directory of “who is next to whom.”

| Event | What clients do | What data does |
|-------|-----------------|----------------|
| **Add `s4` between `s1` and `s2`** | Rebuild the ring from the new membership. Keys whose first clockwise server is now `s4` go there. `s1`’s neighbor may become `s4` | **Cache:** `s4` starts empty; miss → DB → fill. Old copy on `s2` expires. **DB shard:** background-copy only that arc from `s2` → `s4`, then cut traffic, then delete on `s2` |
| **Remove `s2`** | Those keys’ owner becomes the next clockwise server (`s3` if nothing sits between) | **Cache with replica:** read the copy already on `s3`. **Cache without replica:** miss → DB. **DB shard:** `s3` already has a replica, or you restore from backup and stream the range |

Rebalance does **not** scan “all keys to divert.” Ownership is `owner(key) = first clockwise server`. After membership changes, the next request recomputes it. Only keys on the moved arcs change owner (~1/N, less with virtual nodes).

**Who updates the ring**

| Piece | Who |
|-------|-----|
| Server list | Membership: config push, gossip, etcd/ZooKeeper. All clients must see the same list |
| Neighbor of `s1` | Computed locally from that list. No service stores “S2 is next to S1” |
| Copy of data | Cache: lazy fill. Stateful store: the cluster streams the changed ranges (Cassandra/Dynamo). Hinted handoff covers writes that landed on the wrong node during the blip |

Redis Cluster is **hash slots** (16384), not this ring — same idea (minimal remap), different mechanism. Don’t draw them as the same box.

**Virtual nodes.** One point per server leaves fat arcs. Place each server many times (`s1#0` … `s1#149`). Ownership is still “first clockwise point,” but slices are smaller and more even. Add/remove only moves that server’s arcs. When placing replicas, skip other virtual nodes of a server you already chose.

**vs `% N`:** modulo needs a stable N and remaps on resize. Ring membership changes; key→owner is a function of the current server set, not a stored redirect list.

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
