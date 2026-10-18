# Patterns

Recognizing the pattern tells you what's interesting, the known failure modes, and the standard escalation path — so you don't reverse-engineer them live. Patterns combine (YouTube = large blobs + long-running tasks + multi-step + realtime progress). Always start at the simplest rung and climb only when a stated requirement forces it.

Each section: **signals → ladder (✗ naive / ◐ good / ✓ great) → the deep-dive questions interviewers ask → when not to use.**

## Contents
1. Scaling reads · 2. Scaling writes · 3. Dealing with contention · 4. Real-time updates · 5. Long-running tasks · 6. Multi-step processes · 7. Large blobs · 8. Proximity search

---

## 1. Scaling reads

**Signals:** read:write 10:1 to 100:1+ (feeds, product pages, URL redirects, event pages); one item read by millions.

**Ladder**
1. **Optimize inside the DB** — indexes for the actual queries (composite/covering), bigger box (vertical scaling is underrated), denormalize hot read paths, materialized views / precomputed aggregates.
2. **Scale the DB horizontally** — read replicas (async → replication lag; route read-your-writes to primary or pin the session), then shard if the dataset or write load demands it.
3. **External caches** — cache-aside Redis for app data; CDN/edge for static and cacheable (non-personalized) responses; in-process caches for tiny hot sets.

**Deep-dive questions**
- *Queries slow as data grows?* Check plans; index; partition by time; archive cold data; precompute.
- *Millions of concurrent reads of one cached key?* A single cache node is the new hot spot: replicate the key under N suffixes or use replicated (not sharded) cache instances, add short-TTL in-process caching, CDN for public content, request coalescing.
- *Expired hot entry → stampede?* Single-flight/lease per key, serve-stale-while-revalidate, probabilistic early refresh, TTL jitter, pre-warm.
- *Writes must be visible immediately?* Delete-on-write + TTL; versioned keys; CDC-driven invalidation; read-your-writes routing; accept that cross-region caches are eventually consistent and decide where that's OK.

**Not when:** write-heavy or uniformly cold data; caching per-request-unique results; adding Redis before checking an index would fix it.

## 2. Scaling writes

**Signals:** counters, events, telemetry, likes, location pings, clicks; >10k writes/s; bursts.

**Strategies (combine)**
1. **Vertical + right storage engine** — LSM/append-only stores (Cassandra, TSDB, Kafka) for firehoses; drop unnecessary indexes; don't use a DB as a queue.
2. **Sharding & partitioning** — horizontal by a key that spreads writes *and* keeps related data together; vertical partitioning (split hot write columns like `like_count` away from cold profile data).
3. **Burst handling** — write queues/streams to absorb spikes (only if the caller can tolerate async), load shedding by priority (drop analytics before purchases), backpressure.
4. **Batching & hierarchical aggregation** — aggregate in memory/stream (per-minute counts in Flink) and write summaries; batch likes per 30 s; tree aggregation (local → regional → global) for live counts; milestone writes (update a search index at like counts 1, 2, 4, 8…).

**Deep-dive questions**
- *Resharding?* Over-provision logical shards mapped to physical nodes; consistent hashing; online migration: dual-write → backfill → verify → cut reads → stop old writes.
- *A key too hot for one shard?* Split all keys into k sub-keys (writes spread, reads gather) or split only detected hot keys dynamically (`adId:0..N` when rate > threshold; readers must know the fan-out — publish the hot-key list).

**Not when:** modest write rates — one primary handles ~10k TPS; queues on synchronous low-latency paths.

## 3. Dealing with contention

**Signals:** last ticket, last unit in a flash sale, highest bid, one driver ↔ one ride, matching two users, account balance, unique username.

**The race:** read-check-write without atomicity → two buyers both see 1 seat → oversold.

