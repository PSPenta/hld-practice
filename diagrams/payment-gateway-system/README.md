# Payment Gateway

<- [Back to repo](../../README.md) | [Edit diagram](./payment-gateway-system.excalidraw)

![Payment Gateway](./payment-gateway-system.png)

## Functional requirements

1. User / merchant can raise a payment request.
2. Each payment has a **dedicated unique session** (idempotent requests).
3. Multiple methods: UPI, cards, net banking, etc.
4. Clear lifecycle: Created -> Processing -> Completed / Failed / Refunded.
5. Transaction status tracking per payment.
6. Merchant notified on completion (webhooks).

## Non-functional requirements

1. Highly available outside the in-flight transaction path.
2. Each transaction **idempotent + strongly consistent** (no double charge / illegal state jumps).
3. Low latency and horizontally scalable.
4. Client / card data secured (**PCI-DSS** -- minimize PAN scope).

## Entities

| Entity | Role |
|--------|------|
| **Clients (Merchants)** | API keys, webhooks, settlement |
| **Customers** | Payers |
| **Payment Sessions** | One logical payment attempt / intent |
| **Transactions** | Lifecycle + method routing |
| **Banks / PSPs** | External authorize / capture |
| **Ledger** | Credit/debit rows keyed by session / txn |
| **Refunds** | Manual / system refunds with reason |
| **Webhooks** | Merchant event delivery + retries |

## APIs (board)

- `POST /payments/intent` -> SDK / session
- SDK: `POST /payments/initiate` (method + transactionId)
- `POST /webhook/registration`, `GET /payments/:transactionId`
- `POST /payments/:transactionId/refund`

## Challenges / key points

| Challenge | What to say |
|-----------|-------------|
| **Dedup / no double charge** | Client **idempotency key** (or session id) unique; retries return same outcome; never mint a new key per retry. Conditional state transitions only. |
| **Exactly-once effect** | Network is at-least-once; **ledger + unique constraints** make the *effect* once. |
| **Reconciliation** | Async worker: compare PSP statements / webhooks vs ledger; fix stuck `Processing`; replay merchant webhooks. |
| **Unknown outcome after timeout** | Don't blindly re-authorize; **inquire** PSP or wait webhook; reconcile. Fail closed on money uncertainty. |
| **Lifecycle integrity** | State machine: forbid illegal jumps (e.g. Completed -> Created); optimistic version / status guards. |
| **Webhook reliability** | At-least-once to merchant; signed payloads; retries + DLQ; merchant must be idempotent. |
| **PCI scope** | Prefer hosted fields / token vault; don't store raw PAN; reduce audit surface. |
| **Multi-PSP routing** | Route by method / availability; isolate failures with timeouts + circuit breaker. |
| **Refunds** | Separate idempotent refund API; ledger debit/credit; link to original txn. |

## Related docs

- [Idempotency](../../docs/core-concepts.md#idempotency)
- [Inbox / dedupe](../../docs/messaging-and-pipelines.md#inbox-dedupe-table)
- [Reliability in fintech](../../docs/reliability-and-slos.md#reliability-in-fintech-razorpay-class)
- [PCI mindset](../../docs/security-and-compliance.md#pci-dss-mindset-payments-interviews)
- [Circuit breaker](../../docs/core-concepts.md#circuit-breaker)
