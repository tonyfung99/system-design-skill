# Design a Price Tracker (CamelCamelCamel)

Source: Hello Interview outline (premium); answers are independent · Patterns: scaling writes, long-running tasks, notifications · Difficulty: medium

## Scope
- **FR:** view price history for Amazon products (website + Chrome extension); subscribe to price-drop alerts with thresholds. Below the line: other retailers, deal pages.
- **NFR:** track ~500M products; alerts within a reasonable window (e.g., hours for long tail, minutes for popular); fast history charts (< 500 ms); availability > consistency; respect retailer rate limits.

## Crux
Discovering and re-checking 500M products under external rate limits (prioritization), trusting crowd-sourced prices, and turning price changes into targeted notifications efficiently.

## Skeleton
Entities: Product, PricePoint (product_id, price, observed_at, source), Subscription (user, product, threshold), Notification. API: `GET /products/{id}/price-history?period`, `POST /subscriptions`, extension `POST /prices` (observed price).
Crawl scheduler → crawler fleet (+ official APIs) → price-change events (Kafka) → TSDB/columnar store for history → alert matcher → notification service (email/push).

## Deep dives

**1. Tracking 500M products**
You can't crawl everything hourly. **Priority scheduling:** frequency by popularity, subscriber count, volatility, and recency of change (hot → hourly, cold → weekly); discovery via sitemaps, category crawls, and extension sightings. Crawlers obey rate limits (per-domain token bucket), proxy pools, backoff on blocks. Use official product APIs where possible.

**2. Malicious/incorrect extension price reports**
Treat as untrusted signals: validate against recent history (outlier detection), require corroboration from multiple independent users or a confirming crawl before publishing, reputation scores per reporter, rate limit per user, sign extension requests. Extension data mostly *triggers prioritized re-crawls* rather than being trusted outright.

**3. Processing changes → notifications**
Only emit events on actual change (compare to last price in a cache). Matcher: subscriptions indexed by product_id with thresholds sorted (`price_threshold >= new_price` range query → all qualifying subscribers in one scan). Fan out via a queue to the notification service; dedupe (don't re-alert for the same drop); batching/digests for users with many alerts.

**4. Fast price-history charts**
History is append-only time series: store in TimescaleDB/ClickHouse/Cassandra (partition product_id, cluster by time); precompute downsampled series (daily min/max/avg) for long ranges; cache chart data per product + period.

## Staff+ signals
- Make the prioritization function explicit and tie it to a freshness SLO per tier.
- Cost/legality of scraping; ToS risk → prefer affiliate APIs; a business constraint shaping architecture.
- Change-data volume math: 500M products × few checks/day, but only a small % change → emit deltas only.
- Notification storms on Prime Day — rate shaping and priority queues.

## Traps
Uniform crawl schedule; trusting client-reported prices; scanning all subscriptions per price change; aggregating raw history at read time.
