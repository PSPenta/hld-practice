# Building Blocks

The reusable infrastructure pieces you place on an HLD board. Know **when** to use each, **what it gives you**, and **what it costs** — not every product name.

← [README](../README.md) · [Docs index](./README.md)

---

## Index

- [DNS](#dns)
- [CDN (Content Delivery Network)](#cdn-content-delivery-network)
- [Load balancer (LB)](#load-balancer-lb)
- [Forward vs reverse proxy](#forward-vs-reverse-proxy)
- [VPN vs forward proxy](#vpn-vs-forward-proxy)
- [Common proxy / edge products](#common-proxy-edge-products)
- [API gateway](#api-gateway)
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

## Forward vs reverse proxy

Direction is about **who the proxy represents**, not “more secure.”

```text
Forward:  Client → [Forward proxy] → Internet (any site)
Reverse:  Client → [Reverse proxy / CDN] → Your origin servers
```

| | **Forward proxy** | **Reverse proxy** |
|--|-------------------|-------------------|
| Represents | The **client** (egress) | Your **servers** (ingress) |
| Client config | Explicit proxy URL / PAC / transparent intercept | None — client hits your hostname |
| Typical goals | Hide client IP from sites, allow/deny destinations, log/browse policy | TLS, routing, buffering, load balance, WAF, cache |
| Can block sites? | **Yes** — classic corp/school filter (domain, category, URL) | Blocks/abuses **inbound** to your app (WAF), not “employee can’t open YouTube” |
| HLD board | Rare (unless designing corp egress / SWG) | Common — in front of app fleet |

**Nginx:** usually drawn as **reverse** (`proxy_pass` to upstream apps). It *can* act as a forward proxy, but corps often use Squid / commercial SWGs for that.

**Cloudflare:** orange-cloud DNS + proxy = **reverse** proxy/CDN in front of your origin. Blocking employees from websites = **Cloudflare Gateway / Zero Trust** (forward-proxy / secure web gateway style), not the same product mode.

### Minimal setup shapes (interview depth)

**Reverse (Nginx)** — terminate TLS, forward to app pool:

```nginx
server {
  listen 443 ssl;
  server_name api.example.com;
  location / {
    proxy_pass http://app_upstream;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```

**Reverse (Cloudflare)** — add zone → DNS to origin → proxy (orange cloud) on → optional WAF/cache rules; origin may allowlist only Cloudflare IPs.

**Forward** — clients set `HTTP_PROXY` / PAC to Squid or Nginx forward listener; ACLs deny categories/domains. With Cloudflare: enroll device in Zero Trust / WARP → Gateway HTTP policies allow/block.

---

## VPN vs forward proxy

**Do not say “company VPN = forward proxy.”** They often appear together; they are different layers.

| | **Company VPN** | **Forward proxy** |
|--|-----------------|-------------------|
| What it is | Encrypted **network tunnel** onto corp/cloud network | **Application hop** for (usually) HTTP(S) |
| Layer | Mostly L3/L4 (WireGuard, IPsec, OpenVPN, …) | Mostly L7 for web (explicit proxy / CONNECT) |
| Primary job | Reach **private** apps, corp DNS; optionally force all IP egress via corp | Policy on **which sites** clients may open; inspect/log web |
| Site blocking | Via firewall / DNS / gateway **after** the tunnel | Via proxy ACLs / SWG policies on requests |

**Common corp pattern:** VPN (or Zero Trust agent) → then **forced** through a forward proxy / secure web gateway (Zscaler, Cloudflare Gateway, Blue Coat). Private access comes from the tunnel; web allow/deny comes from the proxy/SWG.

**Interview line:** *“VPN is a tunnel. Forward proxy is an HTTP intermediary. Corps combine them: VPN for private access, forward proxy/SWG for outbound web control.”*

---

## Common proxy / edge products

| Product | Typical role in HLD |
|---------|---------------------|
| **Nginx** / **HAProxy** / **Envoy** | Reverse proxy + L7 LB; Envoy also service mesh sidecar |
| **AWS ALB** / **NLB**, GCP/Azure LBs | Managed reverse LB (L7 / L4) |
| **Cloudflare**, Fastly, Akamai | Reverse proxy + CDN + WAF at edge |
| **Squid**, commercial SWG (Zscaler, Blue Coat) | Forward proxy / web filter |
| **Cloudflare Gateway**, Zscaler | Forward-proxy-like secure web gateway (+ often with Zero Trust agent) |
| **API Gateway** (Kong, AWS API GW, Apigee) | Product edge: auth, rate limit, route to many services |

---

## API gateway

Productized **reverse** edge for many services — authn, rate limiting, API keys, request routing, sometimes aggregation.

In interviews, “API Gateway” often means: single entry, authn, rate limiting, then route to microservices. Draw it when you have **many different** backends; a plain reverse proxy/LB is enough for identical app replicas.

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

**Keyword:** inverted index (Elasticsearch / OpenSearch) — BM25, filters, shards.  
**Semantic:** embeddings + ANN (vector DB or ES `dense_vector`).  
Often **hybrid**. Not a replacement for your primary DB.

Typical flow: write to DB → CDC / async indexer → search cluster → query API.

Depth: [Keyword & inverted indexes](./algorithms-and-indexes.md#keyword-search-inverted-indexes-elasticsearch) · [Semantic search](./algorithms-and-indexes.md#semantic-search) · diagram: [Large Scale Search](../diagrams/large-scale-search-system/large-scale-search-system.excalidraw).

## Real-time delivery

| Approach | Pros | Cons |
|----------|------|------|
| **Short polling** | Simple | Wasteful, higher latency |
| **Long polling** | Better than short poll | Connection overhead |
| **SSE** | Simple server→client stream | One-way; HTTP/2 helps |
| **WebSockets** | Bi-directional, low latency | Sticky LB / connection state; scale connection tier |

Chat systems usually need a **connection / presence service** plus durable message store. See [WhatsApp HLD](../diagrams/whatsapp-hld/whatsapp-hld.excalidraw).

## Observability stack

- **Logs** — what happened (structured JSON) → ELK / Loki  
- **Metrics** — aggregates (QPS, latency, error rate) → Prometheus + Grafana  
- **Traces** — request across services → Jaeger / Tempo  
- **Alerts** — on SLOs, not every blip  

ELK vs Prom/Grafana: [Deployment & ops](./deployment-and-ops.md#elk-vs-prometheus-grafana). Diagram: [Logging and Monitoring](../diagrams/logging-and-monitoring-system/logging-and-monitoring-system.excalidraw).

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
