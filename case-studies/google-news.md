# Design a News Aggregator (Google News)

Source: Hello Interview outline (premium); answers are independent · Patterns: scaling reads, long-running tasks · Difficulty: medium

## Scope
- **FR:** view an aggregated feed of articles from thousands of publishers worldwide; infinite scroll; click through to the publisher. Below the line (bonus): category feeds, personalization.
- **NFR:** feed < 200 ms; new articles visible within **30 minutes** of publication; read-heavy with huge spikes on breaking news; availability > consistency.

## Crux
Ingestion is a crawler/poller problem with a freshness SLA; serving is an almost entirely cacheable, read-dominated feed that must survive breaking-news spikes.

## Skeleton
Entities: Publisher (feed URL, crawl cadence), Article (url, title, summary, thumbnail, region, category, published_at), Feed. API: `GET /feed?region&cursor&limit`, `GET /articles/{id}` (→ redirect).
Ingestion: scheduler → collectors poll RSS/sitemaps/webhooks → dedupe (URL canonicalization + content hash/near-dup) → extract metadata, thumbnail to S3/CDN → Article DB → precomputed feeds (Redis sorted sets per region/category, score = recency/rank).

## Deep dives

**1. Pagination consistency**
Offset pagination duplicates/skips as new articles arrive → **cursor** = (published_at, article_id) of the last item; newest-first reads use `< cursor`; "new stories" banner for items above the top.

**2. < 200 ms feed**
Precompute per region/category feeds into Redis; serve top pages directly; CDN-cache the first page per region with short TTL (feeds aren't personalized in v1); hydrate article cards from a cache.

**3. Freshness within 30 min**
Poll cadence per publisher based on historical publish rate (hot publishers every minute, quiet ones hourly); accept push (WebSub/webhooks) where available; queue-based collectors with retries; ingestion lag metric with alert below SLA.

**4. Media**
Download publisher thumbnails once, resize, store in S3, serve via CDN — never hotlink.

**5. Breaking-news spikes**
Everything on the read path is cache/CDN; request coalescing for cache misses; autoscale stateless feed service; graceful degradation (serve slightly stale feed).

**Bonus — categories:** classify at ingestion (publisher tags + ML classifier), maintain per-category sorted sets. **Personalization:** candidate generation (followed topics, history embeddings) + lightweight ranking; precompute for active users, fall back to regional feed.

## Staff+ signals
- Story clustering (group the same event across publishers) as the real product problem; near-duplicate detection (SimHash/MinHash).
- Politeness and publisher agreements; legal constraints on snippets.
- Ranking freshness vs authority; abuse (spam publishers).

## Traps
Offset pagination; crawling every source on one schedule; hotlinking images; building personalization before the core feed works.
