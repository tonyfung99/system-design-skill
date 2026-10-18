# Design a Photo/Video Feed (Instagram)

Source: outline only; answers are independent · Patterns: scaling reads, large blobs, long-running tasks · Difficulty: medium

## Scope
- **FR:** create posts with photos/videos + caption; follow users; chronological feed of followed users. Below the line: likes/comments, stories, explore, DMs.
- **NFR:** feed < 500 ms; media renders "instantly" — photos ≤ 8 MB, videos ≤ 4 GB; 500M DAU; availability > consistency.

## Crux
It's news feed (fan-out) **plus** media: getting large uploads in via presigned URLs and serving optimized media from the edge fast.

## Skeleton
Entities: User, Follow, Post (caption, media keys, status), Media (variants). API: `POST /posts` (→ presigned URLs), `POST /follows`, `GET /feed?cursor`.
Post service (DynamoDB/Cassandra), Follow store with reverse index, Feed service with precomputed feeds (Redis/DynamoDB), S3 + media processing workers + CDN.

## Deep dives

**1. Low-latency feed (< 500 ms)**
Same playbook as `fb-news-feed.md`: precomputed feed per user (latest ~500 post ids) via async fan-out-on-write workers; **hybrid** for celebrities (pull their recent posts at read time and merge); skip inactive users; hydrate post metadata from a replicated cache. Cursor pagination by (created_at, post_id).

**2. Instant media rendering**
- Upload: presigned S3 URL (multipart for videos), post status `processing` → media pipeline (long-running tasks) creates thumbnails/resized variants (WebP/AVIF) and video ABR renditions → status `published` → fan-out begins (don't fan out unviewable posts).
- Serve: CDN with signed URLs; client requests the variant matching screen size/network; progressive/blur-up placeholders; prefetch the next feed page's media.
- Videos: segment-based ABR (see `youtube.md`), first segments small for fast start.

**3. Scale to 500M DAU**
Reads ≫ writes (~100M posts/day vs billions of feed views). Shard post data by post_id/user_id; follow graph by follower and by followee; feed workers autoscale on queue depth; CDN absorbs nearly all media bytes; multi-region read replicas.

## Staff+ signals
- Sequence the publish flow: fan-out only after media is ready; idempotent workers.
- Storage cost math (variants × posts); lifecycle tiers for old media; dedupe identical uploads.
- Celebrity threshold derived from fan-out cost; feed backfill for newly followed accounts.
- Hot posts → replicated metadata cache; CDN shields media.
- Abuse/moderation (CSAM detection hashing) before publish.

## Traps
Uploading media through app servers; fanning out before processing completes; caching media URLs that expire before the cache entry.
