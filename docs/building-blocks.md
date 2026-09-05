# Building Blocks

The reusable infrastructure pieces you place on an HLD board. Know **when** to use each, **what it gives you**, and **what it costs** — not every product name.

← [README](../README.md) · [Docs index](./README.md)

---

## Index

- [DNS](#dns)
- [CDN (Content Delivery Network)](#cdn-content-delivery-network)
- [Load balancer (LB)](#load-balancer-lb)
- [Reverse proxy & API gateway](#reverse-proxy-api-gateway)
- [Stateless app servers](#stateless-app-servers)
- [Caching (Redis / Memcached)](#caching-redis-memcached)
- [Databases](#databases)
- [Object storage (S3-style)](#object-storage-s3-style)
- [Message queues & streams](#message-queues-streams)
- [Search engines](#search-engines)
- [Real-time delivery](#real-time-delivery)
- [Observability stack](#observability-stack)
- [How to pick components in an interview](#how-to-pick-components-in-an-interview)

---

## DNS

Maps hostnames to IPs (and often to load balancers).

- **Routing policies:** simple, weighted, latency-based, geo, failover
- **Interview use:** multi-region / failover; *optionally* weighted DNS for coarse blue/green or canary (prefer LB/mesh for fast rollback — see [Deployment strategies](./deployment-and-ops.md#deployment-strategies))
- **Watch out:** TTL vs failover speed; clients cache DNS

## CDN (Content Delivery Network)

Caches content at edge PoPs close to users.

- **Usual:** images, JS/CSS, videos, static pages (rarely change)
- **Also possible:** cacheable **GET** API responses via `Cache-Control` / TTL (public, same for many users, staleness OK)
- **Avoid at CDN:** private/per-user data, POSTs, strong freshness needs
- **CDN vs Redis:** CDN = HTTP edge, global, URL-keyed · Redis = app cache, regional, explicit keys / fine invalidation · often **both** (CDN outside, Redis at origin)
- **Watch out:** invalidation, `Vary`/auth mistakes, signed URLs for private media

## Load balancer (LB)

Spreads traffic across **healthy** backends so no single instance is the bottleneck. See [Load Balancer diagram](../diagrams/load-balancer-hld/load-balancer-hld.excalidraw).

### Where it sits

```text
Client → DNS → (CDN) → Load Balancer → App / API servers → Cache / DB / …
                    ↑
         often TLS terminates here (L7) or passes through (L4)
```

- **Internet-facing LB:** public entry to your fleet (AWS ALB/NLB, GCP LB, nginx/HAProxy).  
- **Internal LB:** service-to-service or between tiers (app → worker pool).  
- **DNS** points at the LB (or Anycast IP), not at individual app VMs.  
- App instances register into a **target group / pool**; LB only sends traffic to **passing health checks**.

### How it works (loop)

1. Client opens connection to LB VIP.  
2. LB picks a backend by **algorithm** + pool membership.  
3. Forwards bytes (L4) or HTTP request (L7); may add headers (`X-Forwarded-For`).  
4. Continuously **health-checks** backends (TCP ping or `GET /health`).  
5. Unhealthy → removed from pool; recovered → re-added.  
6. Optional: **sticky** sessions, draining on deploy, connection limits.

### L4 vs L7

| | **L4 (transport)** | **L7 (application)** |
|--|--------------------|----------------------|
| Sees | IP, TCP/UDP port, connection | HTTP/gRPC: path, host, headers, method, sometimes body |
| Routing | Per connection / 5-tuple | Per request: `/api` vs `/static`, `Host`, cookies |
| TLS | Often pass-through (or TCP TLS) | Commonly **terminates TLS**, may re-encrypt to backend |
| Performance | Very high QPS / low overhead | More CPU (parse HTTP, TLS) |
| Visibility | Blind to URLs / APIs | Rich routing, WAF-ish rules, auth at edge (light) |
| Examples | NLB, L4 HAProxy, LVS | ALB, nginx, Envoy, Cloudflare |

**Neither is universally “better.”**

| Need | Prefer |
|------|--------|
| Raw TCP/UDP, extreme throughput, WebSockets/MQTT at scale, DB proxies | **L4** |
| Path-/host-based routing, canary by header, TLS offload, HTTP retries | **L7** |
| Simple “spread load across identical app replicas” | Either; **L7** if HTTP APIs, **L4** if opaque TCP |
| gRPC / HTTP2 multiplexing awareness | **L7** (L4 only balances connections, not streams) |

**Common pattern:** L4 (or DNS/Anycast) at the outer edge for sheer volume → L7 inside for HTTP routing — or one L7 internet LB if scale fits.

### Balancing algorithms

| Algorithm | Idea | Use when |
|-----------|------|----------|
| **Round-robin** | Rotate backends | Similar instance size, short requests |
| **Least connections** | Prefer least busy | Long-lived or uneven requests |
| **Weighted** | Bigger boxes get more traffic | Mixed instance sizes / canary weights |
| **Sticky (affinity)** | Same client → same backend (cookie / IP hash) | In-memory sessions (prefer externalize session to Redis instead) |
| **Least response time** | Prefer faster backends | Heterogeneous latency |

### Health checks & failure

- **Shallow:** TCP accept / ping — fast, misses “app dead but port open”.  
- **Deep:** HTTP `/health` or `/ready` — checks deps carefully (don’t fail all nodes if one shared DB blips unless intended).  
- On deploy: **connection draining** — stop new traffic, finish in-flight.  
- LB itself is critical → run **HA pair** or managed multi-AZ LB.

### LB vs reverse proxy vs API gateway

| Role | Focus |
|------|--------|
| **LB** | Distribute to many identical (or pooled) backends |
| **Reverse proxy** | TLS, buffering, routing (often *is* your L7 LB: nginx/Envoy) |
| **API gateway** | Product edge: auth, rate limit, API keys, route to *different* services |

In interviews you can draw one box “LB / Gateway” then clarify L4 vs L7 if asked.

### Interview pitfalls

- Sticky sessions as a default → scales poorly; store session in Redis.  
- No health checks → LB keeps hitting dead nodes.  
- Single LB AZ → regional outage.  
- L4 in front of HTTP canaries that need path/header split → won’t work; need L7 or mesh.

## Reverse proxy & API gateway

- **Reverse proxy:** sits in front of apps (nginx, Envoy) — TLS, buffering, routing
- **API gateway:** productized edge for many services — auth, rate limit, request routing, API keys, sometimes aggregation

In interviews, “API Gateway” often means: single entry, authn, rate limiting, then route to microservices.

## Stateless app servers

Compute that holds **no durable session state** on local disk/memory (sessions in Redis/JWT, files in S3).

- Scale by adding instances behind an LB (**horizontal scaling**)
- Prefer this over vertical “bigger box” once you need HA

## Caching (Redis / Memcached)

In-memory store for hot data to cut DB load and latency.

### Patterns

| Pattern | Behavior |
|---------|----------|
| **Cache-aside** | App reads cache → miss → DB → fill cache (most common) |
| **Read-through** | Cache layer loads from DB on miss |
| **Write-through** | Write cache + DB together |
| **Write-behind** | Write cache first; async flush to DB (faster, riskier) |

### Eviction

- **TTL** — expire after time (stale-but-bounded)
- **LRU / LFU** — evict least recently / frequently used when memory full

### Interview pitfalls

- Cache stampede on hot key miss → soft TTL, singleflight, or lock
- Invalidation after writes (delete key vs update)
- Hot keys / celebrities → local cache, replicate, or split key

See [Distributed Caching System](../diagrams/distributed-caching-system/distributed-caching-system.excalidraw).

## Databases

### Relational (Postgres, MySQL, …)

- Strong schema, joins, transactions (ACID)
- Great for payments, bookings, inventory when correctness matters
- Scale: read replicas first; then shard when a single primary is the limit

### Document (MongoDB, DynamoDB document style, …)

- Flexible documents, horizontal scale friendly
- Good when access is by primary key / known patterns
- Avoid if you need heavy ad-hoc joins

### Wide-column / time-series (Cassandra, Bigtable, …)

- High write throughput, partition-keyed access
- Eventual consistency trade-offs vary by product
- Good for timelines, metrics, append-heavy workloads

### Key ideas

- **Index:** speeds reads; costs writes and storage
- **Replication:** HA and read scale; watch replication lag
- **Sharding:** split by key (userId, tenantId); choose key to avoid hot partitions

## Object storage (S3-style)

Blob store for media, backups, large files — cheap, durable, not for low-latency random row queries.

Pattern: upload via pre-signed URL → store URL/metadata in DB → serve via CDN.

## Message queues & streams

Decouple producers from consumers; smooth spikes; enable async work.

| Style | Example | Use |
|-------|---------|-----|
| **Queue** (competing consumers) | SQS, RabbitMQ | One message → one worker |
| **Pub/Sub** | SNS, Redis pub/sub | One message → many subscribers |
| **Log / stream** | Kafka, Kinesis | Ordered partitions, replay, fan-out |

### Delivery semantics

- **At-most-once** — may lose messages
- **At-least-once** — may duplicate → design **idempotent** consumers
- **Exactly-once** — hard; usually “effectively once” via idempotency + dedupe

### Ops concepts

- **Visibility timeout / ack** — hide in-flight messages
- **Retry + DLQ** — poison messages don’t block the queue forever
- **Ordering** — per partition/key (Kafka) or FIFO queues (limited scale)

See [Distributed Queue](../diagrams/distributed-queue/distributed-queue.excalidraw) and [Notification System](../diagrams/notification-system/notification-system.excalidraw).

## Search engines

Inverted index over text (Elasticsearch, OpenSearch) for keyword search, filters, ranking — **not** a replacement for your primary DB.

Typical flow: write to DB → CDC / async indexer → search cluster → query API.

See [Large Scale Search System](../diagrams/large-scale-search-system/large-scale-search-system.excalidraw).

## Real-time delivery

| Approach | Pros | Cons |
|----------|------|------|
| **Short polling** | Simple | Wasteful, higher latency |
| **Long polling** | Better than short poll | Connection overhead |
| **SSE** | Simple server→client stream | One-way; HTTP/2 helps |
| **WebSockets** | Bi-directional, low latency | Sticky LB / connection state; scale connection tier |

Chat systems usually need a **connection / presence service** plus durable message store. See [WhatsApp HLD](../diagrams/whatsapp-hld/whatsapp-hld.excalidraw).

## Observability stack

- **Logs** — what happened (structured JSON)
- **Metrics** — aggregates (QPS, latency, error rate)
- **Traces** — request across services
- **Alerts** — on SLOs, not every blip

See [Logging and Monitoring](../diagrams/logging-and-monitoring-system/logging-and-monitoring-system.excalidraw).

---

## How to pick components in an interview

1. Start with **Client → LB → App → DB**
2. Add **cache** if read-heavy or expensive queries
3. Add **queue** if work can be async or spiky
4. Add **CDN / object store** for media
5. Add **search** only if full-text / ranked search is a requirement
6. Add **WebSockets** only if true real-time is required

Justify each box with a requirement or bottleneck — empty buzzword boxes lose points.

---

## See also

- [Data stores](./data-stores.md) — ACID/BASE, SQL/NoSQL, OLAP, TiDB/TSDB, LSM, Vector DB
- [Messaging & pipelines](./messaging-and-pipelines.md) — Kafka / Rabbit / SQS, CDC, Spark vs Flink
- [Caching deep dive](./caching.md) — stampede, avalanche, eviction
- [Networking & media](./networking-and-media.md) — TCP/UDP, HLS/DASH, RTMP/SRT
- [Deployment & ops](./deployment-and-ops.md) — Docker, Kubernetes, Prometheus/Grafana
- [Algorithms & indexes](./algorithms-and-indexes.md) — Bloom filters, geo indexes
