# Topic catalog (sneak peek)

Crisp index of topics under `docs/` — **doc → `##` sections only**. Subsections live in each doc’s own Index. Study order: [Docs index](./README.md).

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

- **[Soft Skills](./soft-skills.md)** — conversation, trade-offs, Staff bar, language
  - [Drive the conversation](./soft-skills.md#drive-the-conversation) · [Assumptions](./soft-skills.md#state-assumptions-explicitly) · [Bottleneck first](./soft-skills.md#prefer-simple-then-scale-the-bottleneck) · [Compare & commit](./soft-skills.md#compare-two-options-then-commit) · [Board](./soft-skills.md#use-the-board-clearly) · [Pushback](./soft-skills.md#handle-pushback-well) · [Time](./soft-skills.md#time-management) · [Staff bar](./soft-skills.md#sde3-staff-communication-bar) · [Language](./soft-skills.md#language-that-scores) · [Practice drill](./soft-skills.md#practice-drill)
- **[Staff vocabulary](./staff-vocabulary.md)** — upstream/downstream, capacity, overload, isolation
  - [Upstream/downstream](./staff-vocabulary.md#upstream-vs-downstream-in-your-hld) · [Dependency chain](./staff-vocabulary.md#dependency-chain-on-the-board) · [Traffic & capacity](./staff-vocabulary.md#traffic-capacity) · [Latency & overload](./staff-vocabulary.md#latency-overload) · [Isolation](./staff-vocabulary.md#isolation-failure-spread) · [Queues](./staff-vocabulary.md#queues-async-paths)
- **[Estimation fluency](./estimation-fluency.md)** — DAU→QPS, storage, cache, capacity
  - [Constants](./estimation-fluency.md#useful-constants-memorize-these) · [Traffic](./estimation-fluency.md#traffic-dau-qps) · [Storage](./estimation-fluency.md#storage-growth) · [Bandwidth](./estimation-fluency.md#bandwidth) · [Cache](./estimation-fluency.md#cache-sizing) · [Servers](./estimation-fluency.md#servers-capacity-rough) · [Queues](./estimation-fluency.md#time-to-fill-drain-queues) · [Examples](./estimation-fluency.md#worked-mini-examples)
- **[Prep plan](./prep-plan.md)** — daily/weekly DSA · LLD · HLD schedule

## Foundation

- **[Building blocks](./building-blocks.md)**
  - [DNS](./building-blocks.md#dns) · [CDN](./building-blocks.md#cdn-content-delivery-network) · [LB](./building-blocks.md#load-balancer-lb) · [Forward vs reverse proxy](./building-blocks.md#forward-vs-reverse-proxy) · [VPN vs forward proxy](./building-blocks.md#vpn-vs-forward-proxy) · [Proxy products](./building-blocks.md#common-proxy-edge-products) · [API gateway](./building-blocks.md#api-gateway) · [Stateless apps](./building-blocks.md#stateless-app-servers) · [Caching](./building-blocks.md#caching-redis-memcached) · [Databases](./building-blocks.md#databases) · [Object storage](./building-blocks.md#object-storage-s3-style) · [Queues & streams](./building-blocks.md#message-queues-streams) · [Search](./building-blocks.md#search-engines) · [Realtime](./building-blocks.md#real-time-delivery) · [Observability](./building-blocks.md#observability-stack)
- **[Service architecture](./service-architecture.md)**
  - [Monolith vs microservices](./service-architecture.md#monolith-vs-microservices) · [Services vs workers](./service-architecture.md#services-vs-workers) · [When to use which](./service-architecture.md#where-to-use-services-vs-workers) · [Beyond services/workers](./service-architecture.md#beyond-services-workers) · [Cron vs Temporal](./service-architecture.md#cron-scheduler-vs-temporal) · [DB topology & connections](./service-architecture.md#db-topology-connections)
- **[Core concepts](./core-concepts.md)**
  - [CAP / PACELC](./core-concepts.md#cap-and-pacelc-practical-view) · [Consistency](./core-concepts.md#consistency-models) · [ACID vs BASE](./core-concepts.md#acid-vs-base) · [CRDT](./core-concepts.md#crdt-conflict-free-replicated-data-type) · [Idempotency](./core-concepts.md#idempotency) · [Optimistic locking](./core-concepts.md#optimistic-locking-versioning) · [Latency vs throughput](./core-concepts.md#latency-vs-throughput) · [Metrics vocabulary](./core-concepts.md#latency-metrics-vocabulary) · [Availability](./core-concepts.md#availability-failure-modes) · [Hot keys](./core-concepts.md#partitioning-hot-keys) · [Rate limiting](./core-concepts.md#rate-limiting) · [Backpressure](./core-concepts.md#backpressure) · [Exponential backoff](./core-concepts.md#exponential-backoff) · [Circuit breaker](./core-concepts.md#circuit-breaker) · [Security basics](./core-concepts.md#security-basics-hld-depth) · [Fan-out](./core-concepts.md#fan-out-on-write-vs-read) · [Sync vs async](./core-concepts.md#sync-vs-async)
- **[Scaling](./scaling.md)**
  - [What you’re scaling](./scaling.md#horizontal-scaling-what-are-you-scaling-against) · [Partition vs shard](./scaling.md#partitioning-vs-sharding) · [Consistent hashing](./scaling.md#consistent-hashing) · [Replicas vs sharding](./scaling.md#read-replicas-vs-sharding) · [Stateless vs stateful](./scaling.md#stateless-vs-stateful-horizontal-scale)
- **[Networking & media](./networking-and-media.md)**
  - [TCP vs UDP](./networking-and-media.md#tcp-vs-udp) · [HLS vs DASH](./networking-and-media.md#hls-vs-dash) · [RTMP vs SRT](./networking-and-media.md#rtmp-vs-srt)

## Data & messaging

- **[Data stores](./data-stores.md)**
  - [ACID vs BASE](./data-stores.md#acid-vs-base) · [SQL vs NoSQL](./data-stores.md#sql-vs-nosql) · [OLTP vs OLAP](./data-stores.md#oltp-vs-olap) · [TiDB vs TSDB](./data-stores.md#tidb-distributed-sql-vs-tsdb-time-series-db) · [Downsampling](./data-stores.md#downsampling) · [KB vs Vector DB](./data-stores.md#knowledge-base-vs-vector-db) · [LSM](./data-stores.md#lsm-trees-storage-engine) · [DB deploy](./data-stores.md#db-deployment-strategies)
- **[Caching](./caching.md)**
  - [Patterns](./caching.md#quick-recap-of-patterns) · [Stampede](./caching.md#cache-stampede-aka-dogpile-thundering-herd) · [Avalanche](./caching.md#cache-avalanche) · [Penetration](./caching.md#cache-penetration) · [Invalidation](./caching.md#cache-invalidation) · [Eviction](./caching.md#eviction-policies)
- **[Messaging & pipelines](./messaging-and-pipelines.md)**
  - [Pub/Sub vs queue](./messaging-and-pipelines.md#pubsub-vs-message-queue) · [Kafka / Rabbit / SQS](./messaging-and-pipelines.md#kafka-vs-rabbitmq-vs-sqs) · [ZK → KRaft & alternatives](./messaging-and-pipelines.md#cluster-metadata-coordination-zookeeper-kraft-and-alternatives) · [Kinesis](./messaging-and-pipelines.md#aws-kinesis) · [Lambda vs Kappa](./messaging-and-pipelines.md#lambda-vs-kappa-architecture) · [WAL / binlog](./messaging-and-pipelines.md#write-ahead-log-wal-mysql-binlog) · [CDC](./messaging-and-pipelines.md#cdc-change-data-capture) · [Spark vs Flink](./messaging-and-pipelines.md#event-aggregator-spark-vs-stream-aggregator-flink) · [Outbox / inbox / CQRS](./messaging-and-pipelines.md#related-patterns-also-prerequisites)
- **[Algorithms & indexes](./algorithms-and-indexes.md)**
  - [Bloom](./algorithms-and-indexes.md#bloom-filters) · [Hash vs encrypt](./algorithms-and-indexes.md#hashing-vs-encryption) · [Geo indexes](./algorithms-and-indexes.md#geo-spatial-indexes) · [Proximity](./algorithms-and-indexes.md#proximity-search-nearby) · [Keyword / ES](./algorithms-and-indexes.md#keyword-search-inverted-indexes-elasticsearch) · [Semantic](./algorithms-and-indexes.md#semantic-search) · [Chooser](./algorithms-and-indexes.md#choosing-proximity-vs-keyword-vs-semantic)

## Reliability & correctness

- **[Distributed coordination](./distributed-coordination.md)**
  - [Races](./distributed-coordination.md#race-conditions) · [Distributed tx](./distributed-coordination.md#distributed-transactions) · [2PC / 3PC](./distributed-coordination.md#two-phase-commit-2pc-3pc) · [Saga](./distributed-coordination.md#saga) · [Raft](./distributed-coordination.md#raft-consensus) · [Leader dies mid-tx](./distributed-coordination.md#leader-coordinator-dies-mid-transaction) · [ZK / etcd / Consul](./distributed-coordination.md#zookeeper-etcd-consul-coordination-services)
- **[Reliability & SLOs](./reliability-and-slos.md)**
  - [SLI/SLO/SLA](./reliability-and-slos.md#sli-slo-sla) · [Error budgets](./reliability-and-slos.md#error-budgets) · [Failure modes](./reliability-and-slos.md#failure-modes-checklist) · [Blast radius](./reliability-and-slos.md#blast-radius) · [Load shedding](./reliability-and-slos.md#load-shedding-graceful-degradation) · [Multi-AZ / region](./reliability-and-slos.md#multi-az-vs-multi-region) · [Backoff](./reliability-and-slos.md#exponential-backoff) · [Backpressure](./reliability-and-slos.md#backpressure) · [Fintech](./reliability-and-slos.md#reliability-in-fintech-razorpay-class)
- **[Security & compliance](./security-and-compliance.md)**
  - [Authn/z](./security-and-compliance.md#authn-vs-authz) · [Tokens](./security-and-compliance.md#tokens-sessions) · [TLS / data](./security-and-compliance.md#transport-data-protection) · [Secrets](./security-and-compliance.md#secrets-least-privilege) · [PCI](./security-and-compliance.md#pci-dss-mindset-payments-interviews) · [STRIDE](./security-and-compliance.md#threat-modeling-lightweight-stride) · [Abuse](./security-and-compliance.md#abuse-fraud-controls) · [Privacy](./security-and-compliance.md#privacy)

## Delivery & operations

- **[Deployment & ops](./deployment-and-ops.md)**
  - [Docker](./deployment-and-ops.md#docker) · [K8s](./deployment-and-ops.md#kubernetes-k8s) · [Deploy strategies](./deployment-and-ops.md#deployment-strategies) · [Prometheus / Grafana](./deployment-and-ops.md#monitoring-prometheus-grafana) · [SLI/SLO ops](./deployment-and-ops.md#slis-slos-slas-ops-vocabulary)

---

_Links target each doc’s `##` headings. Nested detail stays inside the doc._
