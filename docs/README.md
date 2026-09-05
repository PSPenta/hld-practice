# Docs index — HLD curriculum

Study path aimed at **SDE3 / Staff** system-design interviews (FAANG-style + fintech like Razorpay). The [root README](../README.md) stays light; depth lives here.

## Index

- [How to use this](#how-to-use-this)
- [Curriculum (grouped)](#curriculum-grouped)
- [Format convention (all docs)](#format-convention-all-docs)
- [Staff / SDE3 bar (self-check)](#staff-sde3-bar-self-check)

---

## How to use this

1. Skim the README checklist until you can run a round without notes.
2. Work **Foundation → Data & messaging → Reliability → Delivery** below (not random tabs).
3. For each diagram in `diagrams/`, redesign aloud in 45–60 minutes, then diff against the file.
4. Staff bar: every design should state **SLOs, failure modes, blast radius, cost, and what you’d cut for MVP**.

---

## Curriculum (grouped)

### A. Interview craft

| Doc | Why |
|-----|-----|
| [Soft skills](./soft-skills.md) | Narrative, trade-offs, Staff-level leadership signals |
| [Estimation fluency](./estimation-fluency.md) | Justify every box with order-of-magnitude math |
| Root [README — standard approach](../README.md#standard-approach-use-this-checklist) | Round structure |

### B. Foundation (draw the boxes)

| Doc | Why |
|-----|-----|
| [Building blocks](./building-blocks.md) | DNS, CDN, LB, gateway, cache, DB, queue, search, realtime |
| [Core concepts](./core-concepts.md) | CAP, consistency, idempotency, fan-out, CRDT pointer |
| [Scaling](./scaling.md) | Bottleneck type (RPS/CPU/mem), partition vs shard |
| [Networking & media](./networking-and-media.md) | TCP/UDP, HLS/DASH, RTMP/SRT, WebRTC |

### C. Data & messaging (where state and events live)

| Doc | Why |
|-----|-----|
| [Data stores](./data-stores.md) | ACID/BASE, SQL/NoSQL, OLAP, TiDB/TSDB, LSM, Vector DB |
| [Caching](./caching.md) | Stampede, avalanche, penetration, invalidation, eviction |
| [Messaging & pipelines](./messaging-and-pipelines.md) | Queue vs pub/sub, Kafka/Rabbit/SQS, WAL, CDC, Spark/Flink |
| [Algorithms & indexes](./algorithms-and-indexes.md) | Bloom, geo, **proximity search**, hash vs encrypt |

### D. Reliability & correctness (Staff differentiator)

| Doc | Why |
|-----|-----|
| [Distributed coordination](./distributed-coordination.md) | Races, 2PC/3PC, Saga, Raft, quorum |
| [Reliability & SLOs](./reliability-and-slos.md) | SLI/SLO/SLA, error budget, multi-region, blast radius |
| [Security & compliance](./security-and-compliance.md) | Authn/z, PCI-DSS, secrets, threat model (fintech bar) |

### E. Delivery & operations

| Doc | Why |
|-----|-----|
| [Deployment & ops](./deployment-and-ops.md) | Docker, K8s, canary/blue-green, Prometheus/Grafana |

---

## Format convention (all docs)

Each doc follows:

1. **Title** + one-line purpose  
2. **Nav** — `← README` · links to related docs / diagrams  
3. **Index** — linked list of `##` sections at the top of multi-topic docs  
4. **Sections** — concept → comparison table → when to use → interview pitfalls  
5. **See also** — cross-links (no orphan topics)

If something feels like a “random glossary dump,” it belongs in the curriculum group above, not a new top-level README bullet.

---

## Staff / SDE3 bar (self-check)

You are ready for a strong round when you can, without notes:

- [ ] Drive FR/NFR → estimate → API → diagram → deep dive in 45 minutes  
- [ ] Name the **bottleneck resource** (RPS vs CPU vs IO vs connections)  
- [ ] Pick consistency **per data path**, not globally  
- [ ] Design **idempotent** writes and at-least-once consumers  
- [ ] State **SLO + error budget** and what you shed under overload  
- [ ] Describe **one region down** and **poison message** behavior  
- [ ] For payments: **idempotency, ledger, PCI scope, reconciliation**  
- [ ] Defend MVP vs v2 and **cost** of the expensive component  

Diagrams in this repo are practice keys — not cheat sheets to memorize box-for-box.
