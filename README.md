# HLD Practice

Interview-oriented **High-Level Design** notes and Excalidraw diagrams for **SDE3 / Staff**-style rounds (FAANG + fintech such as Razorpay).

Open files under [`diagrams/`](./diagrams) with the [Excalidraw extension](https://marketplace.visualstudio.com/items?itemName=pomdtr.excalidraw-editor) (recommended via [`.vscode/`](./.vscode) when you open this repo). Full study path: [`docs/README.md`](./docs/README.md).

## Index

| Section | What you’ll find |
|---------|------------------|
| [What is HLD?](#what-is-hld) | Definition and artifacts |
| [HLD vs LLD](#hld-vs-lld) | Scope and granularity |
| [What does the interviewer evaluate?](#what-does-the-interviewer-evaluate) | Mid/Senior vs Staff bar |
| [How a typical HLD round runs](#how-a-typical-hld-round-runs) | 45–60 min timeline |
| [Standard approach (checklist)](#standard-approach-checklist) | End-to-end design steps |
| [Prerequisites (grouped)](#prerequisites-grouped) | Links into [`docs/`](./docs/README.md) |
| [Quick references (YouTube)](#quick-references-youtube) | Deep dives & walkthrough playlists |
| [How to practice](#how-to-practice) | Using diagrams in this repo |
| [Popular HLDs](#popular-hlds) | Catalog ✅/⚠️/❌ + folder links |
| [Prep plan](./docs/prep-plan.md) | Daily/weekly DSA · LLD · HLD schedule |

**Docs curriculum:** [docs/README.md](./docs/README.md) (study order A→E). Each long doc has its own **Index** at the top.

---

## What is HLD?

**High-Level Design (HLD)** is the architectural blueprint of a system: major components, how they talk to each other, data flow, and the trade-offs that keep the system correct, available, and fast at scale.

In interviews, HLD means designing a product or platform end-to-end at the **service / storage / network** level — not class diagrams or exact SQL schemas.

Typical artifacts: FR/NFR, capacity estimates, API sketch, component diagram, entity-level data model, scaling / consistency / failure / security.

---

## HLD vs LLD

| | **HLD** | **LLD** |
|--|---------|---------|
| Focus | Architecture of the whole system | Design of a module / service |
| Granularity | Services, DBs, queues, caches, LB | Classes, methods, schemas, algorithms |
| Questions | What pieces exist? How do they scale? | How does this piece work internally? |
| Output | Box-and-arrow diagram, APIs, trade-offs | Class diagrams, sequence diagrams, detailed tables |
| Example | “Design Twitter” | “Design the feed ranking service internals” |

Rule of thumb: **Kafka vs SQS**, **SQL vs NoSQL**, **fan-out on write vs read** → HLD. **Hash maps vs trees**, exact indexes/columns → LLD. Interviewers often stay in HLD and only dip into LLD on a hot path (seat booking, idempotent payments).

---

## What does the interviewer evaluate?

### Mid → Senior (baseline)

1. Clarifying FR/NFR before drawing  
2. Structured phases (not random boxes)  
3. Right abstractions (no cargo-cult microservices)  
4. Explicit trade-offs tied to requirements  
5. Scale and bottleneck awareness  
6. Failure thinking (retries, DLQ, multi-AZ)  
7. Clear communication  

### SDE3 / Staff (expected extras)

8. **SLOs / error budgets** and load shedding  
9. **Blast radius** and isolation (cells, bulkheads)  
10. **Consistency per path**, not one global slogan  
11. **Cost and MVP vs v2** — what you won’t build yet  
12. **Multi-region / DR** only when justified (RPO/RTO)  
13. **Security & compliance** on sensitive paths (esp. payments: PCI scope, idempotency, reconciliation)  
14. Mentorship signal: teach as you design; correct yourself out loud  

Weak signals: buzzword salad, ignoring peaks, no idempotency on payments, “we’ll scale later” with no plan.

---

## How a typical HLD round runs

| Time | Phase |
|------|--------|
| 3–5 min | Clarify problem & scope |
| 5–10 min | FR / NFR + rough scale |
| 5 min | APIs / entities |
| 15–25 min | Diagram + main flows |
| 10–15 min | Deep dives (scale, consistency, failure, security) |
| Remaining | Summary, alternatives, MVP cut line |

You drive. Interviewers push with “100× traffic?” and “this node dies?”

---

## Standard approach (checklist)

1. **Clarify** — users, scope, read/write, realtime, integrations  
2. **Requirements** — FR + NFR written down (they justify later choices)  
3. **Estimate** — QPS, storage, bandwidth, cache working set ([doc](./docs/estimation-fluency.md))  
4. **APIs / entities** — resources, auth at high level  
5. **Diagram** — happy path first: `Client → DNS/CDN → LB → Gateway → Services → Cache/Queue/DB/Object store`  
6. **Deep dive** — cache, shard key, async, consistency, fan-out, rate limit, observability, multi-AZ/region, security  
7. **Summarize** — trade-offs, SLO, MVP vs v2, biggest risk  

---

## Prerequisites (grouped)

Don’t learn as a flat list — follow the curriculum in [`docs/README.md`](./docs/README.md).

| Group | Docs |
|-------|------|
| **Interview craft** | [Soft skills](./docs/soft-skills.md) · [Estimation](./docs/estimation-fluency.md) · [Prep plan](./docs/prep-plan.md) |
| **Foundation** | [Building blocks](./docs/building-blocks.md) · [Service architecture](./docs/service-architecture.md) · [Core concepts](./docs/core-concepts.md) · [Scaling](./docs/scaling.md) · [Networking & media](./docs/networking-and-media.md) |
| **Data & messaging** | [Data stores](./docs/data-stores.md) · [Caching](./docs/caching.md) · [Messaging & pipelines](./docs/messaging-and-pipelines.md) · [Algorithms & indexes](./docs/algorithms-and-indexes.md) |
| **Reliability & correctness** | [Distributed coordination](./docs/distributed-coordination.md) · [Reliability & SLOs](./docs/reliability-and-slos.md) · [Security & compliance](./docs/security-and-compliance.md) |
| **Delivery** | [Deployment & ops](./docs/deployment-and-ops.md) |

---

## Quick references (YouTube)

| Playlist | Link |
|----------|------|
| Deep Dives | [Playlist](https://www.youtube.com/playlist?list=PL5q3E8eRUieUHnsz0rh0W6AzwdVJBwEK6) |
| System Design Walkthroughs | [Playlist](https://www.youtube.com/playlist?list=PL5q3E8eRUieWtYLmRU3z94-vGRcwKr9tM) |
| System Design Interview Questions | [Playlist](https://www.youtube.com/playlist?list=PLPtUyMfD0mNJDZg50fg2CptjLBavHot47) |

Use alongside timed redesigns — watch a walkthrough, then redraw from scratch without pausing.

---

## How to practice

1. Install the recommended Excalidraw extension when prompted.  
2. Pick an HLD from the table below — start with **Experience-based** rows (resume products), then ✅ classics, then ⚠️ to harden, then ❌ stretch.  
3. Open the folder link — GitHub renders that folder’s `README.md` with the preview image.  
4. Redesign in **45–60 minutes aloud**; then gap-check against the diagram.  
5. Staff pass: add SLO, one region failure, and cost/MVP cut without prompting.  
6. For experience HLDs: practice **sanitized** versions (no internal secrets) you can redraw when asked “design something you built.”  

Each diagram folder contains:

- `README.md` — preview on GitHub when you open the folder  
- `*.excalidraw` — edit in Cursor/VS Code or [excalidraw.com](https://excalidraw.com)  
- `*.png` — preview image for GitHub folder README  

---

## Popular HLDs

**Completed** = practice-ready for an SDE3/Staff-style redraw (skeleton + enough depth to rehearse deep dives).  
**⚠️** = in repo but **not** Staff-complete yet — use as a starting board, then harden.  
**❌** = not drawn yet.

Interviewers often ask you to **design a system from your own experience** (“Walk me through something you built”, “Design X from your resume”). Treat the **Experience-based** rows below as first-class prep — same bar as classic HLDs, with domain detail only you can bring.

### Experience-based (resume products)

| HLD (angle to practice) | Product / company | Completed | Folder |
|-------------------------|-------------------|-----------|--------|
| AI Equity Research Platform (RAG, agents, SSE, credits) | SuperStocks.ai | ❌ | — |
| Identity & Multi-Profile Auth (OTP, fraud controls) | MediBuddy | ❌ | — |
| In-house Wallet & Settlements (reconciliation, ledgers) | RARIO | ❌ | — |
| B2B Lending / Loan Origination (partners, disbursal, payouts) | DigiPartner · partner.rupyy.com (CarDekho) | ❌ | — |
| Supply-Chain / OMS (orders, returns, refunds, logistics) | Purplle.com (SCM) | ❌ | — |

Related practice already in repo: [Notification System](./diagrams/notification-system/) (Purplle / MediBuddy OTP & SMS), [Payment Gateway](./diagrams/payment-gateway-system/) (Razorpay subscriptions, RARIO wallet adjacent).

### Classic interview HLDs

| HLD | Completed | Folder |
|-----|-----------|--------|
| Notification System | ✅ | [`diagrams/notification-system/`](./diagrams/notification-system/) |
| Payment Gateway | ✅ | [`diagrams/payment-gateway-system/`](./diagrams/payment-gateway-system/) |
| Social Media / News Feed | ✅ | [`diagrams/social-media-app/`](./diagrams/social-media-app/) |
| Distributed Queue | ✅ | [`diagrams/distributed-queue/`](./diagrams/distributed-queue/) |
| BookMyShow / Ticket Booking | ✅ | [`diagrams/bookmyshow-hld/`](./diagrams/bookmyshow-hld/) |
| Distributed Caching | ⚠️ | [`diagrams/distributed-caching-system/`](./diagrams/distributed-caching-system/) |
| Large Scale Search | ⚠️ | [`diagrams/large-scale-search-system/`](./diagrams/large-scale-search-system/) |
| Logging and Monitoring | ⚠️ | [`diagrams/logging-and-monitoring-system/`](./diagrams/logging-and-monitoring-system/) |
| WhatsApp / Chat | ⚠️ | [`diagrams/whatsapp-hld/`](./diagrams/whatsapp-hld/) |
| Load Balancer | ⚠️ | [`diagrams/load-balancer-hld/`](./diagrams/load-balancer-hld/) |
| URL Shortener | ❌ | — |
| Rate Limiter | ❌ | — |
| Uber / Ride Sharing | ❌ | — |
| Proximity Search / Nearby | ❌ | — ([notes](./docs/algorithms-and-indexes.md#proximity-search-nearby)) |
| YouTube / Video Streaming | ❌ | — |
| Dropbox / File Storage | ❌ | — |
| Google Docs / Collaborative Editor | ❌ | — |
| Web Crawler | ❌ | — |
| API Gateway (deep dive) | ❌ | — |
| Recommendation System | ❌ | — |
| Instagram Stories / ephemeral feed | ❌ | — |
| Unique ID Generator | ❌ | — |
| Distributed Lock / Leader Election | ❌ | — |
| Multi-region Active-Active | ❌ | — |
