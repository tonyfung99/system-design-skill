# Key Technologies

Know at least one option per category deeply enough to explain *how it works* and *where it breaks*. Depth probed scales with level: a mid-level candidate can call Elasticsearch "a search index"; a senior who can't explain the inverted index is a yellow flag; a staff candidate should know operational failure modes and when *not* to use it. Breadth before depth.

## Contents
Core DB choice · Postgres · DynamoDB · Cassandra · Redis · Kafka · Queues (SQS) · Elasticsearch · Flink · Temporal · ZooKeeper/etcd · API gateway · Load balancer · CDN · Blob storage · Time-series DBs · Vector DBs · Geospatial indexes · Probabilistic structures · CDC

---

## Picking the core database
Pick one you know; don't stage a SQL-vs-NoSQL debate. Product design → default **Postgres**. Infra/high-write → **DynamoDB or Cassandra**. Justify with a property: "Postgres for the multi-row transaction", "Cassandra for append-heavy, partition-by-chat message history".

## PostgreSQL
- **Reads:** B-tree by default; GIN (full-text `tsvector`, JSONB, arrays), GiST (PostGIS geo, ranges), BRIN (huge append-only time tables), partial and covering indexes. `EXPLAIN ANALYZE`.
- **Writes:** single primary; ~10k–50k simple TPS on good hardware. Speedups: batch inserts/COPY, fewer indexes, partitioning (declarative by time/tenant), unlogged tables for scratch, async commit where loss is tolerable.
- **Replication:** streaming (async default; sync for zero-loss failover at latency cost). Read replicas → replication lag. HA via Patroni/managed failover.
- **Consistency:** ACID; Read Committed default; Repeatable Read; Serializable (SSI) for cross-row invariants; `SELECT … FOR UPDATE`, `SKIP LOCKED` (job queues), advisory locks.
- **Scale-out:** Citus, Vitess-equivalents, app-level sharding.
- **Alternatives when:** extreme write throughput (Cassandra), global multi-region writes (Spanner/CockroachDB), full-text relevance at scale (ES), flexible single-key access at huge scale (DynamoDB).

## DynamoDB
- **Model:** partition key (+ optional sort key). Items ≤ 400 KB. `Query` within a partition by sort-key range; `Scan` = avoid.
- **Indexes:** GSI (different PK; eventually consistent only; own capacity), LSI (same PK, different sort; strongly consistent allowed; created at table creation).
- **Consistency:** per-request — eventual (default, 0.5 RCU/4 KB) or strong (`ConsistentRead`, 1 RCU, leader read, not on GSIs). ACID transactions up to 100 items across tables. Conditional writes for OCC.
- **Scale/HA:** auto-partitioning (consistent hashing), 3-AZ replication, adaptive capacity for hot partitions (still ~1k WCU/3k RCU per partition key), Global Tables (multi-region, last-writer-wins).
- **Extras:** DAX (microsecond cache), Streams (CDC → Lambda/ES), TTL for expiry.
- **Limits:** cost at very high throughput, ad-hoc queries/joins, GSI sprawl means you wanted SQL, AWS lock-in (some interviewers want vendor-neutral).

