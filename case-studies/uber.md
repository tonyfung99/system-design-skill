# Design Ride Sharing (Uber / Lyft)

Source: Hello Interview breakdown (full) · Patterns: proximity, contention, real-time updates, multi-step processes · Difficulty: hard

## Scope
- **FR:** rider enters pickup + destination → fare estimate; requests a ride; matched with a nearby available driver; driver accepts/declines and navigates. Below the line: ratings, scheduled rides, pooling, payments detail.
- **NFR:** match within < 1 min (or fail); **strong consistency in matching** (a driver never gets two rides); high throughput (100k requests from one location at peak, e.g., stadium); location updates from millions of drivers.

## Crux
High-frequency location writes + fast proximity queries, and a matching workflow that offers one driver at a time with timeouts, without double-assigning drivers or dropping requests under surge.

## Skeleton
Entities: Rider, Driver (status), Fare (estimate), Ride (state machine), Location. API: `POST /fare`, `POST /rides {fareId}`, `POST /drivers/location {lat,long}`, `PATCH /rides/{id} {accept|decline}`.
Services: Ride service, Location service, Matching service, Notification (push to driver). Postgres/DynamoDB for rides and fares; mapping/ETA provider.

## Deep dives

**1. Location updates & proximity search**
~10M drivers × update every 5 s ≈ 2M writes/s.
- ✗ Write every update to the primary DB and query lat/long ranges.
- ◐ Batch updates + PostGIS/quadtree — lag, expensive rebuilds.
- ✓ **Redis GEO** (geohash in sorted sets): `GEOADD` per update, `GEOSEARCH` by radius; partition by city/region; TTL/expiry for offline drivers. Ephemeral data → losing it on failover is tolerable (drivers resend within seconds).

**2. Reduce update load without losing accuracy**
✓ **Adaptive update intervals** on device: less often when stationary or offline, more often when moving fast or near a pending pickup.

**3. One request per driver at a time**
✗ In-memory app locks with timers (lost on crash, not shared). ◐ DB status `outstanding_request` + timeout in the service (stuck states on crash). ✓ **Redis lock per driver with TTL = acceptance window (~10 s)** (`SET driver:{id} rideId NX EX 10`); auto-releases on no response. Ride assignment itself is a conditional DB write (`UPDATE rides SET driver=? WHERE id=? AND driver IS NULL`) so correctness doesn't rest on Redis.

**4. Don't drop requests at peak**
✗ First-come without buffering. ✓ **Queue (Kafka) + autoscaled matching workers**; commit offset only after a match/terminal outcome so a crashed worker's request is reprocessed; partition by region.

**5. Driver doesn't respond**
◐ Delay queue (SQS) message at timeout → if still unassigned, offer to next ranked driver. ✓ **Durable execution** (Temporal/Step Functions): the matching workflow (rank candidates → offer → wait for signal or 10 s timer → next) survives crashes and keeps state.

**6. Scale & latency further**
Geo-sharding by region (rides, drivers, matching all local), read replicas, region-local deployments; consistent hashing for rebalancing hot cities.

## Staff+ signals
- H3 hexagonal cells for supply/demand aggregation (surge pricing) — distinct from point lookup.
- ETA-based ranking (not distance), batching matches over a short window for global optimality vs greedy.
- Failure analysis: Redis partition (stale driver locations, stale locks), workflow engine outage, push notification delays.
- Hot spot: stadium event — admission/queueing, pre-positioning, geo-partition splitting.

## Level bar
Mid: functional; knows a spatial index is needed; at least the DB-status locking solution. Senior: 2+ deep dives in detail (location search, locking, queueing) with tradeoffs. Staff: 3+ deep areas, novel insight, interviewer learns something.
