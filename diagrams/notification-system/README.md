# Notification System

<- [Back to repo](../../README.md) | [Edit diagram](./notification-system.excalidraw)

![Notification System](./notification-system.png)

## Functional requirements

1. Send **SMS, WhatsApp, Email, Push, webhooks**.
2. Support notification types: transactional, promotional, general, etc.
3. Opt-in / opt-out for promotional.
4. Configure webhooks for merchants.
5. Batching of promotional notifications.

## Non-functional requirements

1. High availability + eventual consistency.
2. Transactional notification p99 < **5s** (end-to-end).
3. Promotional notification p99 < **30m**.
4. Volume ~**50M/day** (plan **10x** for festivals / campaigns).

## Entities

| Entity | Role |
|--------|------|
| **Client** | Merchant; opted channels, overall quota, throttle |
| **Template** | Channel, category (promo/trans), content / webhook URL, per-template quota |
| **Notification** | Idempotency key, status, provider msg id, recipient |
| **CommsProvider** | Provider key, channel, priority, weight, config |
| **PromotionalOutbox** | Batched promo drain (e.g. 50k/min); optional transactional outbox too |

## APIs (board)

- `POST /send` -- **requires client `idempotencyKey`** (same key on retry)
- `GET /notification/:idempotencyKey` -- status / poll

## Challenges / key points

| Challenge | What to say |
|-----------|-------------|
| **Duplicate sends** | Client-owned **idempotencyKey**; unique `(clientId, key)`; return prior ack on retry; workers dedupe before provider call. |
| **Festival / campaign surge** | Accept fast -> queue; **outbox** for promo batching; scale consumers; don't block API on provider RTT. |
| **Vendor / provider limits** | Per-provider rate limits + priority/weight routing; template + client quotas; 429 / backoff to providers; DLQ after N retries. |
| **Transactional vs promo SLO** | Separate topics / priority; trans path low latency; promo can lag minutes via outbox drain. |
| **At-least-once to providers** | Retries + DLQ; optional **transactional outbox** so accept never loses a send when broker is down. |
| **Opt-out / compliance** | Check opt-in before promo enqueue; DLT / template ids for SMS regs. |
| **Multi-provider failover** | Priority + weight; mark provider inactive; don't lose messages mid-failover. |
| **Rate limiter (3 tiers)** | Free/Pro caps; Enterprise reserved + shared pool -- at gateway before enqueue. |

## Interview must-haves (short)

### Idempotency (client-owned key)

| Failure | Without key | With `idempotencyKey` |
|---------|-------------|------------------------|
| Accept OK, response lost | Retry -> **duplicate SMS/email** | Same key -> prior ack; **no second enqueue** |

Related: [Idempotency](../../docs/core-concepts.md#idempotency)

### Rate limiter (3 tiers)

| Tier | Idea |
|------|------|
| **Free / Pro** | Fixed RPS (+ burst) per API key |
| **Enterprise** | Reserved bandwidth + fair/weighted **shared pool** |

Related: [Rate limiting](../../docs/core-concepts.md#rate-limiting)

## Shape of the board

Client -> Gateway (auth, rate limit) -> API (validate, template, **idempotent accept**) -> channel queues / outbox -> workers -> providers (Kaleyra, SES, FCM, ...) -> retry / DLQ -> optional reconcile.

## Related docs

- [Messaging & DLQ](../../docs/messaging-and-pipelines.md)
- [Transactional outbox](../../docs/messaging-and-pipelines.md#transactional-outbox)
- [Retries / backoff](../../docs/reliability-and-slos.md#exponential-backoff)
