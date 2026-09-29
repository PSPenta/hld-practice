# AI Systems (RAG, LLM in products)

Interview depth for shipping LLM features: retrieval quality, eval, and **when the model must leave the request path**.

← [README](../README.md) · [Docs index](./README.md) · Related: [Data stores — KB vs Vector DB](./data-stores.md#knowledge-base-vs-vector-db) · [Algorithms — semantic search](./algorithms-and-indexes.md#semantic-search) · [Services vs workers](./service-architecture.md#services-vs-workers)

---

## Index

- [RAG pipeline (recap)](#rag-pipeline-recap)
- [What is an embedding (for backend engineers)](#what-is-an-embedding-for-backend-engineers)
- [What breaks first in production RAG](#what-breaks-first-in-production-rag)
- [Evaluate retrieval separately from generation](#evaluate-retrieval-separately-from-generation)
- [When an LLM should not be in the request path](#when-an-llm-should-not-be-in-the-request-path)
- [Keep LLM out of the path but still use it](#keep-llm-out-of-the-path-but-still-use-it)
- [LLM cost doubled — levers](#llm-cost-doubled--levers)

---

## RAG pipeline (recap)

```text
Docs (KB) → chunk + embed → Vector DB (+ optional BM25)
User query → embed → retrieve top-k → (rerank / ACL filter) → LLM → answer + citations
```

Vector DB is a **retrieval index**, not the source of truth. ACL must filter **before** or **with** retrieval.

---

## What is an embedding (for backend engineers)

An **embedding** is a **fixed-length float vector** produced by a model from text (or image). Nearby vectors ≈ similar meaning. You store vectors in an ANN index; at query time you embed the question with the **same model** and search nearest neighbors.

**One thing people get wrong:** indexing with model A / version 1 and querying with model B / version 2 — spaces don’t align → garbage retrieval. Pin **model id + version** for embed-at-index and embed-at-query.

Also wrong: treating the vector DB as the source of truth (KB/docs still own content + ACL).

More: [Semantic search](./algorithms-and-indexes.md#semantic-search).

---

## What breaks first in production RAG

Usually **not** “the model is dumb” first — the **context pipeline** fails:

| Failure | Symptom | Fix direction |
|---------|---------|----------------|
| **Bad chunking** | Answers miss facts that exist in docs | Smaller/overlap chunks; structure-aware splits |
| **Wrong / stale index** | Out-of-date or missing embeddings after doc updates | Re-embed pipeline; version corpus; freshness SLO |
| **Retrieval miss** | Right doc never in top-k | Hybrid BM25+vector; better embeddings; query rewrite |
| **ACL leak / over-filter** | Wrong tenant data or empty context | Filter by tenant **in retrieval**, test with adversarial users |
| **Context overflow** | Truncate important chunks | Rank/rerank; map-reduce summarize; smaller chunks |
| **Latency / cost** | p99 / bill blow up | Cache embeddings & frequent answers; smaller models; async |
| **Hallucination with weak context** | Fluent wrong answer | Require citations; refuse if retrieval score low; grounded prompts |

**Interview line:** “Prod RAG dies on chunking, freshness, ACL, and retrieval recall before ‘pick a bigger model.’”

---

## Evaluate retrieval separately from generation

Don’t only grade final answers — that mixes two systems.

| Layer | What you measure | How |
|-------|------------------|-----|
| **Retrieval** | Did we fetch the right docs/chunks? | Golden set: question → relevant doc ids; **Recall@k**, **MRR**, nDCG |
| **Generation** | Given *fixed* context, is the answer faithful? | Same context every run; score groundedness / citation accuracy / exact facts |
| **End-to-end** | User-facing quality | Online thumbs, task success — after offline gates pass |

**Practice:** fix retrieval until Recall@k is good **on a frozen corpus**, then tune prompts/model. Otherwise you can’t tell which layer regressed.

---

## When an LLM should not be in the request path

Keep the model **off the synchronous user/API path** when:

| Constraint | Why |
|------------|-----|
| **Hard p99 / SLO** | LLM latency is fat-tailed and vendor-dependent |
| **Money / auth / ledger** | Need determinism, idempotency, audit — not probabilistic text |
| **High QPS × cost** | Tokens × RPS blows budget |
| **Must work offline / degraded** | Vendor outage shouldn’t take down checkout |

OK **on** the request path: low-QPS assistive UX (explain report, draft message) with timeouts + fallbacks.

---

## Keep LLM out of the path but still use it

| Pattern | Idea |
|---------|------|
| **Async worker** | API accepts job → queue → worker calls LLM → store result → notify (SSE/webhook/poll) |
| **Precompute** | Nightly / on-doc-change: summarize, embed, generate FAQs into KB/cache |
| **Cache** | Cache embeddings and frequent (query → answer) with TTL; stampede protection |
| **Human-in-the-loop** | Model drafts; human/rules approve before side effect |
| **Rules / templates first** | Deterministic path for hot cases; LLM only for long-tail |

**Interview line:** “Accept fast and enqueue; LLM in a worker with retries/DLQ. Sync path stays within SLO without the model.”

See also: [Services vs workers](./service-architecture.md#services-vs-workers).

---

## LLM cost doubled — levers

Cost ≈ **tokens in × tokens out × price × QPS** (plus embedding/rerank calls).

| Lever | What you change | Quality impact |
|-------|-----------------|----------------|
| **Cache** hits (prompt/answer/embedding) | Fewer model calls | **Least hurt** if cache key is correct |
| **Prompt / context trim** | Fewer input tokens | Low–medium if you keep needed citations |
| **Cheaper / smaller model** on easy path | Lower $/1K tokens | Medium — route hard cases to big model |
| **Lower max tokens / stop early** | Fewer output tokens | Medium — shorter answers |
| **Retrieve less / better** (top-k, rerank) | Smaller context | Can **improve** quality if noise drops |
| **Async / batch** | Same tokens, better utilization | Neutral to UX latency |
| **Rate limit / product quotas** | Less usage | Product constraint |
| **Fine-tune / distill** | Long-term cheaper | Upfront cost; quality TBD |

### Which lever hurts quality least?

Usually **caching identical/near-identical requests** and **dropping unused context**, then **model routing** (small model default). Cutting retrieval/context blindly or forcing a tiny model on hard tasks hurts quality most.

**Interview line:** “I’d check token volume vs QPS first — cache and trim before I sacrifice the model.”
