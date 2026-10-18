# Core Concepts

Technology-agnostic vocabulary. Interviewers assume you know these and probe when you invoke them. At staff+, mention the concept in one line and spend words only on the tradeoff that matters for *this* design.

## Contents
1. Networking · 2. API design · 3. Data modeling · 4. Indexing · 5. Caching · 6. Sharding · 7. Consistent hashing · 8. CAP, PACELC, consistency models · 9. Failure handling

---

## 1. Networking

**Layers that matter:** L3 IP (best-effort packets) · L4 TCP (ordered, reliable, connection setup cost) / UDP (fire-and-forget; video calls, games, DNS, telemetry where late = useless) · L7 HTTP, WebSocket, gRPC, WebRTC.

**Protocol choice**

| Need | Pick | Notes |
|---|---|---|
| Default request/response | HTTP/REST over TCP | 90% of cases |
| Internal, latency-critical | gRPC (HTTP/2, protobuf) | not browser-native; REST outside, gRPC inside |
| Server → client push | SSE | one HTTP response kept open; auto-reconnect with `Last-Event-ID`; plays well with HTTP infra; client writes via normal POSTs |
| Frequent two-way messaging | WebSocket | stateful; needs L4 LB or sticky routing, connection registry, heartbeats, reconnect logic |
| Peer audio/video | WebRTC | STUN/TURN for NAT traversal; signaling server still needed |
| "Real-time" but seconds is fine | (Long) polling | simplest; start here unless requirements say otherwise |

Common mistake: WebSockets because the prompt says "real-time". Ask whether the client actually sends frequently.

**Load balancing:** L7 routes on content (paths, headers), terminates TLS, multiplexes connections downstream. L4 forwards TCP connections; use for long-lived WebSocket connections. Client-side LB (service discovery + smart clients, e.g., Redis Cluster, gRPC) removes a hop for internal traffic. Algorithms: round robin, least connections, consistent hashing (stickiness for stateful backends).

**Geography:** physics sets a floor (~80 ms NYC↔London). Global low latency = CDN for static/cacheable content + regional deployments with data partitioned or replicated by region.

**Failure handling at the network layer:** timeouts on every call; retries with exponential backoff **and jitter**; retry only idempotent operations (or use idempotency keys); circuit breakers to stop hammering a failing dependency and fail fast; bulkheads to isolate pools. Retry storms and thundering herds are classic staff-level probes.

## 2. API design

- **REST:** resources are plural nouns; GET (safe, idempotent), PUT/DELETE (idempotent), POST (not), PATCH (partial). Path for identity (`/events/{id}`), query for filters/sort/pagination, body for payloads.
- **GraphQL:** one endpoint, client picks fields; solves over/under-fetching for many client types. Costs: caching, N+1 resolvers (DataLoader), query complexity limits.
- **RPC/gRPC:** action-oriented, typed contracts, streaming; internal services.
- **Pagination:** offset (simple; breaks under inserts, slow deep pages) vs **cursor** (stable for live feeds; encode last sort key + id).
- **Idempotency keys:** client-generated key on unsafe operations (payments, orders); server stores key → result, returns the original result on retry.
- **Errors:** consistent envelope, correct status codes (429 with `Retry-After`/rate-limit headers).
- **Versioning:** URL (`/v1`) is simplest; header-based is cleaner but less visible.
- **Security:** authenticate every request; derive the actor from the token. JWT for user sessions (stateless; revocation needs short TTL or denylist), API keys for service/developer access; RBAC/ABAC for authorization; rate limiting per identity.

## 3. Data modeling

Start from **access patterns** (which queries, how often, what latency), then entities, keys and relationships.

