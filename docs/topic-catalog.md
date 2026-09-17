# Topic catalog (sneak peek)

Hierarchical index of concepts under `docs/`. Study order stays in [Docs index](./README.md).

← [README](../README.md) · [Docs index](./README.md)

---

## Groups

- [Interview craft](#interview-craft)
- [Foundation](#foundation)
- [Data & messaging](#data-messaging)
- [Reliability & correctness](#reliability-correctness)
- [Delivery & operations](#delivery-operations)

---

## Interview craft

- **[Soft Skills (Interview Communication)](./soft-skills.md)**
  - [Drive the conversation](./soft-skills.md#drive-the-conversation)
  - [State assumptions explicitly](./soft-skills.md#state-assumptions-explicitly)
  - [Prefer simple, then scale the bottleneck](./soft-skills.md#prefer-simple-then-scale-the-bottleneck)
  - [Compare two options, then commit](./soft-skills.md#compare-two-options-then-commit)
  - [Use the board clearly](./soft-skills.md#use-the-board-clearly)
  - [Handle pushback well](./soft-skills.md#handle-pushback-well)
  - [Time management](./soft-skills.md#time-management)
  - [SDE3 / Staff communication bar](./soft-skills.md#sde3-staff-communication-bar)
  - [Language that scores](./soft-skills.md#language-that-scores)
  - [Language that hurts](./soft-skills.md#language-that-hurts)
  - [Practice drill](./soft-skills.md#practice-drill)

- **[Staff HLD Vocabulary](./staff-vocabulary.md)**
  - [Upstream vs downstream (in your HLD)](./staff-vocabulary.md#upstream-vs-downstream-in-your-hld)
  - [Dependency chain on the board](./staff-vocabulary.md#dependency-chain-on-the-board)
  - [Traffic & capacity](./staff-vocabulary.md#traffic-capacity)
  - [Latency & overload](./staff-vocabulary.md#latency-overload)
  - [Isolation & failure spread](./staff-vocabulary.md#isolation-failure-spread)
  - [Queues & async paths](./staff-vocabulary.md#queues-async-paths)
  - [How to use this in a round](./staff-vocabulary.md#how-to-use-this-in-a-round)

- **[Estimation Fluency](./estimation-fluency.md)**
  - [Why estimate?](./estimation-fluency.md#why-estimate)
  - [Useful constants (memorize these)](./estimation-fluency.md#useful-constants-memorize-these)
  - [Traffic: DAU → QPS](./estimation-fluency.md#traffic-dau-qps)
  - [Storage growth](./estimation-fluency.md#storage-growth)
  - [Bandwidth](./estimation-fluency.md#bandwidth)
  - [Cache sizing](./estimation-fluency.md#cache-sizing)
  - [Servers / capacity (rough)](./estimation-fluency.md#servers-capacity-rough)
  - [Time to fill / drain queues](./estimation-fluency.md#time-to-fill-drain-queues)
  - [Worked mini-examples](./estimation-fluency.md#worked-mini-examples)
    - [URL shortener](./estimation-fluency.md#url-shortener)
    - [Ticket booking (show night)](./estimation-fluency.md#ticket-booking-show-night)
    - [Chat](./estimation-fluency.md#chat)
  - [How to present estimates in a round](./estimation-fluency.md#how-to-present-estimates-in-a-round)

- **[Interview Prep Plan (DSA · LLD · HLD)](./prep-plan.md)**
  - [What this plan covers](./prep-plan.md#what-this-plan-covers)
  - [Are these resources enough?](./prep-plan.md#are-these-resources-enough)
  - [Day window (11:00 – 21:00)](./prep-plan.md#day-window-1100-2100)
  - [Daily targets](./prep-plan.md#daily-targets)
  - [Weekly rhythm](./prep-plan.md#weekly-rhythm)
  - [Weekly targets](./prep-plan.md#weekly-targets)
  - [Theory revision (1–2× / week)](./prep-plan.md#theory-revision-1-2-week)
    - [Session (60–90m)](./prep-plan.md#session-60-90m)
    - [First-pass order](./prep-plan.md#first-pass-order)
  - [Language via DSA & LLD](./prep-plan.md#language-via-dsa-lld)
  - [8–10 week phases](./prep-plan.md#8-10-week-phases)
  - [Win condition](./prep-plan.md#win-condition)

## Foundation

- **[Building Blocks](./building-blocks.md)**
  - [DNS](./building-blocks.md#dns)
  - [CDN (Content Delivery Network)](./building-blocks.md#cdn-content-delivery-network)
  - [Load balancer (LB)](./building-blocks.md#load-balancer-lb)
    - [Where it sits](./building-blocks.md#where-it-sits)
    - [How it works (loop)](./building-blocks.md#how-it-works-loop)
    - [L4 vs L7](./building-blocks.md#l4-vs-l7)
    - [Balancing algorithms](./building-blocks.md#balancing-algorithms)
    - [Health checks & failure](./building-blocks.md#health-checks-failure)
    - [LB vs reverse proxy vs API gateway](./building-blocks.md#lb-vs-reverse-proxy-vs-api-gateway)
    - [Interview pitfalls](./building-blocks.md#interview-pitfalls)
  - [Forward vs reverse proxy](./building-blocks.md#forward-vs-reverse-proxy)
    - [Minimal setup shapes (interview depth)](./building-blocks.md#minimal-setup-shapes-interview-depth)
  - [VPN vs forward proxy](./building-blocks.md#vpn-vs-forward-proxy)
  - [Common proxy / edge products](./building-blocks.md#common-proxy-edge-products)
  - [API gateway](./building-blocks.md#api-gateway)
  - [Stateless app servers](./building-blocks.md#stateless-app-servers)
  - [Caching (Redis / Memcached)](./building-blocks.md#caching-redis-memcached)
    - [Patterns](./building-blocks.md#patterns)
    - [Eviction](./building-blocks.md#eviction)
    - [Interview pitfalls](./building-blocks.md#interview-pitfalls)
  - [Databases](./building-blocks.md#databases)
    - [Relational (Postgres, MySQL, …)](./building-blocks.md#relational-postgres-mysql)
    - [Document (MongoDB, DynamoDB document style, …)](./building-blocks.md#document-mongodb-dynamodb-document-style)
    - [Wide-column / time-series (Cassandra, Bigtable, …)](./building-blocks.md#wide-column-time-series-cassandra-bigtable)
    - [Key ideas](./building-blocks.md#key-ideas)
  - [Object storage (S3-style)](./building-blocks.md#object-storage-s3-style)
  - [Message queues & streams](./building-blocks.md#message-queues-streams)
    - [Delivery semantics](./building-blocks.md#delivery-semantics)
    - [Ops concepts](./building-blocks.md#ops-concepts)
  - [Search engines](./building-blocks.md#search-engines)
  - [Real-time delivery](./building-blocks.md#real-time-delivery)
  - [Observability stack](./building-blocks.md#observability-stack)
  - [How to pick components in an interview](./building-blocks.md#how-to-pick-components-in-an-interview)

- **[Service Architecture](./service-architecture.md)**
  - [Monolith vs microservices](./service-architecture.md#monolith-vs-microservices)
  - [Services vs workers](./service-architecture.md#services-vs-workers)
  - [Where to use services vs workers](./service-architecture.md#where-to-use-services-vs-workers)
  - [Beyond services & workers](./service-architecture.md#beyond-services-workers)
  - [Cron / scheduler vs Temporal](./service-architecture.md#cron-scheduler-vs-temporal)
  - [DB topology & connections](./service-architecture.md#db-topology-connections)
    - [Do service and workers share one DB?](./service-architecture.md#do-service-and-workers-share-one-db)
    - [Connection pools](./service-architecture.md#connection-pools)

- **[Core Concepts](./core-concepts.md)**
  - [CAP and PACELC (practical view)](./core-concepts.md#cap-and-pacelc-practical-view)
  - [Consistency models](./core-concepts.md#consistency-models)
  - [ACID vs BASE](./core-concepts.md#acid-vs-base)
  - [CRDT (Conflict-free Replicated Data Type)](./core-concepts.md#crdt-conflict-free-replicated-data-type)
  - [Idempotency](./core-concepts.md#idempotency)
  - [Optimistic locking & versioning](./core-concepts.md#optimistic-locking-versioning)
  - [Latency vs throughput](./core-concepts.md#latency-vs-throughput)
  - [Latency & metrics vocabulary](./core-concepts.md#latency-metrics-vocabulary)
  - [Availability & failure modes](./core-concepts.md#availability-failure-modes)
  - [Partitioning & hot keys](./core-concepts.md#partitioning-hot-keys)
  - [Rate limiting](./core-concepts.md#rate-limiting)
  - [Backpressure](./core-concepts.md#backpressure)
  - [Exponential backoff](./core-concepts.md#exponential-backoff)
  - [Circuit breaker](./core-concepts.md#circuit-breaker)
  - [Security basics (HLD depth)](./core-concepts.md#security-basics-hld-depth)
  - [Fan-out on write vs read](./core-concepts.md#fan-out-on-write-vs-read)
  - [Sync vs async](./core-concepts.md#sync-vs-async)
  - [How to talk about trade-offs](./core-concepts.md#how-to-talk-about-trade-offs)

- **[Scaling, Partitioning & Sharding](./scaling.md)**
  - [Horizontal scaling: what are you scaling against?](./scaling.md#horizontal-scaling-what-are-you-scaling-against)
  - [Partitioning vs sharding](./scaling.md#partitioning-vs-sharding)
    - [Common partition/shard keys](./scaling.md#common-partitionshard-keys)
    - [Strategies](./scaling.md#strategies)
    - [Hot partitions](./scaling.md#hot-partitions)
  - [Consistent hashing](./scaling.md#consistent-hashing)
  - [Read replicas vs sharding](./scaling.md#read-replicas-vs-sharding)
  - [Stateless vs stateful horizontal scale](./scaling.md#stateless-vs-stateful-horizontal-scale)

- **[Networking & Media Streaming](./networking-and-media.md)**
  - [TCP vs UDP](./networking-and-media.md#tcp-vs-udp)
  - [HLS vs DASH](./networking-and-media.md#hls-vs-dash)
  - [RTMP vs SRT](./networking-and-media.md#rtmp-vs-srt)
  - [Related networking prerequisites](./networking-and-media.md#related-networking-prerequisites)

## Data & messaging

- **[Data Stores](./data-stores.md)**
  - [ACID vs BASE](./data-stores.md#acid-vs-base)
  - [SQL vs NoSQL](./data-stores.md#sql-vs-nosql)
  - [OLTP vs OLAP](./data-stores.md#oltp-vs-olap)
  - [TiDB (distributed SQL) vs TSDB (time-series DB)](./data-stores.md#tidb-distributed-sql-vs-tsdb-time-series-db)
    - [TiDB](./data-stores.md#tidb)
    - [TSDB (Time-Series Database)](./data-stores.md#tsdb-time-series-database)
  - [Downsampling](./data-stores.md#downsampling)
  - [Knowledge base vs Vector DB](./data-stores.md#knowledge-base-vs-vector-db)
  - [LSM trees (storage engine)](./data-stores.md#lsm-trees-storage-engine)
  - [DB deployment strategies](./data-stores.md#db-deployment-strategies)
    - [Single region vs multi-region](./data-stores.md#single-region-vs-multi-region)
    - [Replication in multi-region](./data-stores.md#replication-in-multi-region)
    - [Read-heavy vs write-heavy](./data-stores.md#read-heavy-vs-write-heavy)
  - [Quick chooser](./data-stores.md#quick-chooser)

- **[Caching (Deep Dive)](./caching.md)**
  - [Quick recap of patterns](./caching.md#quick-recap-of-patterns)
  - [Cache stampede (a.k.a. dogpile / thundering herd)](./caching.md#cache-stampede-aka-dogpile-thundering-herd)
  - [Cache avalanche](./caching.md#cache-avalanche)
  - [Cache penetration](./caching.md#cache-penetration)
  - [Cache invalidation](./caching.md#cache-invalidation)
  - [Eviction policies](./caching.md#eviction-policies)
  - [Other cache topics worth knowing](./caching.md#other-cache-topics-worth-knowing)

- **[Messaging & Data Pipelines](./messaging-and-pipelines.md)**
  - [Pub/Sub vs Message Queue](./messaging-and-pipelines.md#pubsub-vs-message-queue)
  - [Kafka vs RabbitMQ vs SQS](./messaging-and-pipelines.md#kafka-vs-rabbitmq-vs-sqs)
  - [Cluster metadata & coordination (ZooKeeper, KRaft, and alternatives)](./messaging-and-pipelines.md#cluster-metadata-coordination-zookeeper-kraft-and-alternatives)
  - [AWS Kinesis](./messaging-and-pipelines.md#aws-kinesis)
  - [Lambda vs Kappa architecture](./messaging-and-pipelines.md#lambda-vs-kappa-architecture)
  - [Write-ahead log (WAL) & MySQL binlog](./messaging-and-pipelines.md#write-ahead-log-wal-mysql-binlog)
  - [CDC (Change Data Capture)](./messaging-and-pipelines.md#cdc-change-data-capture)
  - [Event aggregator (Spark) vs Stream aggregator (Flink)](./messaging-and-pipelines.md#event-aggregator-spark-vs-stream-aggregator-flink)
  - [Related patterns (also prerequisites)](./messaging-and-pipelines.md#related-patterns-also-prerequisites)
    - [Transactional outbox](./messaging-and-pipelines.md#transactional-outbox)
    - [Inbox / dedupe table](./messaging-and-pipelines.md#inbox-dedupe-table)
    - [Event sourcing (light)](./messaging-and-pipelines.md#event-sourcing-light)
    - [CQRS (light)](./messaging-and-pipelines.md#cqrs-light)
    - [Log compaction (Kafka)](./messaging-and-pipelines.md#log-compaction-kafka)

- **[Algorithms, Indexes & Crypto Basics](./algorithms-and-indexes.md)**
  - [Bloom filters](./algorithms-and-indexes.md#bloom-filters)
  - [Hashing vs encryption](./algorithms-and-indexes.md#hashing-vs-encryption)
  - [Geo-spatial indexes](./algorithms-and-indexes.md#geo-spatial-indexes)
    - [Geohash](./algorithms-and-indexes.md#geohash)
    - [H3 (Uber)](./algorithms-and-indexes.md#h3-uber)
    - [PostGIS (and geo types in SQL)](./algorithms-and-indexes.md#postgis-and-geo-types-in-sql)
    - [Quadtree / R-tree (concepts)](./algorithms-and-indexes.md#quadtree-r-tree-concepts)
  - [Proximity search (“nearby”)](./algorithms-and-indexes.md#proximity-search-nearby)
  - [Keyword search & inverted indexes (Elasticsearch)](./algorithms-and-indexes.md#keyword-search-inverted-indexes-elasticsearch)
  - [Semantic search](./algorithms-and-indexes.md#semantic-search)
  - [Choosing proximity vs keyword vs semantic](./algorithms-and-indexes.md#choosing-proximity-vs-keyword-vs-semantic)
  - [Other index / structure prerequisites](./algorithms-and-indexes.md#other-index-structure-prerequisites)

## Reliability & correctness

- **[Distributed Coordination & Transactions](./distributed-coordination.md)**
  - [Race conditions](./distributed-coordination.md#race-conditions)
  - [Distributed transactions](./distributed-coordination.md#distributed-transactions)
  - [Two-phase commit (2PC) & 3PC](./distributed-coordination.md#two-phase-commit-2pc-3pc)
    - [2PC](./distributed-coordination.md#2pc)
    - [3PC](./distributed-coordination.md#3pc)
  - [Saga](./distributed-coordination.md#saga)
  - [Raft (consensus)](./distributed-coordination.md#raft-consensus)
  - [Leader / coordinator dies mid-transaction](./distributed-coordination.md#leader-coordinator-dies-mid-transaction)
    - [2PC coordinator dies](./distributed-coordination.md#2pc-coordinator-dies)
    - [Raft / etcd-style leader dies](./distributed-coordination.md#raft-etcd-style-leader-dies)
    - [Saga orchestrator dies](./distributed-coordination.md#saga-orchestrator-dies)
    - [Interview line](./distributed-coordination.md#interview-line)
  - [ZooKeeper / etcd / Consul (coordination services)](./distributed-coordination.md#zookeeper-etcd-consul-coordination-services)
  - [Other coordination prerequisites](./distributed-coordination.md#other-coordination-prerequisites)

- **[Reliability & SLOs](./reliability-and-slos.md)**
  - [SLI, SLO, SLA](./reliability-and-slos.md#sli-slo-sla)
  - [Error budgets](./reliability-and-slos.md#error-budgets)
  - [Failure modes checklist](./reliability-and-slos.md#failure-modes-checklist)
  - [Blast radius](./reliability-and-slos.md#blast-radius)
  - [Load shedding & graceful degradation](./reliability-and-slos.md#load-shedding-graceful-degradation)
  - [Multi-AZ vs multi-region](./reliability-and-slos.md#multi-az-vs-multi-region)
  - [Exponential backoff](./reliability-and-slos.md#exponential-backoff)
  - [Backpressure](./reliability-and-slos.md#backpressure)
  - [Reliability in fintech (Razorpay-class)](./reliability-and-slos.md#reliability-in-fintech-razorpay-class)

- **[Security & Compliance](./security-and-compliance.md)**
  - [Authn vs Authz](./security-and-compliance.md#authn-vs-authz)
  - [Tokens & sessions](./security-and-compliance.md#tokens-sessions)
  - [Transport & data protection](./security-and-compliance.md#transport-data-protection)
  - [Secrets & least privilege](./security-and-compliance.md#secrets-least-privilege)
  - [PCI-DSS mindset (payments interviews)](./security-and-compliance.md#pci-dss-mindset-payments-interviews)
  - [Threat modeling (lightweight STRIDE)](./security-and-compliance.md#threat-modeling-lightweight-stride)
  - [Abuse & fraud controls](./security-and-compliance.md#abuse-fraud-controls)
  - [Privacy](./security-and-compliance.md#privacy)

## Delivery & operations

- **[Deployment & Operations](./deployment-and-ops.md)**
  - [Docker](./deployment-and-ops.md#docker)
  - [Kubernetes (K8s)](./deployment-and-ops.md#kubernetes-k8s)
  - [Deployment strategies](./deployment-and-ops.md#deployment-strategies)
  - [Monitoring: Prometheus, Grafana & ELK](./deployment-and-ops.md#monitoring-prometheus-grafana-elk)
    - [Prometheus](./deployment-and-ops.md#prometheus)
    - [Grafana](./deployment-and-ops.md#grafana)
    - [ELK stack](./deployment-and-ops.md#elk-stack)
    - [ELK vs Prometheus & Grafana](./deployment-and-ops.md#elk-vs-prometheus-grafana)
    - [Full observability trio](./deployment-and-ops.md#full-observability-trio)
  - [SLIs, SLOs, SLAs (ops vocabulary)](./deployment-and-ops.md#slis-slos-slas-ops-vocabulary)
  - [Other ops prerequisites](./deployment-and-ops.md#other-ops-prerequisites)

---

_`##` + useful `###` nested. Single-link (no subpoints): Circuit breaker, Exponential backoff (Reliability), Proximity / Keyword / Semantic search, Cluster metadata & coordination — detail stays in those docs._
