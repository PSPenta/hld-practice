# Distributed Coordination & Transactions

Race conditions, multi-step workflows, and consensus — when “just write to the DB” isn’t enough.

← [README](../README.md) · [Docs index](./README.md)

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
**Cons:** Blocking if coordinator dies after prepare; high latency; poor for long / high-scale microservices.

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

**Properties:** Not ACID atomic across the whole saga — intermediate states exist. Need idempotent steps and well-defined compensations ( whicaren’t always perfect reverses).

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

## ZooKeeper / etcd / Consul (coordination services)

Externalize:

- Leader election
- Config / feature flags
- Service discovery
- Distributed locks (carefully)

Don’t put high-QPS user data here — keep coordination **low volume, high importance**.

---

## Other coordination prerequisites

- **Quorum reads/writes** — Dynamo-style `R + W > N` for consistency tuning
- **Leader vs multi-leader vs leaderless** replication
- **Clock skew** — don’t trust wall clock for ordering; use Lamport/vector clocks or DB order when needed
- **Fencing tokens** — lock breakers so old lock holders can’t write
- **Hystrix-style bulkheads** — isolate pools so one dependency can’t eat all threads
