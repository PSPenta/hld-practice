# Algorithms, Indexes & Crypto Basics

Probabilistic structures, geo indexes, and the hashing vs encryption distinction.

← [README](../README.md) · [Docs index](./README.md)

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

Location queries: “restaurants within 2 km”, “drivers near rider.”

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

**Interview pattern (Uber-like):**

1. Update driver location in **Redis GEO / H3 sets** (hot path)  
2. Query candidates in cell / radius  
3. Exact haversine filter  
4. Persist trips / history in **Postgres + PostGIS** (or similar)

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
