# Distributed Coordination & Transactions

Race conditions, multi-step workflows, and consensus — when “just write to the DB” isn’t enough.

← [README](../README.md) · [Docs index](./README.md)

---

## Index

- [Race conditions](#race-conditions)
- [Distributed transactions](#distributed-transactions)
- [Two-phase commit (2PC) & 3PC](#two-phase-commit-2pc-3pc)
- [Saga](#saga)
- [Raft (consensus)](#raft-consensus)
- [Leader / coordinator dies mid-transaction](#leader-coordinator-dies-mid-transaction)
- [ZooKeeper / etcd / Consul (coordination services)](#zookeeper-etcd-consul-coordination-services)
- [Other coordination prerequisites](#other-coordination-prerequisites)

---

## Race conditions

**What:** Outcome depends on timing of concurrent operations (two bookers, one seat).

**Examples**

- Double booking a seat
- Double spend / double charge
- Lost update on `balance = balance - x` without locking

**Mitigations (pick by constraint)**

| Tool | Idea |
|------|------|
| **DB transaction + row lock** (`SELECT … FOR UPDATE`) | Serialize critical section |
| **Optimistic locking** | Version column; retry on conflict |
| **Conditional write** | `UPDATE … WHERE status='free'` — 0 rows = lose race |
| **Unique constraint** | DB rejects duplicate booking row |
| **Distributed lock** (Redis/ZooKeeper) | Last resort; define TTL & fencing |
| **Single-threaded partition** | All ops for `seatId` on one queue partition |

Interviewers love booking/payment races — name the invariant and the mechanism.

---

## Distributed transactions

A transaction spanning **multiple services or databases**. Hard because of partial failure.

Avoid when possible: **single DB transaction** for the critical invariant; async the rest.

When you can’t:

| Approach | Summary |
|----------|---------|
| **2PC / 3PC** | Coordinated commit across resources |
| **Saga** | Sequence of local tx + compensations |
| **Outbox + async** | Local tx, then propagate |
| **Idempotent retry** | At-least-once with dedupe |

---

## Two-phase commit (2PC) & 3PC

### 2PC

1. **Prepare** — coordinator asks cohorts to prepare; cohorts vote yes (locked) / no  
2. **Commit/Abort** — if all yes → commit; else abort

**Pros:** Strong atomicity across resources (classic XA).  
**Cons:** Blocking if coordinator dies after prepare (detail: [Leader / coordinator dies mid-transaction](#leader-coordinator-dies-mid-transaction)); high latency; poor for long / high-scale microservices.

### 3PC

Adds a **pre-commit** phase to reduce blocking in some failure cases. Still complex, rarely used as the default microservice pattern. Know it exists; prefer sagas/outbox in HLD interviews unless asked about XA.

---

## Saga

Long-running business transaction as **local commits** + **compensating actions** on failure.

Example — book trip:

1. Reserve flight → 2. Reserve hotel → 3. Charge card  
On failure at 3: compensate hotel, compensate flight.

| Style | How |
|-------|-----|
| **Choreography** | Each service emits events; others react (simple, harder to trace) |
| **Orchestration** | Central saga orchestrator calls steps (clearer control flow) |

**Properties:** Not ACID atomic across the whole saga — intermediate states exist. Need idempotent steps and well-defined compensations (which aren’t always perfect reverses).

Great fit: orders, multi-step bookings, provisioning. Poor fit: bank ledger that must never show money in two places without a clear model.

---

## Raft (consensus)

Algorithm for **replicated state machine / leader election** (etcd, Consul, many systems inspired by it).

Core ideas:

- Elected **leader** handles client writes
- Log replicated to **majority (quorum)** before commit
- Followers apply committed log in order

**Use in HLD:** metadata stores, config, locks, service discovery backends — “we need quorum consensus for control plane,” not for every user request path.

Related: **Paxos** (harder to explain), **quorum N/2+1**, **split brain** prevention.

---

## Leader / coordinator dies mid-transaction

“Leader dies mid-tx” means different things. Clarify which leader in the interview.

### 2PC coordinator dies

| When it dies | What cohorts see | Outcome |
|--------------|------------------|---------|
| **Before** any Prepare | Nothing locked | Tx never started — safe to retry with a new coordinator / new tx id |
| **After Prepare, before Commit/Abort** | Cohorts are **prepared** (resources locked) | **Blocking:** they don’t know commit vs abort until a coordinator returns |
| **After** Commit/Abort reached some but not all | Partial delivery | Recovery must **replay the decision** so everyone ends the same |

**Recovery:** new coordinator reads a durable **decision log**. If Commit was decided → send Commit to all; if Abort → Abort; if **no decision logged** yet → typically **Abort** (or stay blocked — this is why 2PC is painful at scale).

### Raft / etcd-style leader dies

Writes: client → leader → replicate to **majority** → commit → apply.

| When leader dies | Result |
|------------------|--------|
| Log entry **not** on a majority | Not committed — **lost**; client retries; new leader won’t have it |
| Entry **on a majority**, client got no ACK | Still **committed**; new leader has it; client retry needs **idempotency** |
| Died after commit, mid-apply | Followers/new leader catch up from the log; apply in order |

This is atomicity of the **replicated log**, not a cross-service business transaction. Multi-service flows still use saga/outbox on top.

### Saga orchestrator dies

Steps already **locally committed**. Orchestrator death does **not** roll back the world.

- New orchestrator (or choreography consumer) resumes from **persisted saga state** (which step completed)
- Retries the next step **idempotently**
- On hard failure → run **compensations**

User may briefly see intermediate states (“hotel reserved, payment pending”) — **eventual** consistency, not 2PC atomicity.

### Interview line

> “2PC coordinator death after prepare **blocks** cohorts on locks until decision-log recovery. Raft leader death: only **majority-replicated** entries survive; clients retry with idempotency. For product flows we prefer **sagas/outbox** so a dead orchestrator **resumes or compensates** — we don’t hold cross-service locks.”

---

## ZooKeeper / etcd / Consul (coordination services)

Externalize:

- Leader election
- Config / feature flags
- Service discovery
- Distributed locks (carefully)

Don’t put high-QPS user data here — keep coordination **low volume, high importance**.

**Kafka:** classic clusters used ZooKeeper for **controller election and partition metadata** (not for message payloads). Modern Kafka replaces that with **KRaft**. SQS has no equivalent in your diagram (AWS-managed); RabbitMQ uses **in-broker clustering / Raft quorum queues**, not ZK. Full comparison: [Messaging — Cluster metadata & coordination](./messaging-and-pipelines.md#cluster-metadata-coordination-zookeeper-kraft-and-alternatives).

---

## Other coordination prerequisites

- **Quorum reads/writes** — Dynamo-style `R + W > N` for consistency tuning
- **Leader vs multi-leader vs leaderless** replication
- **Clock skew** — don’t trust wall clock for ordering; use Lamport/vector clocks or DB order when needed
- **Fencing tokens** — lock breakers so old lock holders can’t write
- **Hystrix-style bulkheads** — isolate pools so one dependency can’t eat all threads
