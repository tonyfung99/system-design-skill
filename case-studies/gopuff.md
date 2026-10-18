# Design Local Delivery (Gopuff)

Source: full breakdown · Patterns: scaling reads, contention, proximity · Difficulty: easy

## Scope
- **FR:** query availability of items deliverable within 1 hour (union of inventory across nearby distribution centers); order multiple items at once. Below the line: delivery routing, payments detail.
- **NFR:** availability queries < 100 ms (they back search); ordering strongly consistent (no two customers buy the same physical unit); 10k DCs, 100k catalog items; ~10M orders/day; availability reads ≫ orders.

## Crux
"Nearby" is defined by **drive time**, not distance, and availability must be fast while ordering stays strictly consistent.

## Skeleton
Entities: Item (catalog), DistributionCenter (location), Inventory (dc_id, item_id, qty), Order, OrderItem. API: `GET /availability?lat&long&items`, `POST /orders`.
Nearby Service (DC list in memory) → Availability Service → Inventory DB; Order Service → same Postgres.

## Deep dives

**1. Nearby DCs with traffic**
- ✗ Euclidean/Haversine radius in SQL — ignores roads and traffic.
- ✗ Travel-time API against all 10k DCs — slow, expensive.
- ✓ DCs barely change → sync to memory every few minutes; **prefilter** by optimistic radius (≈ 60 miles = max 1-hour drive), then call a travel-time service only for candidates; keep those ≤ 60 min.

**2. Fast, scalable availability**
- ✓ Cache availability results in Redis with short TTL (~1 min); Order Service invalidates affected keys on writes.
- ✓ Postgres read replicas for availability (tolerates slight staleness); orders go to the primary. Partition inventory by region (e.g., zip-prefix region id) so queries touch 1–2 partitions.

**3. Consistent ordering**
- ◐ Separate inventory store + distributed lock + order DB — two systems to keep consistent, lock failure modes.
- ✓ **Orders and inventory in one Postgres**, one transaction: check & decrement each item (`UPDATE … SET qty = qty - n WHERE … AND qty >= n`) and insert order rows; SERIALIZABLE (or row locks in consistent order) so concurrent orders for the last unit can't both succeed. 10M orders/day ≈ 115/s avg → trivially within one primary.

## Staff+ signals
- Choosing *not* to distribute: the scale numbers make a single Postgres the simplest correct answer.
- Multi-item orders across multiple DCs: pick fulfilment DC(s) deterministically; lock rows in sorted order to avoid deadlocks.
- Reservation at cart time vs decrement at checkout (UX vs inventory hoarding).
- Physical reality: inventory counts drift (damage, theft) → reconciliation with warehouse systems.
- Precompute per-region availability for hot items; cache keyed by region not raw lat/long.

## Traps
Straight-line distance; distributed locks when one DB transaction suffices; cache keyed by exact coordinates (no hits).
