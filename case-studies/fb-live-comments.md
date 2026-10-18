# Design Live Comments (Facebook Live / Twitch chat)

Source: Hello Interview breakdown (full) · Patterns: real-time updates · Difficulty: medium

## Scope
- **FR:** post comments on a live video; see new comments in near real time; see comments posted before joining. Below the line: replies, reactions, moderation detail.
- **NFR:** millions of concurrent live videos, thousands of comments/s on a popular video; availability > consistency; < 200 ms broadcast latency.

## Crux
Broadcast fan-out: few writers, enormous numbers of readers, unevenly concentrated on a handful of mega-streams.

## Skeleton
Entities: User, LiveVideo, Comment. API: `POST /comments/{liveVideoId}`, `GET /comments/{liveVideoId}?cursor` (history), SSE stream for new comments.
Comment service → Cassandra/DynamoDB (partition liveVideoId, sort by time) → publish to fan-out layer → realtime servers hold viewer connections.

## Deep dives

**1. Push protocol**
◐ WebSockets — bidirectional is overkill when almost nobody writes. ✓ **SSE** for reads + plain POST for writes; works with HTTP infra, built-in reconnect.

**2. Millions of concurrent viewers**
Separate write path (comment service) from read path (realtime messaging servers holding SSE connections).
- ◐ Naive pub/sub: every server gets every comment, filters locally — doesn't scale.
- ✓ **Partitioned pub/sub:** N channels via `hash(liveVideoId) % N`; servers subscribe only to channels for their viewers. Improve further with **viewer co-location** — route viewers of the same video to the same servers (L7 routing on videoId / consistent hashing) so each server subscribes to few channels.
- ✓ **Dispatcher** alternative: a service tracks which servers have viewers of which video (registration + heartbeats) and forwards each comment directly.
- Mega streams (100k+ comments/s visible to millions): ◐ show a **representative sample** (rate-adaptive sampling so each viewer sees a readable ~N/s); ✓ switch to **pull via CDN snapshots** — ring buffer of the latest 100–200 comments written every second to the CDN; clients poll the edge.

**3. Disconnections**
✗ Ignore gaps. ✓ SSE `Last-Event-ID` replay on reconnect, or client-tracked last id → `GET …?since=` catch-up; UX: animate catch-up or "you missed 47 comments".

## Staff+ signals
- Identify the read/write asymmetry immediately and pick SSE without prompting.
- Call out reliability limits of Redis Pub/Sub (at-most-once) and when that's acceptable (comments are ephemeral).
- Hot-stream detection and dynamic mode switch (push → sample → CDN pull).
- Moderation latency vs broadcast latency (pre-publish filtering for spam/abuse with a tight budget).
- Reconnect storms after a server/deploy failure: jittered backoff, gradual draining.

## Level bar
Mid: realizes polling limits, reaches pub/sub with hints. Senior: pub/sub with minimal hints, leads scaling tradeoffs. Staff: names pub/sub limits (reliability, scale) unprompted, knows exact tech and tradeoffs.
