# Design Business Search & Reviews (Yelp)

Source: outline only; answers are independent · Patterns: scaling reads, proximity · Difficulty: easy

## Scope
- **FR:** search businesses (keyword, category, location, rating); view a business; leave a review (rating + text). Below the line: photos, reservations, messaging.
- **NFR:** search < 500 ms; read-heavy (100:1+); availability > consistency; ~100M users, ~10M businesses; one review per user per business.

## Crux
Combining text + geo + filter search with low latency, and keeping derived values (average rating) cheap to serve without hot write contention.

## Skeleton
Entities: Business (location, category, avg_rating, num_reviews), Review (business_id, user_id, rating, text), User. API: `GET /businesses?q&lat&long&radius&category&minRating&cursor`, `GET /businesses/{id}`, `POST /businesses/{id}/reviews`.
Postgres (PostGIS) as source of truth; business service + review service; optional Elasticsearch via CDC.

## Deep dives

**1. Average rating for search results**
- ✗ Compute AVG over reviews at query time.
- ◐ Periodic batch recompute — stale.
- ✓ Maintain `num_reviews` and `avg_rating` (or `rating_sum`) on the business row, updated in the same transaction as the review insert (`rating_sum = rating_sum + :r`). Review volume per business is low → no contention issue; if needed, batch via queue.

**2. One review per user per business**
- ✓ Unique constraint on `(user_id, business_id)` in the DB — app-level checks race. Upsert if edits allowed.

**3. Complex search efficiently**
- Small scale: Postgres with PostGIS GiST + B-tree on category/rating + GIN full-text, combined in one query.
- At scale: Elasticsearch with geo_point, text, filters; sync via CDC; denormalize avg_rating into the document. Geo-shard isn't required (10M docs fits a modest cluster).
- Cache popular (location-cell, query) combinations; key by geohash cell, not raw coordinates.

**4. Search by named place ("Pizza in Mission District")**
- Store place polygons (cities, neighborhoods) in PostGIS / ES geo_shape; resolve name → polygon → `geo_shape`/`ST_Contains` filter; or precompute business → place ids at write time and filter by term (cheaper reads).

## Staff+ signals
- Point out that 10M businesses ≈ tens of GB — fits on one well-indexed Postgres; ES only when relevance/fuzzy matters.
- Ranking: relevance × distance × rating (Bayesian average to avoid 5-star single-review dominance).
- Review spam/fraud and moderation pipeline (async).
- Read-your-writes for the reviewer (show their review immediately even if search index lags).

## Traps
`LIKE '%pizza%'`; lat/long range filters without spatial index; recomputing averages at read time; racy uniqueness checks.
