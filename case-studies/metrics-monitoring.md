# Design Metrics Monitoring & Alerting (Datadog / Prometheus)

Source: Hello Interview outline (premium); answers are independent · Patterns: scaling writes, scaling reads · Difficulty: hard

## Scope
- **FR:** ingest metrics from services; query and visualize on dashboards; define alert rules with thresholds; notify when alerts fire. Below the line: logs, traces, anomaly detection ML.
- **NFR:** ingest millions of data points/s (e.g., 100k hosts × 1k series / 10 s ≈ 10M points/s); dashboard queries over weeks < 1–2 s; alerts fire within < 1 minute; highly available during spikes and incidents (precisely when you need it); handle high cardinality safely.

## Crux
A write-heavy time-series ingestion path, a read path for range aggregations over long windows, and an alerting path that must be faster and more reliable than either.

## Skeleton
Agents (push, batched, compressed) → ingestion gateways → Kafka (partition by series hash) → TSDB writers (LSM, Gorilla compression, time-partitioned blocks) + rollup jobs (1 m, 1 h) → query service (fan-out to shards, merge) → dashboards. Alert evaluator reads from the stream/TSDB → notification service.

## Deep dives

**1. Low-latency dashboards over weeks**
Downsampling/rollups (raw for 1–2 days, 1-minute for weeks, 1-hour for months) chosen by query range; pre-aggregate common group-bys; cache query results for dashboards (many users view the same panels; align to step boundaries); shard by series, scatter-gather per query.

**2. Alerts in < 1 minute**
Don't poll the TSDB with every rule every minute at scale — evaluate **in the stream** (Flink keyed by rule/series, sliding windows) for threshold rules, or partition rule evaluation across workers that query recent data from an in-memory hot tier. Alert state machine (pending → firing → resolved) with `for:` durations, dedupe and grouping before notifying.

**3. High availability during spikes/failures**
Agents buffer locally and retry; Kafka absorbs bursts; ingestion load-sheds low-priority metrics before alert-critical ones; replicate TSDB shards; **run alerting independent of the dashboard path** (separate pipeline, cross-region "watch the watchers" meta-monitoring). Monitoring must survive the incident it's monitoring.

**4. Cardinality explosion**
Series = metric × unique tag combinations held in memory indexes; `user_id`/`request_id` as tags → billions of series → OOM. Enforce per-tenant series limits, reject/drop high-cardinality tags at ingestion, move them to fields or logs/traces, cardinality dashboards for customers, sample.

## Staff+ signals
- Pull (Prometheus scraping) vs push (agents) tradeoff stated and chosen.
- Percentiles: store mergeable sketches (DDSketch/t-digest/HDR histograms), never average p99s.
- Multi-tenancy isolation (noisy tenant can't starve others), retention tiers and cost.
- Clock skew and late data handling in windows.

## Traps
Generic SQL DB for raw points; computing long-range queries from raw data; alerting through the same overloaded query path; unbounded tags.
