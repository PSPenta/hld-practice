# Notification System

? [Back to repo](../../README.md) · [Edit diagram](./notification-system.excalidraw)

![Notification System](./notification-system.png)

## Interview must-haves (this design)

### Idempotency (client-owned key)

`POST /send` (or `/v1/notifications`) **requires** `idempotencyKey` from the client.

| Failure | Without key | With `idempotencyKey` |
|---------|-------------|------------------------|
| Accept succeeded, response lost | Client retries ? **duplicate SMS/email** | Same key ? return prior ack; **no second enqueue** |

- Unique constraint: `(clientId, idempotencyKey)`
- `GET /notification/:idempotencyKey` for status / poll
- Workers stay at-least-once ? also dedupe on `idempotencyKey` / notification id before provider call

Related: [Idempotency](../../docs/core-concepts.md#idempotency)

### Rate limiter (3 tiers)

| Tier | Idea |
|------|------|
| **Free / Pro** | Fixed RPS (+ burst) per client / API key |
| **Enterprise** | **Reserved** bandwidth + access to a **shared pool** (weighted / fair share when pool has headroom) |

Place limiter at API edge (before enqueue). Return `429` with retry guidance. Token bucket per key; enterprise pool is a second bucket (or weighted fair queue) so noisy neighbors don’t eat reserved capacity.

Related: [Rate limiting](../../docs/core-concepts.md#rate-limiting)

### Shape of the board

Client ? API (validate, hydrate template/user, **idempotent accept**, rate limit) ? channel queues ? workers ? APNS / FCM / Twilio / SendGrid ? log + retry / DLQ.
