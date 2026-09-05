# Algorithms, Indexes & Crypto Basics

Probabilistic structures, geo indexes, **proximity search**, and hashing vs encryption.

← [README](../README.md) · [Docs index](./README.md)

---

## Index

- [Bloom filters](#bloom-filters)
- [Hashing vs encryption](#hashing-vs-encryption)
- [Geo-spatial indexes](#geo-spatial-indexes)
- [Proximity search (“nearby”)](#proximity-search-nearby)
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

| | Proximity | Keyword / vector |
|--|-----------|------------------|
| Primary signal | Distance / ETA | Text relevance / embedding similarity |
| Index | Geo cells, GEO, R-tree | Inverted index, ANN |
| Often combined | “Thai near me” = geo filter **then** text/rank (or vice versa) | — |

---

## Other index / structure prerequisites

| Topic | One-liner |
|-------|-----------|
| **B-Tree** | Default relational index; great range scans |
| **Hash index** | Equality only; not range |
| **LSM Tree** | Write-optimized (Cassandra, RocksDB); compaction trade-offs — see [Data stores](./data-stores.md#lsm-trees-storage-engine) |
| **Inverted index** | Token → doc IDs (search) |
| **Trie** | Prefix / autocomplete |
| **Skip list** | Ordered structure in Redis sorted sets internals (conceptual) |
| **Merkle tree** | Anti-entropy / sync verification (Dynamo-style) |
| **HyperLogLog** | Approx distinct counts (cardinality) |
| **Count-Min Sketch** | Approx frequencies |

Autocomplete / search designs lean on **trie + inverted index** — see [Large Scale Search](../diagrams/large-scale-search-system/large-scale-search-system.excalidraw).
