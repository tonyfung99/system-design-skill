# Design a URL Shortener (Bitly)

Source: full breakdown · Patterns: scaling reads · Difficulty: easy

## Scope
- **FR:** create short URL from long URL (optional custom alias, optional expiry); redirect short → long. Below the line: analytics, accounts, link management.
- **NFR:** short codes unique; redirect < 100 ms; 99.99% available (availability > consistency for reads); 1B URLs, 100M DAU; read:write ≈ 1000:1.

## Crux
Generating unique, short codes without coordination bottlenecks, and serving redirects fast at very high read volume. Storage is *not* the problem.

## Skeleton
Entities: URL mapping (short_code PK, long_url, created_at, expires_at, creator). API: `POST /urls {longUrl, alias?, expiresAt?} → shortUrl`; `GET /{code}` → 302. Write service + redirect service (stateless) → Postgres/DynamoDB; Redis cache; CDN in front of redirects.

Redirect status: **302** (temporary) keeps every hit flowing through you (analytics, expiry, edits); 301 lets browsers cache and bypass you. Pick 302 unless the interviewer only cares about load.

## Deep dives

**1. Unique short codes**
- ✗ Prefix of the long URL — collisions everywhere.
- ◐ Hash (canonicalize URL → hash → base62 → take 6–8 chars) — dedupes identical URLs but truncation collides; needs check-and-retry on insert (unique constraint).
- ✓ **Global counter + base62** — Redis `INCR` is atomic and single-threaded; 7 base62 chars ≈ 3.5T codes. Avoid one-call-per-URL: each write server leases **blocks of 1,000 ids** (counter batching). Sequential codes are guessable → if that matters, apply a reversible bijective scramble (e.g., Feistel/sqids) before encoding. Counter durability: Redis persistence/replica, or a DB sequence; a lost range is harmless, a reused one isn't — start new blocks above the max persisted.
- Custom aliases: separate namespace or same table with a unique constraint; reject on conflict.

**2. Fast redirects**
- ◐ Index on short_code (PK) — O(log n), a few ms.
- ✓ Redis cache-aside (LRU, popular codes) — ~1 ms; ✓ CDN/edge (cache 302s or run redirect logic at the edge) for popular links. Respect expiry: cache TTL ≤ time to expiry.

**3. Scale (1B URLs, 100M DAU)**
- Size: ~500 B × 1B ≈ 500 GB → a single Postgres fits; writes ≈ 100k/day ≈ 1/s. Any DB works — pick what you know.
- Reads: 100M DAU × few redirects ≈ ~10k/s average, spikes higher → cache + replicas; redirect service scales separately from write service.
- HA: replicas with failover, backups; counter service HA (replica or fall back to DB sequence blocks).

## Staff+ signals
- Kill the storage debate in one line with the math; spend time on ID generation tradeoffs (coordination, guessability, multi-region: per-region ID prefixes or block allocation).
- 301 vs 302 as a product/analytics decision.
- Abuse: malware/phishing scanning on create, rate limits per user/IP.
- Analytics without slowing redirects: emit click events async to Kafka.

## Traps
Sharding for 500 GB; hashing without collision handling; one Redis call per create; forgetting expiry semantics in the cache.
