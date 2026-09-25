# Messaging & Data Pipelines

How systems move data asynchronously: queues, streams, logs, and processors.

← [README](../README.md) · [Docs index](./README.md) · Related: [Building blocks](./building-blocks.md) · [Logging diagram](../diagrams/logging-and-monitoring-system/logging-and-monitoring-system.excalidraw)

---

## Index

- [Pub/Sub vs Message Queue](#pubsub-vs-message-queue)
- [Kafka vs RabbitMQ vs SQS](#kafka-vs-rabbitmq-vs-sqs)
- [Cluster metadata & coordination (ZooKeeper, KRaft, and alternatives)](#cluster-metadata-coordination-zookeeper-kraft-and-alternatives)
- [AWS Kinesis](#aws-kinesis)
- [Lambda vs Kappa architecture](#lambda-vs-kappa-architecture)
- [Write-ahead log (WAL) & MySQL binlog](#write-ahead-log-wal-mysql-binlog)
- [CDC (Change Data Capture)](#cdc-change-data-capture)
- [Event aggregator (Spark) vs Stream aggregator (Flink)](#event-aggregator-spark-vs-stream-aggregator-flink)
  - [Related patterns (also prerequisites)](#related-patterns-also-prerequisites)
    - [Transactional outbox](#transactional-outbox)
    - [Inbox / dedupe table](#inbox-dedupe-table)
    - [Event sourcing (light)](#event-sourcing-light)
    - [CQRS (light)](#cqrs-light)
    - [Log compaction (Kafka)](#log-compaction-kafka)

---

## Pub/Sub vs Message Queue

| | **Message queue** | **Pub/Sub** |
|--|-------------------|-------------|
| Delivery | Usually **one** consumer (competing workers) | **Many** subscribers independently |
| Goal | Distribute work | Broadcast events |
| Ack model | Consumer acks → message removed | Each subscription has its own progress |
| Example | “Resize this image” | “OrderPlaced → email, analytics, inventory” |

Hybrid products exist (SNS→SQS, Kafka consumer groups). In interviews, pick based on **work distribution** vs **fan-out of events**.

---

## Kafka vs RabbitMQ vs SQS

Same three jobs (work queue, pub/sub, stream), different parts. None delete a message just because a consumer **saw** it — it is finished only on ack / delete / offset commit. Retry is **at-least-once**, so consumers must be idempotent. Order is never global: one FIFO group, one queue, or one partition.

**Parts**

- **SQS** — you don’t run a broker. A **queue** is the only object (standard or FIFO). **MessageGroupId** is the FIFO order scope (like a key). **Visibility timeout** hides a received message until delete or timeout. Pub/sub is **not** SQS: put **SNS** in front and subscribe many queues.
- **RabbitMQ** — the **broker** is the node/cluster. Producer publishes to an **exchange** (fanout / direct / topic). A **binding** + **routing key** copies into one or more **queues**. Consumers compete on a queue; unacked messages stay until `ack`.
- **Kafka** — a **broker** is one server in the cluster (often **MSK**). A **topic** is a named log, split into **partitions** (ordered, append-only shards — the parallelism unit). A **key** is hashed to a partition: same key → same partition → order. A **consumer group** splits partitions among its members (one member per partition). Another group reads the same log with its own **offsets**.

| | **SQS** | **RabbitMQ** | **Kafka** |
|--|---------|--------------|-----------|
| Broker | AWS runs it; you only see the queue | RabbitMQ node/cluster (or Amazon MQ) | Server storing partition replicas; cluster or MSK |
| Stored where | One queue | Queue, after exchange + binding | Partition inside a topic |
| Key / route | FIFO: `MessageGroupId`. Standard: none | Routing key → binding → queue(s) | Record **key** → partition |
| Simple queue (one job, many workers) | Native. Receive → one worker; others don’t see it until timeout or delete | One queue, competing consumers, `prefetch` | Awkward. One consumer group; partitions = parallelism; message stays until retention |
| Pub/sub (many independent readers) | **SNS →** one SQS queue per subscriber | Fanout/topic exchange → one queue per subscriber | One topic, **one consumer group per subscriber** (own offsets) |
| Streaming / replay | No. Ack deletes it | No. Ack removes it | Yes. Retention + reset offset / new group from earliest |
| On read | Invisible for visibility timeout; still on the queue | Unacked (`prefetch`); still on the queue | Offset not committed; record stays on the log |
| Success | `DeleteMessage` | `basic.ack` | Commit offset **after** the side effect |
| Fail before ack | Timeout → visible again; receive count +1 | Channel close or `nack(requeue=true)` → redeliver | Don’t commit → same offset retried; **partition blocked** |
| Retry delay | Visibility timeout (or delay, max 15 min) | Immediate requeue unless you add TTL/delay queues | None. Pause on the offset, or app retry topic |
| DLQ | Not on until you set **redrive** (`maxReceiveCount` + DLQ). FIFO needs a FIFO DLQ | Not on until **`x-dead-letter-exchange`**. Quorum: also `x-delivery-limit` or messages requeue forever / drop | **Not native.** App writes `<topic>.DLT`. Without that, stuck offset or a skip (gap) |
| Produce order | FIFO only, per group, and only if you send the next after the previous succeeds. Standard: none | One queue, **publisher confirms**, next dependent only after confirm | Same key → one partition. `acks=all` + **idempotent producer** so retries don’t reorder |
| Produce fails | Failed send never enters. Later successes in that group stay after earlier ones (**gap**, no swap). Other groups fine. Broker does not know “B depends on A” — you stop sending | Unconfirmed message is not queued. Don’t publish the next in the chain. Other queues fine | Unacked record is not in the log. Idempotent producer retries the **same sequence**. Non-idempotent + in-flight &gt; 1 **can reorder** |
| Consume order | FIFO: failure **blocks** later messages in that group until ack or DLQ. Extra workers don’t order a standard queue | Holds only with **one** consumer. Competing consumers break order; requeue can jump the front | One consumer per partition. Stuck offset blocks the rest. Skip/DLT unblocks but leaves a gap |
| Ops / fit | Simplest. Work queue, AWS glue. Not a log | Medium ops. Routing, delay, priorities | Heavier (MSK still not free). CDC, replay, many readers |

**Example — `userId=42` events E1, E2, E3 (E2 produce fails).** Queue/group/key = `42`.

- **SQS FIFO:** E1 is in the group. E2 never is. If you waited for E1’s success before E3, E3 is not sent. If you sent E3 anyway, consumers see E1 then E3 (gap, not a swap).
- **RabbitMQ:** no confirm on E2 → don’t publish E3 to that queue. Other routing keys are untouched.
- **Kafka key `42`:** E1 is offset N on one partition. Idempotent producer retries E2 in place. E3 follows only after E2 is acked, so dependents don’t leapfrog.

**Example — worker crashes mid-payment.**

- **SQS:** invisible for 30s, then another receive (count 2). After `maxReceiveCount` (if redrive is set) it moves to the DLQ; otherwise it retries until retention.
- **RabbitMQ:** unacked message returns. Without a DLX it can poison-loop. With DLX + delivery-limit, the Nth failure is dead-lettered.
- **Kafka:** offset not committed, so that partition retries the same record and later payments for that key wait. App publishes to `.DLT` and commits only if you accept a gap.

**Example — OrderPlaced to email and analytics.**

- **SQS:** SNS topic, two SQS subscriptions; each queue is a simple competing-worker queue.
- **RabbitMQ:** fanout exchange, bindings to `email.q` and `analytics.q`.
- **Kafka:** topic `orders`, groups `email` and `analytics`; both replay independently.

Parallelize by **more groups/partitions/keys**, not by sharing one ordered stream across many workers.

See [Distributed Queue](../diagrams/distributed-queue/distributed-queue.excalidraw).

---

## Cluster metadata & coordination (ZooKeeper, KRaft, and alternatives)

Brokers need a **control plane**: who is in the cluster, who is **controller/leader**, where each partition/queue replica lives, and ISR / membership changes. That is **not** the data path (your `OrderPlaced` payloads).

### What ZooKeeper did for Kafka (classic)

[ZooKeeper](./distributed-coordination.md#zookeeper-etcd-consul-coordination-services) is a small **quorum coordination** service (ZAB consensus). Older Kafka clusters used it to store/elect:

| Job | Why it matters |
|-----|----------------|
| **Controller election** | One broker is controller: assigns leaders for partitions, reacts to broker fail/join |
| **Broker registration** | Live membership of the cluster |
| **Topic / partition metadata** | Which broker leads which partition; replica assignments |
| **ACL / config (historically)** | Cluster-wide config bits (much of this moved over time) |

**Not ZK’s job:** storing consumer messages or high-QPS offsets (offsets live in Kafka’s `__consumer_offsets` topic in modern setups).

**Pain:** you operated **two** systems (Kafka + ZK). ZK outage or mis-sizing blocked controller election even if disks with data were fine. Hence the move to KRaft.

```text
Classic:  Producers/Consumers → Kafka brokers ← metadata/election → ZooKeeper ensemble
KRaft:    Producers/Consumers → Kafka brokers (controllers use an internal metadata log / Raft)
```

### KRaft (Kafka without ZooKeeper)

**KRaft** = Kafka’s own **Raft**-based metadata quorum (controller voters). Topic/broker metadata lives in an internal log; no external ZK.

| | Classic (ZK) | KRaft |
|--|--------------|-------|
| Extra cluster | Yes — ZK ensemble | No |
| Controller | Elected via ZK | Controllers form a Raft quorum inside Kafka |
| Interview default today | “Legacy / still seen in older deploys” | “Modern Kafka / MSK paths moving here” |

**Interview line:** “ZK coordinated Kafka’s **control plane** (controller, membership, partition leaders). Data stayed on brokers. New Kafka uses **KRaft** so metadata is Raft inside Kafka — one less moving part.”

### Alternatives: who does this job in SQS and RabbitMQ?

Same *need* (membership, leaders, failover) — different *owner*.

| System | Coordination / metadata | What you operate |
|--------|-------------------------|------------------|
| **Kafka + ZK** | External ZK ensemble | Brokers **and** ZK |
| **Kafka + KRaft** | Internal Raft metadata quorum | Brokers only (controllers are Kafka nodes) |
| **Amazon SQS** | **AWS-owned** control plane — no ZK, no broker cluster for you | Queue APIs only (regions, quotas, redrive). Failover/sharding is Amazon’s problem |
| **Amazon MQ (managed Rabbit)** | AWS manages the broker/cluster control plane | You configure queues/exchanges; not ZK |
| **Self-managed RabbitMQ** | **Built-in** clustering (Erlang distribution + membership). **Quorum queues** use **Raft** inside Rabbit for replicated queue leadership — **not ZooKeeper** | Rabbit nodes only (optional peer-discovery plugin: DNS/K8s/etcd — discovery ≠ Kafka-style ZK metadata store) |
| **Kinesis** | AWS control plane + shard map | Managed; closest “Kafka-like” without ZK |

**RabbitMQ detail (interview-useful):**

- **Classic mirrored queues** — older HA; leader + mirrors, promotion on fail (deprecated path in favor of quorum).  
- **Quorum queues** — Raft consensus per queue; elect queue leader; durable under node loss.  
- You do **not** draw ZooKeeper in front of Rabbit unless some org-specific tool uses it for **service discovery** only.

**SQS detail:** there is no “controller election” in your diagram. Draw **queue + visibility timeout + optional DLQ**. Multi-AZ durability and partition placement are behind the API.

### HLD board cheat sheet

| If you draw… | Say… |
|--------------|------|
| Self-hosted Kafka (old) | Brokers + **ZK** (or note migration) |
| Modern Kafka / many MSK setups | Brokers + **KRaft** (no ZK box) |
| SQS | No coordination box |
| RabbitMQ | Cluster / **quorum queues (Raft)** — not ZK |

Related: [ZooKeeper / etcd / Consul](./distributed-coordination.md#zookeeper-etcd-consul-coordination-services) · [Raft](./distributed-coordination.md#raft-consensus).

---

## AWS Kinesis

Managed **streaming** on AWS (closest cousin to Kafka, not to SQS).

| Piece | Role |
|-------|------|
| **Data Streams** | Sharded append log; producers → shards → consumers (apps / Lambda / Firehose) |
| **Firehose** | Managed delivery to S3 / Redshift / OpenSearch (batch flush) |
| **Data Analytics** | SQL on streams (less common in interviews now) |

**vs Kafka:** same idea (shards ≈ partitions, retention, replay). Kinesis = less ops, AWS-native, shard scaling / throughput limits to plan; Kafka / MSK = more control, richer ecosystem (Connect, exact consumer-group patterns).

**vs SQS:** Kinesis = ordered stream + multiple consumers + replay; SQS = task queue, delete-on-ack, no real replay.

**Interview use:** logging/metrics ingest, clickstream, CDC fan-out on AWS → Flink/Spark/Lambda/S3.

---

## Lambda vs Kappa architecture

How you build analytics / derived views from events (Nathan Marz / Jay Kreps ideas).

| | **Lambda** | **Kappa** |
|--|------------|-----------|
| Idea | **Speed layer** (realtime) + **batch layer** (correct/rebuild) + serve | **One** streaming pipeline; recompute by **replaying the log** |
| Batch | Periodic Spark/MapReduce over full history | Optional; prefer replay on stream system |
| Complexity | Two code paths to keep in sync | One code path; needs strong log + replay |
| When | Heavy historical recompute, batch already exists | Log-centric (Kafka/Kinesis), stream processor can redo |

```text
Lambda:  events → stream speed path
              ↘ batch over lake → merge at serve

Kappa:   events → durable log → stream job (replay to fix / backfill)
```

**Pick Kappa** if Kafka/Kinesis + Flink/Spark Structured Streaming can own both realtime and recompute.  
**Pick Lambda** if you already have a lake/warehouse batch world and a separate low-latency path.

---

## Write-ahead log (WAL) & MySQL binlog

A **WAL** appends intended changes to a durable log **before** (or as part of) applying them to data files.

Why it matters:

- Crash recovery: replay log → consistent state
- Replication: replicas follow the leader’s log
- Foundation for CDC and streaming

**MySQL binlog** — logical/physical log of committed changes used for:

- Replica sync (async / semi-sync)
- Point-in-time recovery
- CDC tools (Debezium, Maxwell) reading binlog → Kafka

Postgres equivalent: **WAL** + logical decoding / `pgoutput`.

Interview line: “Source of truth for replication and CDC is the database log, not polling the tables.”

---

## CDC (Change Data Capture)

Stream **row-level changes** (insert/update/delete) from DB → downstream (search index, cache, warehouse, other services).

| Approach | Pros | Cons |
|----------|------|------|
| **Log-based** (binlog/WAL) | Low load, ordered, complete | Needs log access; schema evolution care |
| **Query-based / poll** | Simple | Lag, misses deletes unless soft-delete, load |
| **Trigger-based** | Explicit | Heavy on DB; hard to maintain |

Common pattern: **DB → Debezium → Kafka → consumers** (indexer, cache invalidator, analytics).

Use CDC when search/cache must stay near real-time without dual-writes from app code (dual-write is fragile). Prefer **transactional outbox** if you must emit events from the app safely.

---

## Event aggregator (Spark) vs Stream aggregator (Flink)

Both do large-scale data processing; the interview distinction is **batch vs continuous stream**.

| | **Spark (classic batch / micro-batch)** | **Flink (true streaming)** |
|--|----------------------------------------|----------------------------|
| Model | Jobs over bounded datasets (or micro-batches) | Continuous operators on unbounded streams |
| Latency | Seconds–minutes typical | Sub-second to seconds |
| State | Job-scoped; Spark Streaming improved over time | First-class keyed state + checkpoints |
| Windows | Supported (esp. Structured Streaming) | Rich event-time windows, watermarks |
| Typical use | ETL, hourly aggregates, ML feature batch, S3→warehouse | Real-time metrics, fraud, continuous joins, CEP |

**In logging HLD:** Kafka/Kinesis → **Flink/Logstash** for near-real-time index; **Spark** for periodic heavy aggregates from S3. See [Logging and Monitoring](../diagrams/logging-and-monitoring-system/logging-and-monitoring-system.excalidraw).

Also hear: **Spark Structured Streaming** (micro-batch) vs **Flink** (continuous) — don’t treat names as religion; talk **latency and state**.

---

## Related patterns (also prerequisites)

### Transactional outbox
Write business row + “event to publish” in the **same DB transaction**; a relay publishes to Kafka/SQS. Avoids dual-write loss.

### Inbox / dedupe table
Consumer stores `eventId` (or payment id) **before** side effects → idempotent under at-least-once delivery.

**Payment event consumer (interview answer)**

1. Delivery is **at-least-once** — duplicates happen.  
2. Begin tx / upsert: `INSERT INTO processed_events(event_id) …` with **unique** `event_id`.  
3. If duplicate key → **no-op** (or return prior result); do **not** charge/refund again.  
4. Else apply ledger / status transition; commit.  

That is **idempotent processing**, not “turn on exactly-once.” Keys on retried HTTP POSTs: [Idempotency](./core-concepts.md#idempotency).

### Event sourcing (light)
Store state as a sequence of events; rebuild via replay. Powerful but heavy — mention only when audit/replay is core.

### CQRS (light)
Separate write model from read model (e.g. SQL writes, Redis/ES reads updated via CDC).

### Log compaction (Kafka)
Keep latest value per key — useful for changelog / materialised state topics.
