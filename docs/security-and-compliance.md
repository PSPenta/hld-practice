# Security & Compliance

Enough security architecture for HLD rounds — especially **payments / fintech (Razorpay-class)** and FAANG threat-aware designs.

← [README](../README.md) · [Docs index](./README.md) · Related: [Data stores](./data-stores.md) · [Reliability & SLOs](./reliability-and-slos.md)

---

## Index

- [Authn vs Authz](#authn-vs-authz)
- [Tokens & sessions](#tokens-sessions)
- [Transport & data protection](#transport-data-protection)
- [Secrets & least privilege](#secrets-least-privilege)
- [PCI-DSS mindset (payments interviews)](#pci-dss-mindset-payments-interviews)
- [Threat modeling (lightweight STRIDE)](#threat-modeling-lightweight-stride)
- [Abuse & fraud controls](#abuse-fraud-controls)
- [Privacy](#privacy)

---

## Authn vs Authz

| | **Authentication (authn)** | **Authorization (authz)** |
|--|----------------------------|---------------------------|
| Question | Who are you? | What can you do? |
| Common | Session cookie, JWT, OAuth2/OIDC, mTLS service identity | RBAC, ReBAC, resource policies, API scopes |

**HLD placement:** Gateway verifies identity; **service still enforces authz** on the resource (never trust the client).

Service-to-service: network policy + **mTLS** / SPIFFE-style identity when asked for zero-trust depth.

---

## Tokens & sessions

| | **Token (e.g. JWT)** | **Server session** |
|--|----------------------|--------------------|
| State | Client holds claims (often stateless at app) | Server (or Redis) holds session |
| Revoke | Hard (wait TTL, or blocklist) | Easy (`DELETE` session) |
| Scale | Easy horizontally | Needs shared session store |
| Size / exposure | Keep access token short-lived + minimal claims | Cookie id only; data stays server-side |
| Typical | APIs, mobile, microservices | Web apps, instant logout |

Also: **refresh tokens** (rotate, store hashed); **API keys** for merchants (scoped, rotatable).

---

## Transport & data protection

- **TLS** everywhere external; prefer internal TLS for sensitive paths  
- **Encryption at rest** — disk/KMS for DBs and object storage  
- **Field-level encryption / tokenization** — PANs, national IDs  
- **Hashing ≠ encryption** — passwords use salted slow hashes (Argon2/bcrypt); see [Algorithms](./algorithms-and-indexes.md#hashing-vs-encryption)

---

## Secrets & least privilege

- Secrets in **Vault / cloud secret manager / KMS**, not git or images  
- Rotate keys; short-lived credentials where possible  
- IAM: each service account gets **minimum** queue/DB/bucket permissions  
- Separate prod/stage credentials and networks

---

## PCI-DSS mindset (payments interviews)

You are not expected to recite every control — you **are** expected to shrink scope:

| Do | Don’t |
|----|-------|
| Use payment processor / hosted fields / tokenization | Store raw PAN/CVV on your servers if avoidable |
| Isolate card data network (CDE) | Log full card numbers |
| Idempotent charge APIs + audit trail | Trust client-side “paid=true” |
| Encrypt, access-control, monitor CDE | Mix card data into general app DB casually |

**3DS / SCA**, webhooks with signature verification, and **reconciliation** belong on the payment HLD board.

See [Payment Gateway diagram](../diagrams/payment-gateway-system/payment-gateway-system.excalidraw).

---

## Threat modeling (lightweight STRIDE)

For a component, ask quickly:

| Threat | Example question |
|--------|------------------|
| Spoofing | Can someone call this admin API without merchant auth? |
| Tampering | Can webhook payload be forged? (HMAC signatures) |
| Repudiation | Do we have immutable audit logs for refunds? |
| Info disclosure | Do errors leak existence of emails / card last-4? |
| DoS | Rate limits per API key / IP? |
| Elevation | Can user A refund user B’s payment? |

Staff signal: name **2–3 concrete threats** for the critical path, not “we’ll add security later.”

---

## Abuse & fraud controls

- Rate limiting (user, IP, device, merchant)  
- Bot / device signals on login and pay  
- Velocity checks (refunds/min, cards/device)  
- Idempotency to stop double-charge retries looking like fraud  

---

## Privacy

- PII minimization; retention limits  
- Access audited; encryption; regional residency if asked (GDPR-style)  
- Separate analytics from PII when possible

---

## See also

- [Reliability & SLOs](./reliability-and-slos.md) — fail closed on payments  
- [Distributed coordination](./distributed-coordination.md) — idempotency under retries  
- [Deployment & ops](./deployment-and-ops.md) — secrets, controlled rollout  
