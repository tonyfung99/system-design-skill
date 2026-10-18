# Design a Notification System

Source: Hello Interview outline (premium); answers are independent · Patterns: multi-step processes, scaling writes · Difficulty: medium

## Scope
- **FR:** upstream services send a notification to a user (now or scheduled) over push/SMS/email/in-app; send **campaigns** to user segments; users set preferences (channels, categories, quiet hours, unsubscribe). Below the line: template editor UI, analytics dashboards.
- **NFR:** an accepted notification is **never dropped**; high-priority (OTP, security, transactional) delivered < 5 s even during a campaign blast; no duplicates in practice; campaigns to 100M+ users; provider rate limits respected.

## Crux
Durability from acceptance to provider handoff, priority isolation so a 100M-user campaign can't delay an OTP, and dedupe on top of at-least-once delivery. (Hello Interview deliberately uses queues rather than Temporal here because the delivery/retry machinery *is* the interview.)

## Skeleton
Entities: Notification (id, user, type, priority, template, scheduled_at, status), Preference, Campaign, DeliveryAttempt, DeviceToken. API: `POST /notifications {idempotencyKey, userId, template, priority, sendAt?}` → 202 + id; `POST /campaigns {segment, template, schedule}`; `PUT /users/{id}/preferences`.
Ingest API → durable store + queue → router (preferences, quiet hours, channel choice, rendering) → per-channel, per-priority queues → channel workers → providers (APNs/FCM, Twilio, SES) → status callbacks.

## Deep dives

**1. Never drop an accepted notification**
Ack the caller only after a durable write (DB row + outbox, or Kafka `acks=all`). Each stage consumes at-least-once and commits after hand-off; provider calls retried with backoff; permanent failures → DLQ + status; a sweeper re-enqueues notifications stuck past SLA. Scheduled sends via a time-bucketed scheduler (see `job-scheduler.md`).

**2. High priority < 5 s during a blast**
Separate queues **and** worker pools per priority (and channel); reserve provider rate-limit budget for high priority; campaigns throttled (token bucket) and expanded lazily — stream segment user ids in batches instead of enqueuing 100M messages at once; shed/defer low-priority marketing first.

**3. Duplicates on top of at-least-once**
Idempotency key at ingest (dedupe table with TTL); per-(notification, channel) delivery record with conditional transitions (`pending → sending → sent`) so retries skip completed sends; pass idempotency keys to providers that support them; client-side dedupe by notification id for push/in-app.

## Staff+ signals
- Preference/compliance logic (unsubscribe, CAN-SPAM/GDPR, quiet hours by time zone, frequency caps) centralized in the router.
- Provider failover (secondary SMS provider), per-provider circuit breakers, cost-aware channel selection.
- Campaign fan-out math and pacing; prune invalid device tokens from provider feedback.
- Per-stage observability: accepted → routed → sent → delivered → opened, with lag SLOs.

## Traps
One shared queue for everything; enqueueing a whole campaign at once; acking before a durable write; ignoring provider rate limits.
