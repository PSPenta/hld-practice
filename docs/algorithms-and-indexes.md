# Algorithms, Indexes & Crypto Basics

Probabilistic structures, geo indexes, **proximity / keyword / semantic search**, and hashing vs encryption.

← [README](../README.md) · [Docs index](./README.md) · Related: [Data stores — Vector DB](./data-stores.md#knowledge-base-vs-vector-db) · [Large Scale Search diagram](../diagrams/large-scale-search-system/large-scale-search-system.excalidraw)

---

## Index

- [Bloom filters](#bloom-filters)
- [Hashing vs encryption](#hashing-vs-encryption)
- [Geo-spatial indexes](#geo-spatial-indexes)
- [Proximity search (“nearby”)](#proximity-search-nearby)
- [Keyword search & inverted indexes (Elasticsearch)](#keyword-search-inverted-indexes-elasticsearch)
- [Semantic search](#semantic-search)
- [Choosing proximity vs keyword vs semantic](#choosing-proximity-vs-keyword-vs-semantic)
- [Other index / structure prerequisites](#other-index-structure-prerequisites)

---

## Bloom filters

**Probabilistic set:** “Is element *possibly* in the set?”

| Answer | Meaning |
|--------|---------|
| **No** | Definitely not present (no false negatives) |
| **Yes** | Maybe present (false positives possible) |

**Use cases**

- Cache penetration defense — skip DB if Bloom says no
- DB / LSM — avoid disk lookups for missing keys (e.g. RocksDB)
- Sync systems — quick “do I need this block?”

**Trade-offs:** tiny memory vs tunable false-positive rate; hard deletes need **counting Bloom** or rebuild; not a substitute for auth.

---

## Hashing vs encryption

| | **Hashing** | **Encryption** |
|--|-------------|----------------|
| Goal | Fingerprint / one-way integrity | Confidentiality (hide data) |
| Reversible? | No (by design) | Yes, with key |
| Key? | Optional (HMAC uses key for authenticity) | Required |
| Example | SHA-256 password store (with salt/pepper), checksums, shard keys | TLS, AES for fields at rest |
| Interview misuse | “We hash the PII so it’s secure to log” — often still sensitive | “We encrypt the password” — prefer salted hash for passwords |

**Related**

- **HMAC** — integrity + authenticity of messages
- **Salting** — per-password random to stop rainbow tables
- **Consistent hashing** — placement on a ring (not crypto) — see [Scaling](./scaling.md)
- **Hash for shard key** — distribution, not security

---

## Geo-spatial indexes

Building blocks for location encoding — used heavily by **proximity search** (next section).

### Geohash

- Encode lat/long into a **base32 string**; shared prefix → nearby (usually)
- Easy to index in Redis (`GEO*`) or as string/B-tree prefix
- **Pros:** simple, sortable-ish  
- **Cons:** edge cells that are close in space but distant in hash; precision vs string length

### H3 (Uber)

- Hexagonal hierarchical cells on a globe grid
- Stable cell IDs; k-ring neighbors for “nearby”
- **Pros:** nicer neighbor math than geohash squares; multi-resolution  
- **Cons:** library dependency; still approximate — refine with exact distance

### PostGIS (and geo types in SQL)

- Real GIS types & functions in Postgres (`GEOGRAPHY`/`GEOMETRY`)
- Indexes: **GiST / SP-GiST / BRIN** as appropriate
- **Pros:** exact queries, rich predicates, joins  
- **Cons:** heavier ops; scale with read replicas / careful indexing; not always for ultra-hot driver location fan-in (often Redis geo + DB for durable)

### Quadtree / R-tree (concepts)

- **Quadtree** — recursively split space into 4; good mental model for “points in bounding box”
- **R-tree / GiST** — what many DB geo indexes use under the hood for rectangles/points

---

## Proximity search (“nearby”)

**Problem:** given a location (and optional filters), return the closest / in-radius entities — restaurants, drivers, ATMs, stores.

Common prompts: *Yelp nearby*, *Uber matching*, *Find friends nearby*.

### Requirements to clarify

| Ask | Why it changes the design |
|-----|---------------------------|
| Static POIs vs moving objects? | Places ≈ infrequent updates; drivers ≈ high write QPS |
| Radius vs “top-K nearest”? | Bounding box / cells vs expand rings until K |
| Freshness of location? | TTL, last-seen, stale drivers |
| Filters? | Open now, cuisine, rating, availability |
| Scale | City vs global; QPS of search vs location updates |

### API sketch

```text
GET /nearby?lat=&lng=&radius_m=2000&type=restaurant&limit=20
GET /drivers/nearby?lat=&lng=&limit=10   // or internal match RPC
POST /locations  { entityId, lat, lng, timestamp }
```

### Core algorithm (interview flow)

1. **Index** each entity by cell (geohash / H3) or Redis GEO  
2. On query, compute **center cell + neighbors** (k-ring) covering the radius  
3. Fetch **candidates** in those cells (optionally pre-filtered by type)  
4. **Exact distance** (haversine / geodesic) → sort → take top-K  
5. Apply business filters; paginate if needed  

Never return raw cell results without an exact distance pass (edge errors).

### Hot path vs durable store

| Tier | Role |
|------|------|
| **Redis GEO / memory grid (H3 sets)** | Live locations, low-latency nearby |
| **Postgres + PostGIS / Elastic geo** | POI catalog, rich filters, analytics |
| **Object / CDN** | Map tiles (not the search index) |

**Uber-like live matching**

1. Drivers ping location → write Redis (and optionally async durable log)  
2. Rider request → nearby candidates in cells → rank (distance, ETA, rating) → offer  
3. Persist trip / history in OLTP DB  

**Yelp-like POI search**

1. Index places in Elastic (`geo_point`) or PostGIS  
2. `geo_distance` / bounding box + text filters + ranking  
3. Cache popular “downtown + pizza” style queries carefully (key = geohash prefix + filters)

### Deep-dive topics (Staff)

- **Cell size vs radius** — too coarse → huge candidate sets; too fine → many neighbor cells  
- **Edge cases** — geohash borders; always include neighbors  
- **Moving objects** — write amplification; shard Redis by region/city  
- **Load at city center** — hot geohash; shard or local denser resolution  
- **Privacy** — fuzz location; don’t leak precise coords of users  
- **Consistency** — “driver shown” may have moved; re-check at match time  

### Vs full-text / vector search

Geo answers **where**; keyword/semantic answer **what**. Full write-ups: [Keyword / Elasticsearch](#keyword-search-inverted-indexes-elasticsearch) · [Semantic search](#semantic-search) · [Chooser](#choosing-proximity-vs-keyword-vs-semantic).

| | Proximity | Keyword / vector |
|--|-----------|------------------|
| Primary signal | Distance / ETA | Text relevance / embedding similarity |
| Index | Geo cells, GEO, R-tree | Inverted index, ANN |
| Often combined | “Thai near me” = geo filter **then** text/rank (or vice versa) | — |

---

## Keyword search & inverted indexes (Elasticsearch)

**Problem:** find documents by **words / phrases / filters**, ranked by text relevance — product catalog, logs, help center, “search box” on a site.

Common stack: **Elasticsearch / OpenSearch** (Lucene under the hood). **Not** your system of record — index a projection of OLTP/object data.

### Inverted index (core idea)

Forward index: `docId → tokens`.  
**Inverted index:** `token → postings list` of docIds (plus positions, freqs for ranking/phrases).

```text
Docs:
  D1: "red running shoes"
  D2: "blue running shorts"

After analyze (lowercase, maybe stem):
  red      → [D1]
  running  → [D1, D2]
  shoes    → [D1]
  blue     → [D2]
  shorts   → [D2]

Query "running shoes" → intersect / score postings → rank (e.g. BM25)
```

| Stage | What happens |
|-------|----------------|
| **Analyze** | Char filter → tokenizer → token filters (lowercase, stopwords, stem, synonyms) |
| **Index** | Write tokens into inverted index (+ stored/doc values for filters & sort) |
| **Query** | Analyze query the same way → look up terms → combine (AND/OR/phrase) → **score** |
| **Rank** | **BM25** (default in ES) — TF/IDF-style relevance; boost fields (`title^3`) |

**Filters** (brand, price, `geo_distance`, status) use doc values / bitsets — cheap, often **not** scored. Interview split: **query** (relevance) vs **filter** (exact constraints).

### Elasticsearch on the HLD board

```text
Write path:  App → OLTP DB → (outbox / CDC / queue) → Indexer → ES cluster
Read path:   Client → Search API → ES → (optional hydrate from DB) → response
```

| Concept | Interview meaning |
|---------|-------------------|
| **Index** | Named collection of docs (≈ “table” of searchable JSON) |
| **Document** | One searchable unit (product, log line, article) |
| **Shard** | Horizontal split of an index; parallelism + scale |
| **Replica** | Copy of a shard for HA + read throughput |
| **Primary** | You still own truth in Postgres/S3; ES can be rebuilt |

**Why async indexer:** don’t dual-write DB + ES in one request without outbox — ES lag is OK if you state freshness SLO (seconds–minutes).

### Autocomplete

- **Edge n-grams** / completion suggester in ES, or  
- **Trie / prefix index** in Redis/app for ultra-hot prefixes  

Often both: trie/cache for typeahead, ES for full result page. See [Large Scale Search](../diagrams/large-scale-search-system/large-scale-search-system.excalidraw).

### When keyword search fails (motivation for semantic)

- Synonyms / intent: “sofa” vs “couch”, “login broken on phone” vs exact error string  
- No shared tokens with the doc  
- Multilingual fuzzy meaning  

That’s when you add **semantic** (next) or synonym lists / query rewriting.

### Interview pitfalls

- Using ES as primary DB (weak multi-doc transactions, harder correctness)  
- Sync dual-write without outbox/CDC  
- Mapping explosion / high-cardinality fields as unbounded keywords  
- Scoring everything — push exact constraints to **filters**  
- Ignoring analyze asymmetry (index analyzer ≠ search analyzer → zero hits)

**Interview line:** “Inverted index maps token → docs; ES shards that index. BM25 ranks; filters constrain. Source of truth stays in OLTP; we index via CDC/outbox.”

---

## Semantic search

**Problem:** retrieve by **meaning**, not shared keywords — RAG, “similar products”, support search (“can’t sign in on iOS”).

### How it works

```text
Index time:  text chunk → embedding model → vector → ANN index (HNSW / IVF…)
Query time:  user text → same model → vector → top-k nearest neighbors → optional rerank
```

| Piece | Role |
|-------|------|
| **Embedding** | Dense vector capturing semantics (same model at index + query) |
| **ANN index** | Approximate nearest neighbor (exact kNN too slow at scale) |
| **Stores** | Pinecone, Weaviate, Milvus, Qdrant, **pgvector**, ES/OpenSearch `dense_vector` |
| **Rerank** | Cross-encoder / BM25 blend on a small candidate set |

### Semantic vs keyword

| | **Keyword (inverted / BM25)** | **Semantic (embeddings)** |
|--|------------------------------|---------------------------|
| Match | Shared tokens / phrases | Nearby in vector space |
| Good at | SKUs, exact names, logs, boolean filters | Paraphrase, intent, “messy” language |
| Weak at | Synonyms without config; conceptual queries | Exact SKU / rare tokens; needs embed cost |
| Explainability | Highlight matched terms | Harder (“why this doc?”) |
| Freshness | Reindex tokens | Re-embed on content change (costly) |

### Hybrid (Staff default for product search / RAG)

```text
Query → (optional sparse BM25) + (dense ANN) → fuse scores / RRF → filters (ACL, geo, price) → rerank → results
```

- **ACL / tenant filters** must apply on **candidates** (don’t leak neighbor vectors across tenants).  
- RAG: retrieve chunks → LLM; cite KB ids — vector DB is not the knowledge base. Detail: [Knowledge base vs Vector DB](./data-stores.md#knowledge-base-vs-vector-db).

### MediBuddy / marketplace example

| Intent | Prefer |
|--------|--------|
| “Dr. Sharma cardiologist Koramangala” | Keyword + geo filter |
| “chest pain doctor nearby” | Semantic and/or synonym + **proximity** |
| “labs like vitamin panel” | Semantic or curated taxonomy + keyword |

### Interview pitfalls

- Embedding **without** the same model version at query time  
- No ACL filter on ANN results  
- Replacing ES entirely when users still search SKUs / order ids  
- Ignoring embed + ANN **cost/latency** in NFR  

**Interview line:** “Semantic = embed + ANN. Keyword = inverted index + BM25. Production search is usually **hybrid** plus filters; OLTP remains source of truth.”

---

## Choosing proximity vs keyword vs semantic

| Need | Primary tool |
|------|----------------|
| Near me / ETA / drivers | **Proximity** (geo index) |
| Typeahead, SKU, logs, “exact-ish” text | **Keyword / ES** |
| Paraphrase, RAG, “similar meaning” | **Semantic / vector** |
| “Italian near me” | Geo **filter** + keyword/semantic **rank** |
| Resume AI / SuperStocks-style RAG | KB + chunk + vector (+ optional BM25) |

---

## Other index / structure prerequisites

| Topic | One-liner |
|-------|-----------|
| **B-Tree** | Default relational index; great range scans |
| **Hash index** | Equality only; not range |
| **LSM Tree** | Write-optimized (Cassandra, RocksDB); compaction trade-offs — see [Data stores](./data-stores.md#lsm-trees-storage-engine) |
| **Inverted index** | Token → doc IDs — see [Keyword search](#keyword-search-inverted-indexes-elasticsearch) |
| **ANN / HNSW** | Approx nearest vectors — see [Semantic search](#semantic-search) |
| **Trie** | Prefix / autocomplete |
| **Skip list** | Ordered structure in Redis sorted sets internals (conceptual) |
| **Merkle tree** | Anti-entropy / sync verification (Dynamo-style) |
| **HyperLogLog** | Approx distinct counts (cardinality) |
| **Count-Min Sketch** | Approx frequencies |

Autocomplete / search designs lean on **trie + inverted index** (+ optional vectors) — see [Large Scale Search](../diagrams/large-scale-search-system/large-scale-search-system.excalidraw).
