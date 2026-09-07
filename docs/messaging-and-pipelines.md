# Messaging & Data Pipelines

How systems move data asynchronously: queues, streams, logs, and processors.

← [README](../README.md) · [Docs index](./README.md) · Related: [Building blocks](./building-blocks.md) · [Logging diagram](../diagrams/logging-and-monitoring-system/logging-and-monitoring-system.excalidraw)

---

## Index

- [Pub/Sub vs Message Queue](#pubsub-vs-message-queue)
- [Kafka vs RabbitMQ vs SQS](#kafka-vs-rabbitmq-vs-sqs)
- [AWS Kinesis](#aws-kinesis)
- [Lambda vs Kappa architecture](#lambda-vs-kappa-architecture)
- [Write-ahead log (WAL) & MySQL binlog](#write-ahead-log-wal-mysql-binlog)
- [CDC (Change Data Capture)](#cdc-change-data-capture)
- [Event aggregator (Spark) vs Stream aggregator (Flink)](#event-aggregator-spark-vs-stream-aggregator-flink)
- [Related patterns (also prerequisites)](#related-patterns-also-prerequisites)

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

| | **Amazon SQS** | **RabbitMQ** | **Kafka** |
|--|----------------|--------------|-----------|
| Model | Managed queue | Smart broker (AMQP) | Distributed append log |
| Ordering | Standard: none; FIFO: per group (limited) | Per-queue; routing flexible | **Per partition** by key |
| Replay | No (after delete/ack) | Generally no | **Yes** (retain + offset reset) |
| Throughput | High, simple | Medium; rich routing | Very high, sequential disk |
| Ops | Almost none | You run clusters (or cloud) | Heavier ops / MSK |
| Best for | Simple async jobs, decouple AWS services | Complex routing, per-message control | Event streams, CDC, analytics, replay |

**Rule of thumb**

- **SQS** — simple **work queue** (not pub/sub): one message → one worker; similar jobs; AWS-native; + DLQ. For pub/sub use **SNS→SQS**.
- **RabbitMQ** — complex routing (exchanges/keys), priorities, delay/retry patterns; classic task messaging.
- **Kafka** — high volume, **per-key order**, **replay**, **many consumer groups** on same log (CDC, analytics, event bus).

**Cost / ops (rough):** SQS simplest (fully managed). RabbitMQ medium (Amazon MQ or self-host). Kafka highest complexity/cost at scale — but **MSK / Confluent** means it’s not always “self-managed.” Pick on **replay / fan-out / routing needs**, not cost alone.

See [Distributed Queue](../diagrams/distributed-queue/distributed-queue.excalidraw).

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
Consumer stores `eventId` before side effects → idempotent under at-least-once delivery.

### Event sourcing (light)
Store state as a sequence of events; rebuild via replay. Powerful but heavy — mention only when audit/replay is core.

### CQRS (light)
Separate write model from read model (e.g. SQL writes, Redis/ES reads updated via CDC).

### Log compaction (Kafka)
Keep latest value per key — useful for changelog / materialised state topics.
