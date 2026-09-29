# Data Stores

Choosing where data lives: transactional DBs, analytics, time series, and AI retrieval.

← [README](../README.md) · [Docs index](./README.md) · Related: [Building blocks](./building-blocks.md) · [Algorithms & indexes](./algorithms-and-indexes.md)

---

## Index

- [ACID vs BASE](#acid-vs-base)
- [Isolation levels (anomalies)](#isolation-levels-anomalies)
- [MVCC & long-running transactions](#mvcc--long-running-transactions)
- [SQL vs NoSQL](#sql-vs-nosql)
- [OLTP vs OLAP](#oltp-vs-olap)
- [TiDB (distributed SQL) vs TSDB (time-series DB)](#tidb-distributed-sql-vs-tsdb-time-series-db)
- [Downsampling](#downsampling)
- [Knowledge base vs Vector DB](#knowledge-base-vs-vector-db)
- [AI systems (RAG / LLM)](./ai-systems.md) — prod failures, eval, off request path
- [LSM trees (storage engine)](#lsm-trees-storage-engine)
- [DB deployment strategies](#db-deployment-strategies)
- [Online schema change / large-table ALTER](#online-schema-change--large-table-alter)
- [Quick chooser](#quick-chooser)

---

## ACID vs BASE

| | **ACID** (classic RDBMS) | **BASE** (many distributed NoSQL designs) |
|--|--------------------------|---------------------------------------------|
| **A**tomicity | All-or-nothing transaction | Soft state / eventual outcomes |
| **C**onsistency | Constraints hold after commit | **C**onsistency is eventual |
| **I**solation | Concurrent tx don’t stomp each other | — |
| **D**urability | Committed data survives crash | — |
| **BA** | — | **B**asically **A**vailable |
| **S** | — | **S**oft state (may change without write) |
| **E** | — | **E**ventually consistent |

**Interview use**

- Payments, bookings, ledger → lean **ACID** (or carefully designed equivalents)
- Feeds, session cache, presence, analytics ingest → **BASE**-style often OK
- Real systems mix both: ACID for money path, eventual for read models

BASE is a slogan, not a product checkbox — say *which* invariant is relaxed.

---

## Isolation levels (anomalies)

**Interview snapshot**
- **What:** How much one transaction can see of concurrent others’ writes.
- **Why:** Wrong level → dirty/non-repeatable/phantom bugs or needless lock pain.
- **Trade-off:** Stronger isolation = fewer anomalies, more blocking / aborts.
- **Example:** Inventory check under READ COMMITTED can see a seat free that another txn already booked (phantom / non-repeatable depending on DB).

| Anomaly | Meaning |
|---------|---------|
| **Dirty read** | Read a write that later **rolls back** |
| **Non-repeatable read** | Re-read same row → different committed value |
| **Phantom read** | Re-run same range query → new/missing rows |
| **Lost update** | Two read-modify-write overwrite each other |

| Level (SQL) | Dirty | Non-repeatable | Phantom | Notes |
|-------------|-------|----------------|---------|-------|
| **READ UNCOMMITTED** | possible | possible | possible | Rarely used |
| **READ COMMITTED** | prevented | possible | possible | Postgres default; each statement sees latest committed |
| **REPEATABLE READ** | prevented | prevented | DB-dependent* | Snapshot of first read; *Postgres prevents phantoms via SSI-ish snapshots; MySQL RR uses gap locks |
| **SERIALIZABLE** | prevented | prevented | prevented | Looks like serial execution; more aborts/retries |

**Interview line:** “Name the anomaly you fear (dirty / non-repeatable / phantom / lost update), then pick the weakest level that prevents it — don’t default to SERIALIZABLE.”

---

## MVCC & long-running transactions

**Interview snapshot**
- **What:** MVCC keeps row versions so readers/writers don’t block; a long open txn pins an old snapshot.
- **Why:** That pin blocks vacuum → bloat → everyone else’s queries get slower.
- **Trade-off:** Short transactions keep the system healthy; very long “one big txn” feels simpler in app code but taxes the fleet.
- **Example:** A report txn open for 2 hours → dead tuples pile up → OLTP p99 climbs until it commits.

**MVCC (Multi-Version Concurrency Control):** readers don’t block writers (and usually vice versa). Each row version keeps enough info for a transaction to see a **consistent snapshot**. Postgres-style: updates/deletes leave old versions until **VACUUM** can reclaim them.

### What a long-running transaction does to the rest of the system

| Effect | Why |
|--------|-----|
| **Bloat** | Old row versions stay visible to that snapshot → table/index grow; more IO |
| **Slow queries for everyone** | Seq/index scans touch more dead tuples; cache less effective |
| **VACUUM / autovacuum stalls** | Can’t remove versions still needed by the open txn (or older snapshots) |
| **Replication / slots lag** (logical) | Slot holds WAL until consumer catches up — disk fills |
| **Lock / idle-in-transaction** | Holds row/table locks or connection pool slots; others wait |

**Interview line:** “Long open transactions pin an old snapshot under MVCC → dead tuples pile up → everyone pays with bloat and slower scans until the txn ends and vacuum can run.”

**Mitigations:** short transactions; no interactive pauses mid-txn; statement/idle timeouts; watch `idle_in_transaction`; batch work in small commits; monitor bloat / vacuum lag.

---

## SQL vs NoSQL

| | **SQL / relational** | **NoSQL** (document, KV, wide-column, graph, …) |
|--|----------------------|--------------------------------------------------|
| Schema | Tables, rigid or evolving with migrations | Flexible documents / sparse columns |
| Query | Rich joins, ad-hoc SQL | Access by key / partition; joins limited or app-side |
| Transactions | Mature multi-row ACID | Often single-key or limited multi-key |
| Scale out | Replicas first; sharding harder | Designed for horizontal partition early |
| Examples | Postgres, MySQL, SQL Server | MongoDB, DynamoDB, Cassandra, Redis, Neo4j |

**Pick SQL when:** complex queries, strong invariants, multi-entity transactions.  
**Pick NoSQL when:** known access patterns, massive scale, flexible attrs, simple key lookups.

**Nuance:** “NewSQL” / distributed SQL (CockroachDB, **TiDB**, Spanner) aims for SQL + horizontal scale — see below.

Don’t choose NoSQL only because it’s trendy; choose it for an access pattern.

---

## OLTP vs OLAP

| | **OLTP** | **OLAP** |
|--|----------|----------|
| Goal | Run the product (orders, clicks, chats) | Analyze the business (aggregates, trends) |
| Workload | Many small read/write tx | Large scans, group-bys, joins |
| Latency | ms | seconds–minutes (or interactive BI) |
| Model | Normalized or lightly denormalized | Star/snowflake, columnar, wide events |
| Stores | Postgres, MySQL, DynamoDB | ClickHouse, BigQuery, Snowflake, Redshift, Druid |

**Pattern:** OLTP DB → CDC / ETL → **OLAP warehouse or columnar store**. Don’t run heavy analytics on the primary OLTP box.

---

## TiDB (distributed SQL) vs TSDB (time-series DB)

These acronyms get confused in conversation — different jobs.

### TiDB

- **Distributed SQL** (MySQL-compatible), inspired by Google Spanner + HTAP goals
- Horizontal scale with SQL surface; TiKV storage, optional TiFlash for analytical replicas
- **Use when:** you outgrow single MySQL but want SQL + transactions, not a full rewrite to NoSQL

Related family: CockroachDB, YugabyteDB, Spanner, Vitess (sharding MySQL).

### TSDB (Time-Series Database)

- Optimized for **metrics / events over time**: `(metric, tags, timestamp) → value`
- High ingest, downsampling, retention policies, compression
- Examples: Prometheus TSDB, InfluxDB, TimescaleDB, OpenTSDB, VictoriaDB

**Use when:** monitoring, IoT, trading ticks, usage meters — not as your general user-profile store.

---

## Downsampling

Reduce resolution of time-series (or metrics) as data ages to save storage and speed charts.

Example:

- Last 24h: raw 15s samples  
- Last 7d: 1m averages  
- Last 1y: 1h averages / percentiles

**How:** rollups (avg, min, max, p99), continuous aggregates (Timescale), recording rules (Prometheus), stream jobs (Flink/Spark).

**Trade-off:** lose fine detail historically; keep what SLOs need (often max/p99, not only avg).

Pairs with TSDB retention: hot raw → warm downsampled → cold object storage / drop.

---

## Knowledge base vs Vector DB

Both show up in “AI search / RAG” designs; they solve different layers.

| | **Knowledge base (KB)** | **Vector DB** |
|--|-------------------------|---------------|
| What it is | Curated corpus of documents + metadata (+ often search UI/ACL) | Index of **embeddings** for similarity search |
| Primary query | Keyword / structured / browse by doc id | Approximate nearest neighbor (ANN): “semantically close to this vector” |
| Good at | Source of truth, permissions, citations, versioning | Fuzzy semantic retrieval (“login broken on mobile”) |
| Examples | Confluence, Notion, wiki, S3+metadata, Elastic docs | Pinecone, Weaviate, Milvus, Qdrant, pgvector |

**Typical RAG HLD**

```text
Docs (KB) → chunk + embed → Vector DB
User query → embed → ANN top-k → (optional rerank / keyword filter) → LLM → answer + citations
```

Often **hybrid**: keyword (BM25/ES) + vector, with ACL filters from the KB.  
Vector DB is not a replacement for the KB — it’s a **retrieval index** over content that still lives somewhere authoritative.

**Prod failures, retrieval eval, LLM off the request path:** [AI systems](./ai-systems.md).

Full search comparison (keyword inverted index vs semantic vs proximity): [Algorithms & indexes](./algorithms-and-indexes.md#choosing-proximity-vs-keyword-vs-semantic).

---

## LSM trees (storage engine)

**Log-Structured Merge-tree:** buffer writes in memory (memtable) → flush sorted **SSTables** to disk → background **compaction** merges levels.

| Pros | Cons |
|------|------|
| Excellent write throughput | Read amplification (may check several levels) |
| Sequential disk writes | Compaction CPU/IO spikes |
| Fits SSD / high ingest | Tombstones until compacted |

Used inside: Cassandra, RocksDB, LevelDB, Bigtable-style systems, many TSDB/OLAP pieces.

Contrast **B-tree** (inplace, InnoDB): better point/range reads in place; random write amplification on disk.

**Interview line:** write-heavy wide-column / TSDB → LSM; classic OLTP relational → B-tree (often).

More index types: [Algorithms & indexes](./algorithms-and-indexes.md).

---

## DB deployment strategies

### Single region vs multi-region

| | **Single region (multi-AZ)** | **Multi-region** |
|--|----------------------------|------------------|
| Protects | AZ / box failure | Region disaster; closer reads for global users |
| Latency | Low inside region | Cross-region RTT on sync paths |
| Complexity | Baseline for prod | Replication lag, conflict, failover runbooks |
| Default interview | Start here | Only if NFR needs global latency or low RPO/RTO across regions |

### Replication in multi-region

| Mode | Behavior | Trade-off |
|------|----------|-----------|
| **Async primary → replicas** | Fast writes locally; replicas lag | Possible data loss on primary region loss (RPO &gt; 0) |
| **Semi-sync / sync** | Wait for remote ack | Higher write latency; stronger durability |
| **Active-passive** | One writer region; others read / standby | Simpler failover |
| **Active-active** | Writes in multiple regions | Conflicts — need keys-by-region, CRDTs, or conflict rules |
| **Read replicas regional** | Local reads, central writes | Stale reads; great for read-heavy global apps |

State **RPO/RTO** when you draw multi-region DB. Detail: [Reliability — multi-region](./reliability-and-slos.md#multi-az-vs-multi-region).

### Read-heavy vs write-heavy

| Workload | Deploy moves |
|----------|----------------|
| **Read-heavy** | Cache → **read replicas** (same or other regions) → CDN for public GETs → consider CQRS/read models |
| **Write-heavy** | Batch/async where possible → partition/shard on write key → LSM/wide-column if fit → avoid sync cross-region on every write |
| **Read + write both hot** | Split paths: sync write to primary; async CDC to read stores; don’t force one DB shape for both |

**Connection note:** every replica and every service/worker pool multiplies connections — use a pooler; see [Service architecture — DB topology & connections](./service-architecture.md#db-topology-connections).

---

## Online schema change / large-table ALTER

**Interview snapshot**
- **What:** Evolve schema on a big table without long exclusive locks / downtime.
- **Why:** Naive `ALTER TABLE` on millions of rows can lock writes for minutes–hours.
- **Trade-off:** Online tools (pt-osc, gh-ost, native online DDL) are safer but slower and need dual-write/cutover discipline.
- **Example:** Add nullable column with default via online DDL → backfill in batches → switch app → drop old path.

**Naive ALTER risk:** table rewrite / exclusive lock → writers queue → outage.

**Safer patterns**

| Pattern | Idea |
|---------|------|
| **Expand–contract** | Add nullable column / new table → deploy dual-write → backfill → switch reads → drop old |
| **Online DDL tools** | `gh-ost` / `pt-online-schema-change` / cloud provider online DDL — copy + trigger/binlog, cutover |
| **Batch backfill** | Chunked UPDATEs with sleep; watch replication lag |
| **New table + swap** | Build shadow table; rename atomically at cutover |

**Don’t:** run heavy `ALTER` in peak traffic without a rollback plan; don’t assume “Postgres is MVCC so ALTER is free” — still verify lock mode (`ACCESS EXCLUSIVE` vs weaker).

**Interview line:** “Large-table change = expand–contract or online DDL + lag-aware backfill, not a blocking ALTER in prod.”

---

## Quick chooser

| Need | Lean toward |
|------|-------------|
| Money / booking invariants | ACID SQL |
| Huge key-value / timeline writes | Wide-column / LSM NoSQL |
| Dashboards & funnels | OLAP / warehouse |
| Metrics & downsampling | TSDB |
| Outgrew MySQL, keep SQL | TiDB / distributed SQL |
| Semantic AI retrieval | Vector DB + KB source of truth |
