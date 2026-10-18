# Design a News Feed (Facebook / Twitter)

Source: Hello Interview breakdown (full) · Patterns: scaling reads, fan-out · Difficulty: medium

## Scope
- **FR:** create posts; follow/friend users; view feed of followed users' posts, reverse chronological; paginate. Below the line: ranking, comments, likes, media.
- **NFR:** available (tolerate ≤ 1 min staleness); post + feed < 500 ms; 2B users; unlimited follows and followers.

## Crux
**Fan-out.** Read-time assembly breaks for users following thousands; write-time fan-out breaks for users with millions of followers. Plus hot posts read by everyone.

## Skeleton
Entities: User, Follow (follower, followee), Post, FeedEntry. API: `POST /posts`, `PUT /users/{id}/followers`, `GET /feed?cursor&pageSize`.
DynamoDB (or Cassandra): Post (PK post_id; GSI creator_id + created_at), Follow (PK follower, SK followee; GSI on followee). Feed service; Post service.

## Deep dives

**1. Users who follow many accounts (fan-out on read is too slow)**
Precompute feeds: `PrecomputedFeed` keyed by user_id holding the latest ~200 post ids (compact). On post creation, append to each follower's feed. Also consider product limits (5,000 friends cap) — a staff move.

**2. Users with huge follower counts (fan-out on write explodes)**
- ✗ Post service writes to millions of feeds synchronously.
- ◐ Async feed workers behind a queue (SQS/Kafka) — scales but a celebrity post = millions of writes, wasteful for inactive followers.
- ✓ **Hybrid:** fan-out on write for normal accounts; for celebrities (above a follower threshold), don't fan out — merge their recent posts at read time into the precomputed feed. Optionally skip fan-out to inactive users and rebuild on login.

**3. Uneven post reads (viral post = hot key)**
- ◐ Distributed post cache (LRU, long TTL, invalidate on edit) — the shard holding a viral post is still hot.
- ✓ **Redundant (replicated) post cache:** N independent cache instances each able to serve any post; LB spreads reads, so a hot post's load divides by N. Fewer unique posts fit, acceptable because the backing store absorbs the tail.

## Staff+ signals
- Quantify fan-out cost (avg followers × posts/day) to set the celebrity threshold; mention the follow graph's power law.
- Feed storage choice: Redis lists/sorted sets vs DynamoDB rows; memory sizing (200 ids × 8 B × active users).
- Consistency behaviors: deleted/edited posts in precomputed feeds (filter at read, tombstones); unfollow cleanup lazily.
- Pagination with cursors over merged sources (precomputed + celebrity pull) — stable ordering by (timestamp, post_id).
- Ranking as a later stage (candidate generation → ranker), not part of v1.
- Backfill/rebuild of feeds when the fan-out pipeline lags; lag SLO = 1 minute.

## Level bar
Mid: functional design with some good solutions. Senior: 2+ deep dives, proactively surfaces fan-out issues. Staff: covers all deep dives (or new ones), surfaces issues unprompted, discusses performance tuning.
