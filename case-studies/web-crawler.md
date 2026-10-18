# Design a Web Crawler

Source: full breakdown · Patterns: long-running tasks, scaling writes · Difficulty: hard

## Scope
- **FR:** crawl the web from seed URLs; extract and store page text for later processing (e.g., LLM training). Below the line: rendering JS-heavy pages, ranking, indexing.
- **NFR:** fault tolerant (resume without losing progress); polite (robots.txt, per-domain rate limits); finish **10B pages in < 5 days**; scalable.

## Crux
A data pipeline problem: a durable frontier, politeness per domain, dedupe at 10B scale, and enough throughput to finish on time.

## Data flow
Seed URLs → frontier queue → fetch (DNS, robots check, rate limit) → store raw HTML (S3) → parse/extract text + links → dedupe → enqueue new URLs → repeat.

## Skeleton
Frontier queue (SQS/Kafka) → Crawler workers (fetch only, network-bound) → S3 raw HTML → Parsing queue → Parser workers (CPU-bound) → S3 text + Metadata DB (URL, status, last crawled, content hash, s3 links) → new URLs back to frontier. Splitting fetch and parse lets each scale independently and lets you re-parse without re-fetching.

## Deep dives

**1. Fault tolerance**
✗ In-memory retry timers. ◐ Kafka with a retry topic and manual backoff (DIY). ✓ **SQS**: visibility timeout as lease; exponential backoff by `ChangeMessageVisibility` based on receive count; DLQ after N attempts. Workers are stateless; progress lives in the queue + metadata DB.

**2. Politeness & robots.txt**
Cache robots.txt per domain (with refresh policy); check allow/disallow before fetch; honor crawl-delay; **per-domain rate limit (~1 req/s)** via Redis (sliding window/token bucket keyed by domain) — if limited, requeue with delay (+ jitter so workers don't sync up). Optionally group URLs by domain in the frontier to reduce lock churn.

**3. Scale to 10B pages in 5 days**
- Throughput: 10B / (5 × 100k s) ≈ 20k pages/s. A network-optimized instance can fetch hundreds–thousands of pages/s (async I/O); budget ~30% for overhead → tens of crawler machines, plus parser fleet autoscaled on queue depth.
- DNS: cache resolutions, multiple resolvers.
- Dedupe URLs: canonicalize; check metadata DB (unique index) or a **Bloom filter** (Redis Bloom) for "seen" — false positives skip a few pages (acceptable).
- Dedupe content: hash page content, index the hash; skip identical pages across URLs.
- Crawler traps: max depth per domain, URL length limits, pattern detection.

## Staff+ signals
- Do the throughput math and size the fleet; note bandwidth (20k × ~100 KB ≈ 2 GB/s).
- Frontier prioritization (PageRank-ish importance, freshness for re-crawls) as a second-phase concern.
- Domain hot spots (a huge domain with 1 req/s politeness becomes the long pole → start it early).
- Re-crawl scheduling and change detection (ETags, Last-Modified).
- Legal/ethical: robots, opt-outs, PII handling in training data.

## Level bar
Mid: working data flow, basics of politeness, rough scaling. Senior: politeness/robots in depth, scaling to the 5-day target. Staff: 3+ deep areas, interviewer learns something; covering all of the above even partially is a strong spot.
