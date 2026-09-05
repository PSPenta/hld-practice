# Data Stores

Choosing where data lives: transactional DBs, analytics, time series, and AI retrieval.

← [README](../README.md) · [Docs index](./README.md) · Related: [Building blocks](./building-blocks.md) · [Algorithms & indexes](./algorithms-and-indexes.md)

---

## Index

- [ACID vs BASE](#acid-vs-base)
- [SQL vs NoSQL](#sql-vs-nosql)
- [OLTP vs OLAP](#oltp-vs-olap)
- [TiDB (distributed SQL) vs TSDB (time-series DB)](#tidb-distributed-sql-vs-tsdb-time-series-db)
- [Downsampling](#downsampling)
- [Knowledge base vs Vector DB](#knowledge-base-vs-vector-db)
- [LSM trees (storage engine)](#lsm-trees-storage-engine)
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

## Quick chooser

| Need | Lean toward |
|------|-------------|
| Money / booking invariants | ACID SQL |
| Huge key-value / timeline writes | Wide-column / LSM NoSQL |
| Dashboards & funnels | OLAP / warehouse |
| Metrics & downsampling | TSDB |
| Outgrew MySQL, keep SQL | TiDB / distributed SQL |
| Semantic AI retrieval | Vector DB + KB source of truth |
