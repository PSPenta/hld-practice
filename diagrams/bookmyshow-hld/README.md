# BookMyShow / Ticket Booking

<- [Back to repo](../../README.md) | [Edit diagram](./bookmyshow-hld.excalidraw)

![BookMyShow / Ticket Booking](./bookmyshow-hld.png)

## Functional requirements

1. List / search shows (filters: name, actors, time, location, free text).
2. Admin CRUD on shows / events.
3. Users can view show details and seat availability.
4. Users can book tickets (reserve -> pay -> confirm; popular-event admit/assign flow).

## Non-functional requirements

1. Low-latency search (p99 < 1s).
2. High availability for show CRUD and search.
3. **Strong consistency for seat booking** (no double booking).

## Entities

| Entity | Role |
|--------|------|
| **Event / Show** | id, name, type (movie/play/concert), duration, actors, description, `popularEvent` |
| **Seat** | showId, price, `lockedTill`, `lockedBy`, status (`available` / `locked` / `booked`) |
| **Booking** | showId, seats / seatCategory, bookedBy, status, `paymentIdempotentKey` |
| **Payment** | External gateway; idempotent key; webhook / poll / reconcile |

## APIs (board)

- `GET /shows?...` / `GET /shows/:id`
- Admin: `POST|PUT|DELETE /shows`
- Normal: `POST /booking/reserve`, `POST /booking/confirm`
- Popular surge: `POST /booking/admit`, `GET /booking/available-seats`, `POST /booking/assign-seats`

## Challenges / key points

| Challenge | What to say |
|-----------|-------------|
| **IPL / Coldplay-style surge** | Millions hit limited inventory; don't let everyone contend on the same seat rows. Use **admit queue / category allocation**, assign seats after payment, rate-limit + shed. |
| **Double booking** | Strong consistency: seat lock with TTL (`lockedTill`), conditional update / unique booking constraint, release on cancel / timeout. |
| **Hot seat / hot show** | Partition by `showId`; popular shows need dedicated pools / queues so one show doesn't melt the fleet. |
| **Payment + seat race** | Hold seat with short TTL; confirm only with payment success + **idempotency key**; reconcile stuck `processing` bookings. |
| **Search vs booking path** | Search can be eventual / cached; booking path must be strongly consistent. |
| **Retry after timeout** | Same payment idempotency key; verify lock still held by this booking before confirm. |

## Related docs

- [Idempotency](../../docs/core-concepts.md#idempotency)
- [Optimistic locking](../../docs/core-concepts.md#optimistic-locking-versioning)
- [Race conditions](../../docs/distributed-coordination.md#race-conditions)
- [Payment Gateway diagram](../payment-gateway-system/)