| Model | Shines for | Watch out |
|---|---|---|
| Relational (Postgres/MySQL) | default for product designs; joins, transactions, constraints, many indexes | single-writer ceiling, sharding is manual |
| Document (Mongo, DynamoDB documents) | aggregate-shaped reads, evolving schemas | cross-document consistency, duplicated data |
| Key-value (Redis, DynamoDB) | known-key lookups at huge scale | no ad-hoc queries |
| Wide-column (Cassandra, Bigtable, Scylla) | write-heavy, time-ordered per partition (messages, events) | query-first modeling, no joins, tombstones |
| Graph (Neo4j) | deep multi-hop traversals | rarely needed; social graphs usually live in KV/SQL adjacency tables |

**Normalize by default**, denormalize specific hot read paths once you've identified them (store `username` on the order, `like_count` on the post). Denormalization buys read speed with write fan-out and drift risk.

**NoSQL modeling**: partition key = the thing you always query by and that spreads load; sort key = the order you read in (`user_id` PK, `created_at` SK for "posts by user, newest first"). Queries you didn't model for become scans or secondary indexes.

Avoid the SQL-vs-NoSQL debate. Say what *this* database gives you: "Postgres — transactions across ticket and booking." Broad claims ("NoSQL for scale", "SQL because relationships") are yellow flags.

## 4. Database indexing

- **B-tree / B+tree**: default. Sorted → equality, range, ORDER BY. Each node ≈ one page; 3–4 page reads for millions of rows.
- **LSM tree**: memtable + WAL → immutable sorted SSTables → compaction. Sequential writes = very high write throughput (Cassandra, RocksDB, DynamoDB-style storage, TSDBs). Reads may touch several SSTables → bloom filters, compaction tuning; write and space amplification tradeoffs.
- **Hash index**: O(1) equality, no ranges. Mostly in-memory stores.
- **Geospatial**: geohash (string prefix = cell; neighbors needed at edges), quadtree (adaptive to density), R-tree (bounding rectangles; shapes; PostGIS GiST). See `technologies.md` → geo.
- **Inverted index**: term → posting list of doc ids; full-text search (Elasticsearch, Postgres GIN/tsvector).
- **Composite indexes**: column order matters — equality columns first, then range/sort (leftmost-prefix rule). `(event_id, status, expires_at)`.
- **Covering index**: includes all selected columns → index-only scan.
- **Costs**: storage, write amplification per index; tiny or write-only tables may not deserve one.
- Secondary indexes in a separate system (ES, PostGIS replica) are fed by CDC and are **eventually consistent** — fine for search, say so.

## 5. Caching

**Where:** client (browser/app), CDN/edge (static + cacheable API responses), in-process (config, feature flags, tiny hot sets; per-instance, no coherence), external (Redis/Memcached; shared, the default for app data).

**Strategies**
- **Cache-aside** (default): read cache → miss → DB → populate with TTL. Write: update DB, then **delete** the key.
- **Read-through**: cache library loads on miss (same semantics, cleaner code).
- **Write-through**: write cache and DB synchronously; consistent, slower writes, caches cold data.
- **Write-behind/back**: write cache, flush async; fast, risks loss — only for counters/metrics you can lose.
- **Write-around**: write DB only; avoids polluting the cache with write-once data.

**Eviction:** LRU (default), LFU (stable popularity), FIFO, TTL (staleness bound).

**Problems interviewers probe**
- **Invalidation**: delete-on-write + TTL safety net; CDC-driven invalidation for multi-writer systems; versioned keys; read-your-writes by routing the writer to the primary briefly.
- **Stampede / thundering herd**: single-flight lock per key (one rebuild, others wait or serve stale), probabilistic early refresh, stale-while-revalidate, TTL jitter, pre-warming known hot keys.
- **Hot keys**: replicate the key across nodes (`key#1..N`, random read suffix), local in-process cache with short TTL, read replicas (clients must be configured to read from them), request coalescing.
- **Cache down**: in-process fallback, circuit breaker to protect the DB, degrade gracefully — never let a cache outage become a DB outage.

Name the **data structure**, not just "cache it": sorted set for leaderboards, hash for objects, set for dedupe, HyperLogLog for unique counts, GEO for locations.