## Cassandra (and Scylla)
- **Model:** keyspace → table; primary key = partition key (placement) + clustering columns (on-disk sort order within the partition). Model **one table per query**; denormalize.
- **Architecture:** leaderless ring, consistent hashing with vnodes, replication factor N (NetworkTopologyStrategy across DCs), gossip for membership, hinted handoff + read repair + anti-entropy repair.
- **Storage:** commit log + memtable → SSTables (LSM), compaction strategies (size-tiered, leveled, time-window for TTL'd time series). Very fast writes.
- **Consistency:** tunable per query (ONE, QUORUM, LOCAL_QUORUM, ALL); R + W > N for read-your-writes. Lightweight transactions (Paxos) for single-partition compare-and-set — slow, use sparingly.
- **Good for:** write-heavy, time-ordered per key (chat messages partitioned by `chat_id`, clustered by `message_id`/time — Discord's model), availability-first, multi-DC.
- **Bad for:** strict consistency, joins, ad-hoc aggregations, huge partitions (bucket by time), heavy deletes (tombstones).

## Redis
- **What:** single-threaded, in-memory data-structure server; ~100k+ ops/s per node, sub-ms. Strings, hashes, lists, sets, sorted sets, bitmaps, HyperLogLog, geo, streams, Bloom/CMS (Redis Stack).
- **Topology:** single node → primary + async replica (failover may lose acknowledged writes) → Cluster with 16,384 hash slots; clients cache the slot map and follow `MOVED`. No cross-node queries: multi-key ops only within a slot → use **hash tags** `{user:123}:posts`.
- **Uses:** cache (cache-aside, TTL, LRU/LFU eviction); **distributed lock** (`SET key token NX EX ttl`, release via Lua compare-and-delete; Redlock is debated — add fencing tokens or enforce invariants in the DB); **leaderboards** (sorted sets, `ZINCRBY`, `ZREVRANGE`); **rate limiting** (INCR+EXPIRE fixed window, Lua token bucket, sorted-set sliding log); **proximity** (`GEOADD`/`GEOSEARCH`); **pub/sub** (fire-and-forget, at-most-once; good for chat fan-out across servers); **streams** (consumer groups, durable-ish work queues); atomic counters/ID generation (`INCR`).
- **Hot keys:** in-process cache, key copies with random suffix, replica reads (`READONLY`), but replicas don't help write-hot keys.
- **Don't:** use as system of record, store working sets that don't fit RAM economically, expect query flexibility, or replace Kafka for durable replayable streams.

## Kafka
- **Model:** topics → partitions (ordered, append-only logs) replicated across brokers (leader + ISR). Producers choose partition by key hash; consumers in a **consumer group** split partitions (≤ one consumer per partition per group); offsets committed per partition; retention by time/size or log compaction (latest value per key).
- **Queue vs stream:** one group consuming = work queue; many groups = pub/sub fan-out with replay.
- **Scale:** ~1M msgs/s & TBs per broker (hand-wavy). Scale via partitions (parallelism cap) and brokers. Keep messages small (< 1 MB) — put blobs in S3 and send pointers.
- **Hot partitions:** no key (sticky/round-robin) if order doesn't matter, salt hot keys (`adId:k`), compound keys, backpressure.
- **Durability:** `acks=all`, `min.insync.replicas=2`, RF=3; idempotent producer; transactions for read-process-write exactly-once inside Kafka.
- **Consumer errors:** no built-in retry — retry topics with backoff + DLQ topic (or use SQS if you want this for free). Commit offsets **after** processing (at-least-once) + idempotent sinks.
- **Use for:** async decoupling, ordering per key, event streams consumed by many (analytics, search indexing, CDC), buffering bursts, durable replay. **Not** for per-user topics at billions scale.

## Simple queues (SQS, RabbitMQ)
SQS: at-least-once, visibility timeout (lease), built-in redrive to DLQ after N receives, delay queues (≤ 15 min), FIFO queues with dedupe ids and message groups (per-group ordering). Great for job queues and exponential backoff via `ChangeMessageVisibility`. RabbitMQ: routing exchanges, priorities, acks. Choose over Kafka when you need per-message retry/visibility semantics more than replay/throughput.

## Elasticsearch / OpenSearch
- **Model:** index → shards (Lucene indexes) with replicas; documents with mappings (`text` analyzed vs `keyword` exact). Analyzers: tokenize, lowercase, stem, synonyms. Inverted index for terms; doc values (columnar) for sort/aggregation; BKD trees for numbers/geo.
- **Write path:** refresh (default ~1 s) makes new segments searchable → near-real-time; segments are immutable; updates = delete + reinsert; merges compact.
- **Query path:** coordinating node scatters to shards, gathers, merges top-K (query then fetch). Pagination: `search_after` (+ point-in-time) instead of deep from/size.
- **Features:** relevance (BM25), fuzzy matching (edit distance), geo queries, aggregations, request/filter caches.
- **Rules:** not your source of truth (sync via CDC from Postgres/DynamoDB); read-heavy — high-churn fields (like counts) cause write amplification → batch or update on milestones; denormalize; it's eventually consistent; for < ~100k slow-changing docs, a DB query or in-memory index is enough.
- **Lessons to reuse:** immutability enables caching/compression; separate query coordination from storage; index structures follow access patterns.

## Flink (stream processing)
- **Concepts:** sources/sinks, keyed streams (`keyBy` → partitioned state), operators, **windows** (tumbling, sliding, session), **event time + watermarks** (bounded out-of-orderness; allowed lateness), state backends (heap, RocksDB for large state).
- **Fault tolerance:** periodic distributed checkpoints (Chandy-Lamport barriers) to S3; on failure restore state + rewind source offsets → exactly-once state; end-to-end exactly-once needs transactional/idempotent sinks.
- **Architecture:** JobManager (scheduling, checkpoints) + TaskManagers (slots, parallelism).
- **Use for:** real-time aggregation (ad clicks per minute, top-K views), fraud/anomaly detection, enrichment, CEP. Spark Structured Streaming = micro-batch alternative; Kafka Streams = library alternative.
- **Judgment:** with 1-minute windows and replayable input, aggressive checkpointing may be unnecessary — replay from the log.

## Temporal (durable execution)
Workflow code (deterministic) + Activities (side effects, retried). The server persists an event history per workflow; workers replay it to rebuild state after crashes; timers, signals, queries are durable. Task queues route work to worker pools; history shards scale the cluster.
- **Use when** you're hand-building a state machine: multi-step dependent work, long waits (driver offer timeouts, human approval), partial-failure recovery, compensation.
- **Limits:** not a stream processor; activities still need idempotency against external systems; workflow code can't do nondeterministic I/O; some interviewers want *you* to design the orchestration.

## ZooKeeper / etcd (coordination)
- Small, strongly consistent (ZAB / Raft), replicated metadata stores. ZooKeeper: znodes (persistent, ephemeral — vanish with the session, sequential), watches. etcd: KV with leases and watches.
- **Uses:** service discovery, config with push notifications, leader election (lowest ephemeral-sequential node / lease), distributed locks with fencing (zxid / revision as token), shard/ownership maps (which chat server owns which hash range).
- **Not for:** high-volume data, large values, high write rates. Modern stacks often use etcd (Kubernetes), Consul, or managed equivalents; Kafka moved off ZooKeeper (KRaft).

## API gateway
Request validation, authn/z, rate limiting, routing to services, protocol translation, response transformation/aggregation, caching, logging/metrics. Stateless → horizontally scaled; globally via edge PoPs. Options: AWS API Gateway, Kong, Apigee, Envoy, NGINX. Include it for microservice/product designs; skip for a simple client-server system. Draw it, say "routing + auth + rate limiting", move on.

## Load balancer
Wherever multiple identical servers exist. L7 (HTTP-aware routing, TLS termination) by default; **L4 for long-lived WebSocket connections**. Health checks, connection draining for deploys. Don't draw one in front of every service — note "horizontally scaled behind an LB".

## CDN
Edge caches near users (Cloudflare, CloudFront, Akamai, Fastly). Static assets, media, video segments, and cacheable API responses (non-personalized search results, snapshots of live comments, short-URL redirects). TTL + purge APIs; signed URLs/cookies for private content; origin shielding to protect the origin; also DDoS/WAF. Edge compute (Workers/Lambda@Edge) can run redirect or auth logic at the edge.

## Blob storage
S3/GCS/Azure Blob: effectively infinite, 11 nines durable, cheap (~$0.02/GB-month vs ~$0.25+/GB-month for DynamoDB). Pattern: metadata row in DB with pointer; presigned upload/download URLs; multipart upload for large files; event notifications on object creation; lifecycle policies to cold tiers; CDN in front for reads. Don't use as a primary database.

## Time-series databases
InfluxDB, Prometheus, TimescaleDB, VictoriaMetrics, M3. Built from: append-only writes, LSM storage, delta-of-delta + XOR (Gorilla) compression, time-partitioned blocks (drop whole blocks for retention), per-block metadata/bloom filters, downsampling/rollups (raw → 1 min → 1 h), inverted index over tags → series ids.
- **The trap:** cardinality. Series = unique tag combinations held in an in-memory index. user_id/request_id as a tag → billions of series → OOM. High-cardinality values must be fields, not tags. (Billions of videoIds as series makes InfluxDB/Prometheus wrong for Top-K.)
- Real-time OLAP (ClickHouse, Druid, Pinot) is the alternative for high-cardinality group-by analytics.

## Vector databases
Embeddings (128–1536 dims) + similarity (cosine, dot, L2). Approximate nearest neighbor indexes: **HNSW** (default; fast, memory-hungry), IVF (+PQ compression), LSH, Annoy. Filtering: pre-filter vs post-filter vs filtered HNSW — matters when filters are selective. Options: pgvector / Redis / OpenSearch first; Pinecone/Weaviate/Milvus/Qdrant at scale.
- **Interview uses:** semantic search, recommendations (two-stage: ANN candidates → reranker), image similarity, RAG, dedupe, anomaly detection.
- **Gotchas:** not a source of truth; model change = full re-embed (version embeddings); cold start; index build time; 1M × 1536-d ≈ 6 GB raw.

## Geospatial indexes
- **Geohash:** base-32 prefix = cell; prefix queries on a B-tree; query cell + 8 neighbors.
- **S2 / H3:** sphere-aware hierarchical cells (H3 hexagons → uniform neighbor distances; Uber).
- **Quadtree:** recursively split dense regions; good for uneven density (static data, rebuild periodically).
- **R-tree (PostGIS GiST):** bounding boxes; polygons, delivery zones, containment.
- **Redis GEO:** geohash scores in a sorted set; `GEOSEARCH` radius; ideal for high-frequency location updates with TTL'd presence.
- Rule: cells for moving points, trees for shapes, post-filter always.

## Probabilistic data structures
- **Bloom filter:** set membership with false positives, no false negatives. URL-seen in crawlers, "already swiped" filters, skip disk lookups in LSM. Can't delete (counting Bloom can). Size from n and target FP rate (~10 bits/item ≈ 1%).
- **Count-Min Sketch:** frequency estimates that only over-count; heavy hitters/Top-K with a heap; hot-key detection.
- **HyperLogLog:** cardinality (unique visitors) in ~12 KB at ~0.8% error; mergeable across shards/time.
- **Approximate quantiles:** t-digest, DDSketch, HDR/exponential histograms for p50/p99 latency; mergeable across hosts (averaging percentiles is wrong).
- Use only when exactness isn't required — some prompts (Top-K "must be precise") forbid it; say so.

## Change Data Capture (CDC)
Tail the DB's replication log (Debezium on Postgres WAL/MySQL binlog, DynamoDB Streams) and publish row changes to Kafka.
- **Use for:** keeping derived stores in sync (Elasticsearch, caches, warehouses), zero-downtime migrations (backfill + stream + cutover), outbox relay.
- **Not for:** business events with intent ("order shipped → email") — emit explicit domain events (outbox) instead of inferring from row diffs; simple cache invalidation where delete-on-write suffices.
- Expect lag (seconds), ordering per key, schema evolution, and the need to re-snapshot.
