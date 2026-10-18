# Design Event Ticketing (Ticketmaster)

Source: Hello Interview breakdown (full) · Patterns: contention, scaling reads, real-time updates · Difficulty: medium

## Scope
- **FR:** view events; search events; book tickets. Below the line: view my bookings, admin event creation, dynamic pricing.
- **NFR:** availability for search/view, **consistency for booking (no double booking)**; handle 10M users on one event; search < 500 ms; read-heavy 100:1.

## Crux
(1) Holding a seat during checkout without double booking or stranded seats. (2) Surviving a Taylor-Swift on-sale: tens of millions of reads + a stampede of booking attempts for few seats.

## Skeleton
Entities: Event, Venue, Performer, Ticket (event_id, seat, status, price), Booking (user, tickets, status). API: `GET /events/{id}` (event + venue + seat map), `GET /events/search?q&start&end&cursor`, `POST /bookings {ticketIds}` → reserve, then payment confirms.
API gateway → Event service, Search service, Booking service → Postgres; Stripe for payment (tokenized client-side; webhook confirms).

## Deep dives

**1. Reserving tickets during checkout**
- ✗ Long-running DB lock (`SELECT FOR UPDATE` held for minutes) — locks are for milliseconds; connection exhaustion, deadlocks, crash leaves state unclear.
- ◐ Status (available/reserved/booked) + `reserved_at` + cron to release — releases late; cron failure strands seats.
- ✓ **Implicit expiry:** short transaction reserves only if `status='available' OR (status='reserved' AND expires_at < now())`, sets `expires_at = now()+10m`. A sweeper only tidies up; correctness never depends on it. Compound index on (event_id, status, expires_at).
- ✓ **Redis lock with TTL** (`SET ticket:{id} userId NX EX 600`): automatic expiry, fast under contention. Seat-map reads need reserved state → sorted set scored by expiry per event. Redis outage degrades UX (users may lose seats at payment) but **never double-books**, because the final booking is a DB transaction with OCC/conditional update. TTL expiring mid-payment → one commit fails → auto-refund; extend TTL when payment starts.
- Payment: Stripe webhook updates ticket→sold and booking→confirmed in one transaction; handler idempotent on bookingId.

**2. Scaling the view path to tens of millions**
Event/venue/performer data is read-mostly → aggressive cache (Redis, long TTL, invalidate on change), stateless horizontally scaled Event service, CDN for static seat-map assets.

**3. UX under extreme demand**
- ◐ SSE pushing seat-status changes — for mega events the map empties instantly; chaotic UX.
- ✓ **Virtual waiting room** (admin-enabled per event): users enqueue (Redis sorted set by arrival), hold an SSE connection for position/ETA, are admitted in batches as capacity/tickets allow; admitted session ids stored with TTL; Booking service rejects non-admitted requests. Protects the backend *and* fixes UX — a product solution to a technical problem (a classic senior→staff delta).

**4. Search latency**
- ◐ B-tree indexes + query tuning — `LIKE '%taylor%'` still scans.
- ✓ Postgres full-text (tsvector/GIN) at modest scale, or ✓ Elasticsearch synced via CDC for fuzzy, relevance, geo and facets.

**5. Repeated searches**
Redis cache keyed by normalized query params (short TTL); ES request cache; CDN caching only for non-personalized results.

## Staff+ signals
- State that booking is CP and browse is AP, and design the boundary.
- Bot/scalper defense (waiting room + per-user limits + CAPTCHA at admission); fairness (random admission vs FIFO).
- Hot partition: one event's ticket rows on one shard → pre-generate tickets, shard by event_id but expect the hot event to dominate; admission control is the real throttle.
- Failure analysis of the lock store and the payment webhook (retries, out-of-order).
- Pre-warm caches and pre-scale before scheduled on-sales.

## Level bar
Mid: working view/book + at least status/expiry with cron. Senior: speed through HLD; ES search, distributed lock or equivalent, scaling discussion (hints OK). Staff: 2–3 deep areas with novel insight; interviewer only focuses, never steers.
