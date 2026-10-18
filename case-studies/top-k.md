# Design Top-K Videos by Views (YouTube Top K / Trending)

Source: full breakdown · Patterns: scaling writes, scaling reads, streaming · Difficulty: hard

## Scope
- **FR:** query top K (≤ 1,000) videos for tumbling windows of 1 hour, 1 day, 1 month, and all-time.
- **NFR:** ≤ 1 min from view to being counted; **precise (no approximation)** unless renegotiated; results in tens of ms; massive view rate; billions of videos.

## Crux
Ingesting ~700k views/s and maintaining exact windowed counts over billions of keys, then answering top-K instantly. Estimation actually matters here.

## The math (do it out loud)
- 70B views/day ÷ ~100k s/day ≈ **700k writes/s** → far past one RDBMS.
- ~1M new videos/day × 10 years ≈ 3.6–4B videos; id+count ≈ 16 B → **~64 GB per full count set** → fits in RAM, but per-window sets multiply it.

## Skeleton
View events → Kafka (partitioned by videoId) → counting layer (Flink or consumers) → per-window aggregates → top-K precomputation → cache → Top-K service (`GET /views/top?window=HOUR&k=100`).

## Deep dives

**1. Cut read queries**
◐ Cache top-K per (window, truncated timestamp) with TTL. ✓ **Precompute** every minute (cron or the stream job) and write to cache; reads never touch the DB.

**2. Massive writes**
Shard the pipeline by videoId end-to-end (Kafka partition → Flink key → storage shard). Pre-aggregate per minute in the stream before writing (1 write per active video per minute instead of per view). Each shard produces a local top-K; a merger combines shard top-Ks (exact, because each video lives on exactly one shard).

**3. Fast top-K queries**
◐ Aggregate at coarser granularity (hour → day → month rollups) so long windows sum fewer rows. ✓ **Maintain a table per window** with an index on count → top-K is an index scan. ✓ (caveats) keep aggregates in Flink state (RocksDB backend), compute top-K natively each minute, write straight to cache — removes the DB but makes state/recovery the hard part.

**4. Sliding windows**
Maintain running totals: each minute add the new minute's counts and subtract the minute that fell out (needs minute-level history retained for the window length — a month of minutes is expensive). Mature move: propose sliding for "last hour" only and tumbling for day/month — matches product expectations.

**5. Approximation (if allowed)**
Count-Min Sketch + min-heap of top 1,000 (Redis CMS + sorted set, or in Flink state). Dramatically less memory; only over-counts. Must renegotiate the "precise" requirement first.

**6. Specialized databases?**
✗ InfluxDB/Prometheus — billions of videoIds as series = cardinality explosion. ◐ TimescaleDB hypertables + continuous aggregates. ◐ Real-time OLAP (Druid/Pinot/ClickHouse) with ingest-time rollups — powerful, but know their operational failure modes (compaction, node loss) if you propose them.

## Staff+ signals
- Let the numbers pick the architecture, narrated ("64 GB → one heap could hold all-time counts; but 700k/s writes → shard by videoId").
- Exactness vs cost as an explicit negotiation; late events and watermarks for window correctness.
- Fault tolerance: replay from Kafka retention vs checkpoints; idempotent minute writes (upsert by (videoId, minute)).
- Hot videos are fine for counting (pre-aggregation absorbs them).

## Level bar
Mid: end-to-end but sub-optimal; some pinch points. Senior: near-optimal, proactive bottleneck removal, tech tradeoffs from experience. Staff: covers deep dives not enumerated, clear judgment across the whole solution space.
