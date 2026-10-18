# Design a Flash Sale

Source: Hello Interview outline (premium); answers are independent · Patterns: contention, scaling writes, real-time updates · Difficulty: hard

## Scope
- **FR:** view the flash-sale item; secure/reserve an available unit for a limited time while the sale is active; complete payment to purchase. Below the line: multi-item carts, recommendations.
- **NFR:** **never oversell** (strong consistency on inventory); millions of users arriving at the same second for ~thousands of units; reservations expire promptly; fair access; the rest of the site stays up.

## Crux
Extreme contention on one inventory counter plus a traffic spike orders of magnitude above capacity — solved more by **admission control** than by a faster lock.

## Skeleton
Entities: Sale (item, start, end, total_units), Reservation (user, sale, expires_at, status), Order. API: `GET /sales/{id}` (CDN-cached), `POST /sales/{id}/reservations` → reservation or sold-out, `POST /reservations/{id}/purchase`.
Edge/CDN + waiting room → Sale service → inventory authority → Postgres (reservations, orders) → payment workflow.

## Deep dives

**1. Consistency under extreme contention**
- ✗ Read count, check, write — oversells.
- ◐ `UPDATE sale SET available = available - 1 WHERE id=? AND available > 0` — correct, but one hot row serializes everything; DB melts at millions of attempts.
- ✓ **Single-writer inventory in Redis**: Lua script decrements and records the reservation atomically (`DECR` if > 0, `SET res:{user} … EX`); durable reservation row written after; Redis acts as a fast gate in front of a DB that only sees winners (~thousands of writes). Or pre-generate **tokens** (one per unit) and pop them atomically. Reconcile Redis vs DB; rebuild Redis from DB on failover (briefly pause sales rather than risk overselling).
- Per-user limit enforced in the same script.

**2. Release expired reservations**
Inline expiry check (reservation valid only if `expires_at > now`) so correctness never waits on cleanup; return units to the pool via sorted set of expiries swept every second, keyspace expiry events (best-effort), or durable timers (Temporal) — sweeper failures delay re-release but never cause oversell.

**3. Millions arriving at once**
Serve the product page from CDN; static "sale starts in" countdown client-side; **virtual waiting room**: admit users at a controlled rate (token issued with signed admission), reject fast once sold out (sold-out flag cached at the edge); shed load at the edge before it reaches origin; isolate the sale path from the rest of the site (separate services/capacity).

**4. Fairness**
FIFO by arrival in the waiting room vs randomized lottery among those present at start (defeats bots that win by milliseconds); per-account/payment limits; bot detection (CAPTCHA at admission, device fingerprinting); push queue position via SSE.

## Staff+ signals
- Quantify: 5M users, 10k units → 99.8% will lose; design the "lose fast and cheaply" path as carefully as the win path.
- Separate gate (Redis, fast, approximate-safe) from record of truth (DB) with reconciliation.
- Pre-scale, load-test, and kill switches; degrade non-essential features during the sale.
- Payment failures/abandonment return inventory; second-chance waves.

## Traps
DB row lock as the only mechanism; no admission control; cron-only reservation expiry; letting the sale take down the main site.