**Ladder (prefer the lowest that works)**
1. **Atomic conditional write in one DB** — `UPDATE seats SET status='held', holder=? WHERE id=? AND status='available'` → check rows affected. Or `UPDATE inventory SET qty=qty-1 WHERE id=? AND qty>0`. Often all you need.
2. **Pessimistic locking** — `SELECT … FOR UPDATE` inside a *short* transaction. Lock in a consistent order to avoid deadlocks; never hold across user think-time.
3. **Optimistic concurrency (OCC)** — version column, `WHERE version = ?`, retry on conflict. Great when conflicts are rare; terrible when everyone hits the same row. ABA risk → monotonically increasing versions, not value comparison.
4. **Isolation levels** — SERIALIZABLE (or SSI) when invariants span rows (orders + inventory in one Postgres txn); costs aborts/retries.
5. **Reservations with expiry** — status + `expires_at` checked inline ("available OR reserved-but-expired"), so a lagging sweeper never blocks sales; or a Redis key `SET NX EX`. Reservation ≠ sale: the final commit still uses step 1. With **counter-style inventory** (hotel room-nights, flash-sale units), prefer **hold rows** (`hold_id, qty, expires_at, status`) and compute availability inside the transaction as `total − booked − SUM(active, unexpired holds)`: expiry is then implicit and a stalled sweeper only cleans up. If you keep a bare `held` counter instead, a sweeper must release it — say that sweeper lag can only *undersell*, never double-book, and alert on the age of the oldest unreleased expired hold, because a stall silently strands inventory. Confirm with `WHERE status='HELD' AND expires_at > now()`; give hold creation an idempotency key. Multi-row holds (a 3-night stay) change all rows in one transaction and roll back unless every row matched; take locks in an explicit fixed order (`SELECT … ORDER BY night FOR UPDATE`) — don't rely on an UPDATE's scan order — and keep a deadlock-retry as backstop.
6. **Distributed locks** — Redis (single node or Redlock) as an *efficiency* tool; ZooKeeper/etcd with fencing tokens when a stale holder would corrupt data. Enforce the invariant where the data lives.
7. **Serialize the hot item** — route all writes for one key to a single partition/actor/queue consumer (auction per item, flash-sale item); in-memory counter with durable log; pre-split inventory into buckets/tokens.

**Deep-dive questions**
- *Deadlocks with pessimistic locks?* Global lock ordering, short txns, lock timeouts, detect-and-retry.
- *Everyone wants the same resource (flash sale)?* Admission control (virtual waiting room), pre-allocated tokens per unit, single-writer per item, decrement in Redis with Lua then persist, reject fast when sold out.
- *Cross-database atomicity?* Avoid by co-locating; else saga with compensations or 2PC (rarely worth it).

**Not when:** contention doesn't exist (likes, views — use counters/aggregation, not locks). Splitting data into multiple stores means re-solving problems the database already solved — interviewers probe whether you know what you gave up.

## 4. Real-time updates

Two hops: **client ↔ server** and **source → the server holding that client's connection**.

**Hop 1 — protocol ladder**
- Simple polling (start here if seconds of delay is fine) → long polling → **SSE** (one-way push, HTTP-friendly, built-in reconnect + `Last-Event-ID`) → **WebSocket** (bidirectional, stateful, L4 LB) → WebRTC (peer-to-peer media).

**Hop 2 — server-side fan-in/fan-out**
- **Pull/poll from a store** — servers poll DB/cache; simple, latency = interval.
- **Consistent-hash routing** — each user/doc owned by one server (ring in ZooKeeper/etcd); senders route directly. Good when processing is stateful and heavy (collaborative editing, game rooms). Rebalancing moves connections.
- **Pub/sub** — Redis Pub/Sub / Kafka / NATS; connection servers subscribe to channels for their connected users/topics. Decouples senders; at-most-once in Redis Pub/Sub → pair with durable inbox + sequence numbers.

**Deep-dive questions**
- *Connection failures?* Heartbeats (app-level ping/pong every 10–30 s), client reconnect with backoff+jitter, resume from last sequence/event id, durable inbox for catch-up.
- *One update → millions of subscribers (celebrity, live event)?* Hierarchical fan-out, co-locate viewers of the same stream on the same servers, sample/aggregate (show a representative subset of comments), or switch to pull: periodic snapshots via CDN.
- *Ordering across servers?* Per-conversation sequence numbers (assigned by a single owner or atomic INCR); clients reorder/de-dupe; accept server-receive-time ordering where users tolerate it.
- *Deploys with millions of sockets?* Drain gradually, reconnect storms throttled with jitter, connection servers kept thin and stateless beyond the socket.

**Not when:** "real-time" means "within a minute" → polling. Don't use WebSockets for read-mostly push.

## 5. Managing long-running tasks

**Signals:** anything > a few seconds — transcoding, report/PDF generation, code execution, bulk import, campaign sends, LLM generation.

**Shape:** API validates → writes job row (status=queued) → enqueues → returns `jobId` (202). Workers pull, execute, update status; client polls/SSE/webhook for completion.

**What you gain:** fast responses, independent scaling of workers (GPU vs CPU pools), fault isolation, buffering. **What you lose:** simplicity, immediate feedback, and you now own job state, retries, and duplicates. For short jobs (< ~1–2 s), stay synchronous — clearer backpressure and UX.

**Implementation:** queue = SQS (visibility timeout, built-in retries/DLQ, delay queues), RabbitMQ, Redis streams/lists, or Kafka (ordering, replay; DIY retries). Workers: autoscaled on queue depth; containers/serverless.

