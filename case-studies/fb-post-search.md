# Design Post Search (Facebook / Twitter search)

Source: Hello Interview breakdown (full) · Patterns: scaling reads, scaling writes, search · Difficulty: hard

## Scope
- **FR:** create and like posts; search posts by keyword; sort results by recency or like count. Below the line: fuzzy matching, personalization, privacy filtering, images.
- **NFR:** median query < 500 ms; very high query and write volume; new posts searchable within 1 minute; all posts discoverable (old ones may be slower); highly available.

Note: interviewers often **forbid Elasticsearch** here — you're expected to build the inverted index yourself. Ask.

## Crux
Building and maintaining an inverted index under huge write volume (posts *and* likes, which change sort order constantly) while keeping queries fast.

## Skeleton
Write path: Post/Like service → Kafka → Ingestion service tokenizes posts → Redis-based index: per keyword, two sorted sets — `kw:{word}:recent` (score = created_at) and `kw:{word}:likes` (score = like count). Query path: Search service → cache → Redis index → hydrate post ids from Post DB.

## Deep dives

**1. High query volume**
◐ Distributed cache of query results with TTL < 1 min (honours freshness SLA). ✓ **CDN edge caching** of `/search` responses via cache-control headers — popular queries served in tens of ms. (Works because results aren't personalized in scope.)

**2. Multi-keyword and phrase queries**
◐ Intersect posting lists of each term (sets may be millions of ids → network + CPU heavy). ✓ Index **bigrams/shingles** ("taylor swift") as keys — phrase queries become single lookups; cost is a much larger (sparser) index → index bigrams selectively or only above a frequency threshold.

**3. Write volume (likes)**
Each like would rewrite sort scores in every keyword set containing the post. ◐ Batch likes per post over ~30 s and apply one increment. ✓ **Two-stage:** update index like-scores only at milestones (powers of 2/10) so writes drop exponentially; at query time fetch top N candidates by approximate score, then fetch exact like counts from the Like service and re-rank. Index is knowingly stale; final order is correct.

**4. Storage**
Cap posting lists per keyword (keep top ~1k–10k per sort); tier rarely queried keywords to blob storage (S3/R2) based on search analytics; query hot tier first, fall back to cold with a latency penalty (meets "old posts discoverable, slower OK").

## Staff+ signals
- Explicitly separate index freshness (≤ 1 min via Kafka ingestion lag) from result-cache TTL.
- Shard the index by keyword (hot keywords → replicate) vs by document (scatter-gather every query) — and justify.
- Privacy/visibility filtering is the real-world hard part (per-viewer ACLs make caching hard) — scope it explicitly.
- Rebuild/backfill path from Kafka/the post store when tokenization changes.

## Level bar
Mid: both ingestion and query sides defined; good discussion even from bad solutions. Senior: optimized critical paths, proper reverse index, caching strategy. Staff: several deep dives with novel insight.
