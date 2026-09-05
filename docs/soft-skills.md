# Soft Skills (Interview Communication)

HLD rounds grade how you **collaborate and narrate**, not only whether the boxes are “right.”

← [README](../README.md) · [Docs index](./README.md)

---

## Drive the conversation

You are the pilot. A good default loop:

1. Clarify → 2. Requirements → 3. Estimate → 4. APIs → 5. Diagram → 6. Deep dive → 7. Summary

Say the phase out loud: “I’ll lock NFRs next, then sketch APIs.”

If stuck, narrate options instead of going silent.

## State assumptions explicitly

Examples:

- “Assuming 100M DAU unless you want different numbers.”
- “Media goes to object storage; we only store URLs in DB.”
- “Strong consistency only for booking; feed can be eventual.”

Assumptions let the interviewer correct course early — that’s a positive signal.

## Prefer simple, then scale the bottleneck

MVP path:

```text
Client → LB → App → DB
```

Then: “Reads dominate → add cache.” “Spiky fan-out → add queue.”  

Jumping to Kubernetes + service mesh + CQRS + event sourcing on minute five looks like avoidance of the real problem.

## Compare two options, then commit

Template:

> “Postgres gives transactions for booking; Cassandra would scale writes but weakens the invariant we need. I’ll take Postgres with replicas and shard later by `showId`.”

Dithering forever or refusing to pick is worse than a justified “good enough” choice.

## Use the board clearly

- Label every box
- Number the steps of the main flow (1 → 2 → 3)
- Separate **read path** and **write path** if they differ
- Keep a small corner for FR / NFR / estimates / SLO

Draw for a teammate who wasn’t in your head.

## Handle pushback well

Interviewers probe on purpose.

- Don’t defend a bad idea out of ego — revise and say what changed
- If you disagree, restate the trade-off + why your NFRs still hold
- “I don’t know that product’s internals, but the pattern I’d use is…” is fine

## Time management

| If time is… | Do this |
|-------------|---------|
| Short | Happy path + one bottleneck deep dive |
| Plenty | Failure modes, security, multi-region |
| Over-indexing on schema | Zoom back to components |

Ask: “Want more depth on consistency or on scaling the chat tier?”

## SDE3 / Staff communication bar

Beyond a clean diagram, show you could **own the system in production**:

| Signal | What to say / do |
|--------|------------------|
| **SLO-first** | Propose availability/latency targets before deep tech |
| **Risk register** | “Biggest risk is double-charge; we mitigate with idempotency + ledger” |
| **Blast radius** | “Cell per merchant shard so noisy neighbor can’t take payment globally” |
| **Cost** | “Kafka is justified by replay; SQS would be cheaper if no replay needed” |
| **MVP cut** | “v1: single region + Postgres; v2: CDC to search and multi-region read” |
| **Teach** | Explain for a strong SDE2 on the loop — clarity beats cleverness |
| **Metrics for success** | “I’d watch charge success rate, p99 authorize, reconciliation lag” |
| **Org reality** | Ownership boundaries: who owns webhook delivery vs ledger |

Fintech (Razorpay-class): volunteer **PCI scope reduction, idempotency, reconciliation, fail-closed** without waiting to be asked.

## Language that scores

- “Bottleneck”, “blast radius”, “idempotent”, “degrade gracefully”, “hot key”, “SLO / p99”, “error budget”, “RPO/RTO”
- Quantify when you can (“~10K QPS”, “p99 &lt; 200ms”)
- Avoid unsupported absolutes (“Kafka is always better”)

## Language that hurts

- Buzzword salad with no requirement attached
- “We’ll use microservices” with no split reason
- Ignoring failure and retries entirely
- Designing only for the average and forgetting peaks
- Hand-waving payments security

## Practice drill

1. Pick a problem from [Popular HLDs](../README.md#popular-hlds)
2. Speak a 45–60 minute design **aloud** (record yourself)
3. Replay: clarifying questions, assumptions, trade-offs, SLO, failure mode
4. Compare to the diagram in this repo if one exists
5. Re-run once focusing only on **Staff extras** (SLO, blast radius, cost, MVP)

Communication is a skill — reps matter as much as reading diagrams.
