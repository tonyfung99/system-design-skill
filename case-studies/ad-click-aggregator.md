# Design an Ad Click Aggregator

Source: Hello Interview breakdown (full) · Patterns: scaling writes, stream processing · Difficulty: hard

## Scope
- **FR:** user clicks an ad and is redirected to the advertiser; advertisers query click metrics over time at ≥ 1-minute granularity. Below the line: ad targeting, serving, billing, fraud ML.
- **NFR:** peak 10k clicks/s; sub-second analytics queries; **no lost clicks**; as real-time as possible; **idempotent** (count each click once).

## Crux
Separating the write-optimized ingestion path from the read-optimized query path, with pre-aggregation in between — and proving correctness (no loss, no double count) despite streaming shortcuts.

## Skeleton
Click → Click Processor (`GET /click/{adId}?impressionId=…` → 302 to advertiser URL — server-side redirect so we always record it) → stream (Kafka/Kinesis, partition by adId) → Flink (1-minute tumbling windows by adId) → OLAP store (ClickHouse/Druid/Pinot or Cassandra with time-bucketed rows) → Query service for advertisers. Raw events also land in S3 (data lake).

Batch alternative (◐): write raw clicks to a DB/S3 and aggregate every N minutes with Spark — simpler, but minutes of latency. Stream (✓) gives near-real-time.

## Deep dives

**1. 10k clicks/s and hot ads**
Click processor stateless behind an LB; stream sharded by adId (Kinesis ~1k records/s/shard); one Flink task per partition; OLAP sharded by advertiserId. **Hot ad** (Nike × LeBron) → salt the partition key for known-hot ads (`adId:0..N`, chosen by spend/previous volume) and merge sub-aggregates downstream.

**2. No lost clicks**
Stream replication + retention (e.g., 7 days) → if Flink dies, replay from offsets. Flink checkpointing exists, but with 1-minute windows on a replayable log you lose at most a minute and can recompute — questioning whether checkpointing is necessary is a seniority signal. **Reconciliation:** periodic batch job (Spark over raw S3 events) recomputes authoritative counts and corrects the OLAP store — fast path for latency, slow path for correctness (lambda-style).

**3. Abuse & idempotency**
✗ Dedupe on (userId, adId) — needs login, and blocks legitimate repeat clicks. ✓ **Signed impression ID** per ad render: Ad Placement Service issues a unique impressionId (signed so it can't be forged); the click processor verifies the signature and checks a dedupe cache (Redis, `SETNX` with TTL; ~1.6 GB/day if 100M impressions) before emitting. Dedupe must happen **before** the stream so duplicates across window boundaries aren't double counted.

**4. Low-latency queries**
Already pre-aggregated per minute; add rollup tables (hourly/daily) for long ranges via scheduled jobs or extra Flink windows; queries hit the coarsest sufficient granularity.

## Staff+ signals
- Clear batch vs streaming tradeoff discussion and why reconciliation makes the streaming shortcut safe.
- Exactly-once semantics: Kafka transactions + Flink two-phase-commit sinks vs at-least-once + idempotent upserts keyed by (adId, minute).
- Late events and watermarks; click-time vs processing-time windows; billing implications.
- Database choice argued from query shape (group-by over time per advertiser) and data volume.
- Ideally teaches the interviewer something from experience.

## Level bar
Mid: pre-aggregation, at least batch processing; responds to probes on idempotency/DB choice. Senior: fast HLD, then scaling, latency, fault tolerance; contrasts batch and real-time. Staff: weighs batch vs real-time in detail, fault-tolerant design, storage tradeoffs, deep on chosen areas.
