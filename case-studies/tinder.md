# Design a Dating App (Tinder)

Source: Hello Interview breakdown (full) · Patterns: contention, proximity, scaling reads · Difficulty: medium

## Scope
- **FR:** profile with preferences (age range, interests, max distance); view a stack of candidates matching preferences and distance; swipe yes/no; mutual yes → match notification. Below the line: chat, photo upload pipeline, premium features.
- **NFR:** **strong consistency for swipes/matches** (if B already swiped yes on A, A's yes must produce the match); 20M DAU × ~100 swipes = ~2B swipes/day (~20k/s avg); stack loads < 300 ms; never re-show already-swiped profiles.

## Crux
Atomic mutual-swipe detection at high write volume, and generating geo+preference-filtered stacks fast while excluding a large, growing swipe history.

## Skeleton
Entities: User/Profile (location, prefs), Swipe (swiper, swipee, decision), Match. API: `POST /profile`, `GET /feed?lat&long&distance`, `POST /swipe/{userId} {decision}`.
Profile service (Postgres), Swipe service (Cassandra for durable swipes), Redis for atomic match checks, push notifications (APNs/FCM).

## Deep dives

**1. Consistent, low-latency swipes**
- ✗ Poll the DB for reciprocal swipes — not immediate.
- ◐ Transaction (insert swipe + check reverse) — Cassandra LWTs are single-partition only; 2B swipes/day can't live in one partition.
- ✓ Co-locate a pair: partition key = **sorted(userA, userB)** so both directions live in one partition; batch write + read in that partition.
- ✓ **Redis atomic check** with the same pair key: hash with both users' decisions; a Lua script does set-mine + get-theirs atomically → match. Cassandra remains the durable log. Handle Redis failover (rebuild from Cassandra; a rare missed match can be reconciled by a sweep).

**2. Fast stack generation**
◐ Indexed query (Elasticsearch/OpenSearch with geo + filters) — fresh but per-request cost and CDC lag. ◐ Precompute stacks in the background and cache — instant but stale and exhaustible. ✓ **Both:** serve a cached precomputed stack; refill/refresh via indexed query when it runs low or when the user moves/changes prefs; invalidate on significant location change.

**3. Don't re-show swiped profiles**
✗ Query swipe history and filter (large `NOT IN` sets, eventual consistency lag). ✓ Client keeps recent swipes and filters locally (covers replication lag). ✓ For heavy swipers, a per-user **Bloom filter** of swiped ids (false positives just skip a few profiles; rebuildable from Cassandra).

## Staff+ signals
- Explain why the pair-key trick turns a distributed transaction into a single-partition operation.
- Hot users (very popular profiles) skew both swipe writes and candidate reads → ranking/exposure throttling as product + system lever.
- Location privacy (coarsen to cells); feed relevance and fairness (ELO-like desirability scores) as a later ranking stage.
- Notification path reliability (push delivery is best-effort; in-app match list is the source of truth).

## Level bar
Mid: functional feed/swipe/match; traditional + geo filters; avoid re-showing. Senior: proactive tradeoffs in feed building; index choice; staleness of caches. Staff: depth across all, may steer to interesting areas, peer-level tradeoffs.
