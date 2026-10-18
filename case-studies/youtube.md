# Design a Video Platform (YouTube / Netflix)

Source: full breakdown · Patterns: large blobs, long-running tasks, multi-step · Difficulty: medium

## Scope
- **FR:** upload videos; watch (stream) videos. Below the line: comments, recommendations, search, live streaming.
- **NFR:** availability > consistency; uploads of tens of GB; low-latency streaming even on poor networks; ~1M uploads/day, ~100M views/day; resumable uploads.

## Crux
(1) Turning an uploaded file into adaptive-bitrate streams via a parallel processing pipeline. (2) Moving huge files in and out without your servers in the data path.

## Skeleton
Entities: Video (bytes in S3), VideoMetadata (title, uploader, status, manifest URLs, chunks[]), User. API: `POST /videos` (metadata → presigned multipart URLs), `GET /videos/{id}` (metadata + manifest URL).
Video service (stateless) → metadata in Cassandra/DynamoDB (partition by videoId); S3 for originals and processed segments; processing pipeline; CDN for playback.

## Deep dives

**1. Adaptive bitrate streaming**
Output = segments (2–10 s) in multiple resolutions/codecs + manifests (HLS `.m3u8` / DASH `.mpd`: a primary manifest listing renditions, media manifests listing segment URLs). Player picks rendition per segment based on bandwidth.
Pipeline as a **DAG**: split original (ffmpeg) → transcode each segment × each rendition in parallel (+ audio, captions, thumbnails) → assemble manifests → mark ready. Orchestrate with a DAG/workflow engine (Temporal) and a worker pool; intermediate artifacts in S3 (pass pointers, not bytes). Optional: start processing as chunks arrive (pipelined upload).

**2. Resumable uploads**
Client chunks (5–10 MB) with fingerprints; metadata tracks chunk status; S3 multipart upload; backend verifies parts (ETag/ListParts) rather than trusting the client; `CompleteMultipartUpload` event triggers processing; resume by reading which chunks are done.

**3. Scale (1M uploads/day, 100M views/day)**
Stateless services scale horizontally; Cassandra scales by videoId (replicate/cache hot metadata); processing autoscaled on queue depth; S3 is not the bottleneck but is regional → **CDN** caches segments near viewers (popular content pushed proactively). Most bytes served from CDN; origin shielding.

## Staff+ signals
- Cost dominates: transcode ladder per popularity (encode cheap renditions first, premium codecs like AV1 only for popular videos), CDN egress, storage tiers for cold originals.
- Long tail: most videos are rarely watched — don't push them to every edge.
- Failure handling in the DAG: idempotent segment tasks, retries, partial reprocessing; versioned pipelines when codecs change.
- Upload abuse/moderation and copyright (async content ID) before publish.
- View counting at scale is its own problem (see `top-k.md`).

## Level bar
Mid: functional upload/playback via S3, converge on multipart upload and segment-based streaming. Senior: detail post-processing and multipart/resumable uploads. Staff: depth on all, steer into interesting areas (cost, pipeline reliability), peer-level tradeoffs.
