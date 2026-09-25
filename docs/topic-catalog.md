# Topic catalog

**Central index of every HLD topic in `docs/`.** Pick any bullet — the link opens the detailed explanation. Study order: [Docs index](./README.md) (curriculum A→E).

← [README](../README.md) · [Docs index](./README.md)

**How to use:** Ctrl/Cmd+F a keyword → click the link. Anchors match each doc’s own **Index** (see [format convention](./README.md#format-convention-all-docs): `&` removed, not turned into `--`, except where a doc’s Index already uses `--`).

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
  - [Resources map](./prep-plan.md#resources-map)
  - [Day window (full-time)](./prep-plan.md#day-window-full-time)
  - [Daily targets](./prep-plan.md#daily-targets)
  - [Weekly rhythm](./prep-plan.md#weekly-rhythm)
  - [Weekly targets](./prep-plan.md#weekly-targets)
  - [Spaced revision (HLD + LLD lang docs)](./prep-plan.md#spaced-revision-hld--lld-lang-docs)
    - [Daily 20m (pick one lane)](./prep-plan.md#daily-20m-pick-one-lane)
    - [Deep theory (2×/week, 60–90m)](./prep-plan.md#deep-theory-2week-60-90m)
    - [First-pass HLD order (weeks 1–4)](./prep-plan.md#first-pass-hld-order-weeks-1-4)
    - [Weekend revision (1×)](./prep-plan.md#weekend-revision-1)
  - [DSA — Striver topic rotation](./prep-plan.md#dsa--striver-topic-rotation)
  - [LLD — language docs then problems](./prep-plan.md#lld--language-docs-then-problems)
  - [HLD — theory + spoken redesign](./prep-plan.md#hld--theory--spoken-redesign)
  - [12-week phases](./prep-plan.md#12-week-phases)
  - [Staff / SDE3 bar (weekly self-check)](./prep-plan.md#staff--sde3-bar-weekly-self-check)
  - [Win condition](./prep-plan.md#win-condition)

---

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
  - [Caching (Redis / Memcached)](./building-blocks.md#caching-redis-memcached) — overview; deep dive → [Caching](./caching.md)
    - [Patterns](./building-blocks.md#patterns)
    - [Eviction](./building-blocks.md#eviction)
  - [Databases](./building-blocks.md#databases)
    - [Relational (Postgres, MySQL, …)](./building-blocks.md#relational-postgres-mysql)
    - [Document (MongoDB, DynamoDB document style, …)](./building-blocks.md#document-mongodb-dynamodb-document-style)
    - [Wide-column / time-series (Cassandra, Bigtable, …)](./building-blocks.md#wide-column-time-series-cassandra-bigtable)
    - [Key ideas](./building-blocks.md#key-ideas)
  - [Object storage (S3-style)](./building-blocks.md#object-storage-s3-style)
  - [Message queues & streams](./building-blocks.md#message-queues-streams)
    - [Delivery semantics](./building-blocks.md#delivery-semantics) — at-least-once / effectively once
    - [Ops concepts](./building-blocks.md#ops-concepts)
  - [Search engines](./building-blocks.md#search-engines)
  - [Real-time delivery](./building-blocks.md#real-time-delivery)
  - [Observability stack](./building-blocks.md#observability-stack)
  - [How to pick components in an interview](./building-blocks.md#how-to-pick-components-in-an-interview)

- **[Service Architecture](./service-architecture.md)**
  - [Monolith vs microservices](./service-architecture.md#monolith-vs-microservices)
  - [Services vs workers](./service-architecture.md#services-vs-workers)
  - [Where to use services vs workers](./service-architecture.md#where-to-use-services-vs-workers)
  - [API contracts other teams depend on](./service-architecture.md#api-contracts-other-teams-depend-on)
    - [Evolving without breaking them](./service-architecture.md#evolving-without-breaking-them)
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
    - [Where to enforce (request path)](./core-concepts.md#where-to-enforce-request-path) — retried POST key
  - [Optimistic locking & versioning](./core-concepts.md#optimistic-locking-versioning)
  - [Latency vs throughput](./core-concepts.md#latency-vs-throughput)
  - [Latency & metrics vocabulary](./core-concepts.md#latency-metrics-vocabulary)
  - [Availability & failure modes](./core-concepts.md#availability-failure-modes)
  - [Partitioning & hot keys](./core-concepts.md#partitioning-hot-keys)
  - [Rate limiting](./core-concepts.md#rate-limiting)
  - [Backpressure](./core-concepts.md#backpressure)
  - [Exponential backoff](./core-concepts.md#exponential-backoff)
  - [Circuit breaker](./core-concepts.md#circuit-breaker)
    - [Problem it solves](./core-concepts.md#problem-it-solves)
    - [Trip criteria (strategies)](./core-concepts.md#trip-criteria-strategies)
    - [Scope: what one breaker protects](./core-concepts.md#scope-what-one-breaker-protects)
    - [Use cases: Redis, DB, both](./core-concepts.md#use-cases-redis-db-both)
    - [How to implement](./core-concepts.md#how-to-implement)
    - [Fallbacks (pick per dependency)](./core-concepts.md#fallbacks-pick-per-dependency)
    - [Interview pitfalls](./core-concepts.md#interview-pitfalls)
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

---

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

- **[AI Systems (RAG / LLM in products)](./ai-systems.md)**
  - [RAG pipeline (recap)](./ai-systems.md#rag-pipeline-recap)
  - [What breaks first in production RAG](./ai-systems.md#what-breaks-first-in-production-rag)
  - [Evaluate retrieval separately from generation](./ai-systems.md#evaluate-retrieval-separately-from-generation)
  - [When an LLM should not be in the request path](./ai-systems.md#when-an-llm-should-not-be-in-the-request-path)
  - [Keep LLM out of the path but still use it](./ai-systems.md#keep-llm-out-of-the-path-but-still-use-it)

- **[Caching (Deep Dive)](./caching.md)**
  - [Quick recap of patterns](./caching.md#quick-recap-of-patterns) — when cache-aside vs write-through
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
    - [What ZooKeeper did for Kafka (classic)](./messaging-and-pipelines.md#what-zookeeper-did-for-kafka-classic)
    - [KRaft (Kafka without ZooKeeper)](./messaging-and-pipelines.md#kraft-kafka-without-zookeeper)
    - [Alternatives: who does this job in SQS and RabbitMQ?](./messaging-and-pipelines.md#alternatives-who-does-this-job-in-sqs-and-rabbitmq)
    - [HLD board cheat sheet](./messaging-and-pipelines.md#hld-board-cheat-sheet)
  - [AWS Kinesis](./messaging-and-pipelines.md#aws-kinesis)
  - [Lambda vs Kappa architecture](./messaging-and-pipelines.md#lambda-vs-kappa-architecture)
  - [Write-ahead log (WAL) & MySQL binlog](./messaging-and-pipelines.md#write-ahead-log-wal-mysql-binlog)
  - [CDC (Change Data Capture)](./messaging-and-pipelines.md#cdc-change-data-capture)
  - [Event aggregator (Spark) vs Stream aggregator (Flink)](./messaging-and-pipelines.md#event-aggregator-spark-vs-stream-aggregator-flink)
  - [Related patterns (also prerequisites)](./messaging-and-pipelines.md#related-patterns-also-prerequisites)
    - [Transactional outbox](./messaging-and-pipelines.md#transactional-outbox)
    - [Inbox / dedupe table](./messaging-and-pipelines.md#inbox-dedupe-table) — payment consumer idempotency
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
  - [Why a query with an index can still be slow](./algorithms-and-indexes.md#why-a-query-with-an-index-can-still-be-slow)
    - [How to prove which reason it is](./algorithms-and-indexes.md#how-to-prove-which-reason-it-is) — `EXPLAIN ANALYZE`
  - [Other index / structure prerequisites](./algorithms-and-indexes.md#other-index-structure-prerequisites)

---

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
  - [Error budgets](./reliability-and-slos.md#error-budgets) — what to do when exhausted
  - [Failure modes checklist](./reliability-and-slos.md#failure-modes-checklist)
  - [Blast radius](./reliability-and-slos.md#blast-radius)
  - [Load shedding & graceful degradation](./reliability-and-slos.md#load-shedding-graceful-degradation)
  - [Multi-AZ vs multi-region](./reliability-and-slos.md#multi-az-vs-multi-region)
  - [Exponential backoff](./reliability-and-slos.md#exponential-backoff)
    - [Formula (typical)](./reliability-and-slos.md#formula-typical)
    - [When to use](./reliability-and-slos.md#when-to-use)
    - [When **not** to blind-retry](./reliability-and-slos.md#when-not-to-blind-retry)
    - [Pair with](./reliability-and-slos.md#pair-with)
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

---

## Delivery & operations

- **[Deployment & Operations](./deployment-and-ops.md)**
  - [Docker](./deployment-and-ops.md#docker)
  - [Kubernetes (K8s)](./deployment-and-ops.md#kubernetes-k8s)
    - [Cluster, Node, Pod](./deployment-and-ops.md#cluster-node-pod)
    - [Kubelet & Kube-proxy](./deployment-and-ops.md#kubelet--kube-proxy)
    - [Workloads](./deployment-and-ops.md#workloads) — Deployment, ReplicaSet, StatefulSet, DaemonSet, Job, CronJob
    - [Service types & Ingress](./deployment-and-ops.md#service-types--ingress) — ClusterIP, NodePort, LoadBalancer
    - [ConfigMap & Secret](./deployment-and-ops.md#configmap--secret)
    - [Volumes, PV & PVC](./deployment-and-ops.md#volumes-pv--pvc)
    - [Namespace](./deployment-and-ops.md#namespace)
    - [Horizontal Pod Autoscaler (HPA)](./deployment-and-ops.md#horizontal-pod-autoscaler-hpa)
    - [Probes, resources & Staff gotchas](./deployment-and-ops.md#probes-resources--staff-gotchas) — readiness vs liveness
    - [When to use what (cheat sheet)](./deployment-and-ops.md#when-to-use-what-cheat-sheet)
  - [Kubernetes vs Amazon ECS](./deployment-and-ops.md#kubernetes-vs-amazon-ecs)
  - [Deployment strategies](./deployment-and-ops.md#deployment-strategies)
  - [Monitoring: Prometheus, Grafana & ELK](./deployment-and-ops.md#monitoring-prometheus-grafana-elk)
    - [Prometheus](./deployment-and-ops.md#prometheus)
    - [Grafana](./deployment-and-ops.md#grafana)
    - [ELK stack](./deployment-and-ops.md#elk-stack)
    - [ELK vs Prometheus & Grafana](./deployment-and-ops.md#elk-vs-prometheus-grafana)
    - [Full observability trio](./deployment-and-ops.md#full-observability-trio)
  - [SLIs, SLOs, SLAs (ops vocabulary)](./deployment-and-ops.md#slis-slos-slas-ops-vocabulary)
  - [Other ops prerequisites](./deployment-and-ops.md#other-ops-prerequisites)

- **[Prometheus & Grafana setup (detailed)](./prometheus-grafana-setup.md)**
  - [End-to-end flow](./prometheus-grafana-setup.md#end-to-end-flow)
  - [What `/metrics` returns](./prometheus-grafana-setup.md#what-metrics-returns)
  - [Sample `/metrics` export](./prometheus-grafana-setup.md#sample-metrics-export)
  - [Backend vs frontend](./prometheus-grafana-setup.md#backend-vs-frontend)
  - [Without Kubernetes](./prometheus-grafana-setup.md#without-kubernetes)
  - [With Kubernetes](./prometheus-grafana-setup.md#with-kubernetes)
  - [Configuring Grafana](./prometheus-grafana-setup.md#configuring-grafana)
  - [Alertmanager (optional)](./prometheus-grafana-setup.md#alertmanager-optional)
  - [Other common tools](./prometheus-grafana-setup.md#other-common-tools)
  - [Interview vs ops depth](./prometheus-grafana-setup.md#interview-vs-ops-depth)

---

## Quick jump (mock-interview hot topics)

| Topic | Link |
|-------|------|
| Index still slow + EXPLAIN | [Why a query with an index can still be slow](./algorithms-and-indexes.md#why-a-query-with-an-index-can-still-be-slow) |
| Cache-aside vs write-through | [Caching patterns](./caching.md#quick-recap-of-patterns) |
| Cache stampede | [Stampede](./caching.md#cache-stampede-aka-dogpile-thundering-herd) |
| Idempotency / retried POST | [Idempotency](./core-concepts.md#idempotency) |
| At-least-once + payment consumer | [Inbox / dedupe](./messaging-and-pipelines.md#inbox-dedupe-table) |
| API contract / versioning | [API contracts](./service-architecture.md#api-contracts-other-teams-depend-on) |
| SLI / SLO / error budget | [SLI, SLO, SLA](./reliability-and-slos.md#sli-slo-sla) · [Error budgets](./reliability-and-slos.md#error-budgets) |
| K8s vs ECS | [Kubernetes vs Amazon ECS](./deployment-and-ops.md#kubernetes-vs-amazon-ecs) |
| Readiness probe | [Probes](./deployment-and-ops.md#probes-resources--staff-gotchas) |
| RAG / LLM off path | [AI systems](./ai-systems.md) |

---

_Every top-level doc in the curriculum appears above as its own bold entry (AI Systems is not buried under Data Stores). When you add a new `##` section to a doc, add it here and in that doc’s Index._
