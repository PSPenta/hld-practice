# Large Scale Search

<- [Back to repo](../../README.md) | [Edit diagram](./large-scale-search-system.excalidraw)

![Large Scale Search](./large-scale-search-system.png)

## Functional requirements

1. Full-text **keyword** search on large documents.
2. Ranked search results.
3. Pagination of results.
4. Near real-time indexing for recently added/updated docs.
5. Autocomplete suggestions.

## Non-functional requirements

1. Search latency p99 < **200ms**.
2. Autocomplete p99 < **50ms**.
3. High availability (no SPOF).
4. Eventual consistency between OLTP source and search index.

## Entities / components

| Piece | Role |
|-------|------|
| **CRUD Service + Database** | Source of truth for documents |
| **CDC (Debezium -> Kafka)** | Async projection into search |
| **Search Service / Search Engine** | Query orchestrator |
| **Tokeniser worker** | Analyze text -> tokens |
| **Indexing worker** | Maintain inverted index + document store + trie |
| **Ranker worker** | **BM25** (TF, IDF, doc length) |
| **Inverted Index** | `token -> [{docId, TF, docLen}, ...]` |
| **Document Store** | Hydrate full rows for results |
| **Trie** | Prefix autocomplete |
| **Redis** | Hot query cache + prefix cache |

## APIs (board)

- `POST /save`
- `GET /search?q=&limit=&offset=` (prefer **cursor** for deep pages)
- `GET /autocomplete?q=`

## Challenges / key points

| Challenge | What to say |
|-----------|-------------|
| **Write sync / freshness** | Don't dual-write in `/save`. DB commit -> CDC -> indexer. Lag SLO ~**200ms-2s**; alert above that; index rebuildable from DB. |
| **Update / delete in index** | CDC must **rewrite** postings (not append-only) and remove deleted docs from index/trie. |
| **Ranking** | Not "max tokens matched" -- **BM25** needs TF + IDF + length in postings; optional field boosts. |
| **HA / sharding** | Shard inverted index / doc store (e.g. hash ring / consistent hashing) + **read replicas**; Search Service fans out and merges. |
| **Autocomplete latency** | Trie + Redis prefix cache; don't scan inverted index for typeahead. |
| **Cache failure modes** | Stampede (NX/singleflight), avalanche (TTL jitter), penetration (negative cache). |
| **Pagination** | `offset` OK early; deep pages -> **cursor / search_after / keyset**. |
| **Semantic / proximity (extensions)** | Vectors + **ANN** for paraphrase; geo/proximity is a **different** problem (H3/geohash) -- add only if FR asks. |
| **ACL / filters** | Enforce `tenant_id` / status at retrieve; don't post-filter after leaky top-k. |

## Related docs

- [Keyword / BM25 / ES](../../docs/algorithms-and-indexes.md#keyword-search-inverted-indexes-elasticsearch)
- [Cursor vs offset](../../docs/algorithms-and-indexes.md#cursor-vs-offset-paginated-queries)
- [Semantic / ANN](../../docs/algorithms-and-indexes.md#semantic-search)
- [Caching deep dive](../../docs/caching.md)
- [CDC](../../docs/messaging-and-pipelines.md#cdc-change-data-capture)