Don't cache data that changes every request; don't cache before you know the hot path.

## 6. Sharding

Partitioning splits data within one database; **sharding** spreads it across machines. Bring it up only once the numbers justify it (TBs, sustained >10k writes/s, or read load replicas can't absorb).

- **Shard key**: high cardinality, even load, matches the dominant query so it hits one shard (user_id for user-centric apps). The cost: queries on other dimensions become scatter-gather.
- **Strategies**: hash (default; even spread, no range queries across keys), range (time/tenant ranges; easy scans, hotspot risk — e.g., time-ordered writes all hit the newest shard), directory/lookup (flexible, extra hop + dependency), geo (locality + residency).
- **Problems**: hot spots (celebrity, Taylor Swift event) → salting/splitting hot keys, dedicated shards; cross-shard transactions → design boundaries to avoid them, else sagas / 2PC; resharding → consistent hashing, virtual shards (many logical shards mapped to fewer physical nodes), online migration with double writes + backfill.
- Modern managed stores (DynamoDB, Cassandra, Spanner, Vitess, Citus, CockroachDB) shard for you; say so and focus on the key choice.

## 7. Consistent hashing

`hash(key) % N` remaps ~all keys when N changes. A hash ring maps nodes and keys onto a circle; a key belongs to the next node clockwise, so adding/removing a node moves ~1/N of keys. **Virtual nodes** (many points per physical node) even out the distribution and spread a failed node's load across many peers.

Distinguish: virtual nodes fix *structural* imbalance (key distribution); replication and key salting fix *workload* imbalance (hot keys). In practice replication (N consecutive ring nodes) means node failure needs no data movement; movement happens only on planned membership changes. Redis Cluster uses 16,384 fixed hash slots instead — simpler, needs explicit slot migration.

Only go deep when designing the distributed system itself (distributed cache, KV store, message broker, chat server routing). Otherwise: "DynamoDB/Cassandra handle partitioning via consistent hashing."

## 8. CAP, PACELC and consistency models

Partitions happen, so CAP reduces to: **during a partition, refuse (C) or serve possibly stale data (A)?** Ask: *would a stale read be catastrophic?*
- Consistency: booking seats, inventory, money, matching (one driver → one ride), auctions' winning bid, unique usernames.
- Availability: feeds, profiles, reviews, search, view counts, analytics.
- Mixed per feature is the norm and the staff answer: "strong for booking, available for browse."

**PACELC**: else (no partition), trade **latency vs consistency** — strong consistency costs coordination round trips even on a good day.

Spectrum: strong/linearizable → sequential → causal (replies after posts) → read-your-writes / monotonic reads (session guarantees) → eventual. Tools: single leader + synchronous replication, quorum (R + W > N), consensus (Raft/Paxos/ZAB) for metadata, CRDTs for convergent multi-writer data.

CAP-consistency ≠ ACID-consistency (the latter is "invariants hold").

## 9. Failure handling and correctness toolkit

- **Idempotency** everywhere a retry can happen: idempotency keys, conditional writes, dedupe tables, upserts.
- **At-least-once + idempotent consumer** is the practical "exactly-once". True exactly-once exists only inside a system boundary (Kafka transactions + Flink checkpoints).
- **Outbox pattern**: write the business row and an outbox row in one transaction; a relay publishes to Kafka. Avoids dual-write inconsistency. CDC is the log-based variant.
- **Backpressure**: bounded queues, 429/503 with `Retry-After`, load shedding by priority. A queue without backpressure only hides insufficient capacity.
- **Timeouts < caller's timeout** (deadline propagation).
- **Fencing tokens** with leases so a paused ex-leader can't write.
- **Dead-letter queues** for poison messages, with alerting and replay tooling.
- **Reconciliation jobs** comparing source of truth against derived state (payments vs PSP, clicks vs aggregates).