**Deep dives**
- *Worker dies mid-job?* Visibility timeout / lease + heartbeat; job re-delivered; idempotent work (dedupe on jobId, deterministic outputs, conditional writes).
- *Repeated failures?* Max attempts with exponential backoff → DLQ + alert; distinguish transient vs permanent errors.
- *Duplicate work?* Idempotency key on submit; claim with conditional update (`status queued→running WHERE status='queued'`).
- *Backpressure?* Bound queue depth; reject/429 at admission; autoscale; priority queues.
- *Mixed workloads?* Separate queues/pools by size or priority so big jobs don't starve small ones.
- *Dependencies between jobs?* DAG orchestration → multi-step pattern / workflow engine.

## 6. Multi-step processes (sagas, workflows, durable execution)

**Signals:** order fulfillment, payments, onboarding, ride matching with timeouts, video pipeline DAGs, agent pipelines — many flaky services, human/external waits, compensation.

**Ladder**
1. **Single-server orchestration** — fine until the server crashes mid-flow; state in memory is lost.
2. **Persisted state machine** — status column + queues + timers + cron; works, but you're hand-building a workflow engine; business and failure logic interleave.
3. **Saga** — sequence of local transactions with compensating actions (refund, release inventory). **Choreography** (services react to events; loose coupling, hard to see the whole flow) vs **orchestration** (central coordinator; explicit, testable).
4. **Durable execution / workflow engines** — Temporal, Cadence, AWS Step Functions, Azure Durable Functions. Workflow code is replayed deterministically from an event history; activities (side effects) are retried; timers and signals are durable.

**Deep dives**
- *Orchestrator crashes halfway?* With durable execution, history replay resumes at the last completed step; activities must be idempotent.
- *Updating a workflow with in-flight executions?* Versioning (patch markers / version gates), run old and new side by side, migrate long-lived ones explicitly.
- *State size growth?* Continue-as-new, keep payloads as pointers (S3 keys), not blobs.
- *External events (webhook, driver accepts)?* Signals to the running workflow; correlate by id; timeouts as durable timers.
- *Step must run exactly once?* Impossible end-to-end; at-least-once + idempotency key at the external system (e.g., PSP idempotency key).

**Not when:** a single async job (just a queue), or high-volume stream processing (Kafka/Flink). Some interviewers want you to *design* the orchestration — don't drop "Temporal" in as a magic box without explaining what it does.

## 7. Handling large blobs

**Signals:** images, video, documents, backups — anything over ~10 MB that doesn't need SQL queries.

**Ladder**
- ✗ Store in DB / stream bytes through app servers (bandwidth, memory, timeouts, double transfer).
- ◐ Blob store, but proxy through servers.
- ✓ **Presigned URLs**: server authorizes and returns a short-lived, scoped upload URL; client uploads directly to S3/GCS; downloads via CDN with signed URLs/cookies. Metadata (owner, status, size, checksum, storage key) in the DB.

**Resumable / large uploads:** chunk (5–100 MB parts) with S3 multipart (GCS resumable, Azure block blobs); parallel parts; client tracks ETags; resume by listing uploaded parts. Content-defined chunking + per-chunk fingerprints enable dedupe and delta sync.

**State synchronization:** record `pending` before upload; storage event notification (`ObjectCreated`) flips to `uploaded`; verify server-side (ListParts/HEAD, checksums) rather than trusting the client; sweep orphaned pending uploads and incomplete multipart uploads (lifecycle rules).

**Deep dives**
- *Fails at 99%?* Resume missing parts only.
- *Abuse?* Presign with content-length range, content-type, short expiry, per-user quotas; virus/content scanning post-upload before publishing.
- *Metadata?* Separate from bytes; consistent via status transitions + events.
- *Fast downloads?* CDN, range requests, regional replication, adaptive bitrate for video.

## 8. Proximity search

**Signals:** "near me" — drivers, restaurants, delivery centers, friends.

- Small set (< ~100k, slow-changing): load into memory and compute distance — no index needed.
- **Encoded cells** (geohash, S2, H3) on an ordinary index, or **Redis GEO** (geohash in a sorted set): cheap writes → best for **moving points** (drivers update every few seconds). Always query the cell plus neighbors (edge problem).
- **Spatial trees** — quadtree (adapts to density), R-tree / PostGIS GiST (shapes, polygons, containment), BKD in Elasticsearch (great reads, write-once segments).
- The index only yields candidates; **always post-filter** with exact distance/ETA, then rank.
- Global queries are rare; partition by region/city.
