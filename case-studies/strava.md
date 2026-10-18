# Design Activity Tracking (Strava)

Source: outline only; answers are independent · Patterns: scaling writes, real-time updates · Difficulty: medium

## Scope
- **FR:** start/pause/stop/save runs and rides; see live stats (route, distance, time) during the activity; view own and friends' completed activities. Below the line: segments, kudos/comments, training plans.
- **NFR:** works **offline**; 10M concurrent activities; durable once saved; eventual consistency for social views; leaderboard (bonus).

## Crux
The phone is the primary system during the activity — compute locally, sync later. The server only needs to absorb uploads (and optional live sharing) at scale.

## Skeleton
Entities: User, Activity (state, start/end, distance, duration, route polyline/S3 key), Friend. API: `POST /activities` (create), `PATCH /activities/{id}` (state transitions), `POST /activities/{id}/routes` (batched GPS points), `GET /activities?mode=friends&cursor`, `GET /activities/{id}`.

## Deep dives

**1. Offline tracking**
Client records GPS points locally (SQLite), computes distance/time on device, persists durably; on connectivity, uploads in batches with idempotent sequence numbers (resume where it left off). Server accepts out-of-order/late batches.

**2. 10M concurrent activities**
If clients send a point every 5 s: 2M writes/s — avoid. Batch on device (every 30–60 s or on save) → ~200k writes/s or far fewer if only uploading at completion. Store the full route as a compressed blob (S3) + summary row in DB (Cassandra/DynamoDB partitioned by user_id, sorted by time). Ingestion via a stateless upload service; heavy post-processing (map matching, segment detection) async via queue.

**3. Real-time sharing with friends**
Opt-in only. Client sends live position every N seconds to a live-location service (Redis with TTL); friends viewing poll every few seconds or subscribe via SSE. No need for WebSockets.

**4. Leaderboard of top athletes**
Precomputed aggregates (weekly distance per user) updated on activity save; Redis sorted sets per scope (global, region, friends computed on read by intersecting friend ids with scores); recompute periodically from durable store.

## Staff+ signals
- Recognize that on-device computation removes most backend load — the strongest simplification.
- Data model for routes: polyline encoding, compression ratios, map-matching cost.
- Privacy zones (hide home start/end), visibility controls.
- GPS noise and cheating (vehicle speeds) detection in async pipeline before leaderboards.

## Traps
Streaming every GPS point to the server; designing as if the network is reliable; sharding a 10M-user leaderboard prematurely.
