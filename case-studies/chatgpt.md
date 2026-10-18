# Design an LLM Chat Service (ChatGPT)

Source: outline only; answers are independent · Patterns: real-time updates, long-running tasks · Difficulty: hard

## Scope
- **FR:** send a prompt and receive a streamed AI response; view past chats and resume with context carried across turns. Below the line: tools/agents, file uploads, voice, sharing.
- **NFR:** time-to-first-token low (≲ 1 s); streams survive reconnects and deploys; GPU capacity is scarce and expensive — maximize utilization; fair across users with paid tiers prioritized; tens of millions of users.

## Crux
Generation is a long-running, expensive, GPU-bound task whose output must be streamed; the system is really a scheduler for scarce GPUs with a resumable streaming front end.

## Skeleton
Entities: Conversation, Message (role, content, status), Run/Generation (id, model, status, tokens), User/Plan. API: `POST /conversations/{id}/messages` → run id; `GET /runs/{id}/stream` (SSE); `POST /runs/{id}/cancel`; `GET /conversations?cursor`.
API → conversation store (Postgres/DynamoDB) → **router/scheduler** → inference workers (GPU pools per model, continuous batching, KV cache) → token stream buffer (Redis Streams / Kafka) → SSE edge servers.

## Deep dives

**1. Stream tokens to the client**
SSE (one-way, HTTP-friendly). Decouple generation from the connection: workers append tokens to a per-run stream (Redis Stream) with offsets; SSE servers tail it and forward. Persist the final message on completion (and periodically for long outputs).

**2. Survive reconnects and deploys**
Client reconnects with `Last-Event-ID` = last token offset → SSE server resumes from the buffer; generation continues independently of the client connection; buffers TTL'd after completion. Deploys drain SSE servers gracefully (clients reconnect elsewhere — any server can tail any run).

**3. Route & schedule across GPU workers**
Queue per model; scheduler places requests on workers with capacity (KV-cache memory, batch slots); **continuous batching** maximizes throughput; prefix/KV-cache affinity (route follow-up turns to the worker holding the conversation's prefix cache) when it pays; autoscale is slow (GPU cold start, model load) → keep headroom, shed load gracefully.

**4. Heavy users vs fairness & paid tiers**
Token-based rate limits per user/tier (tokens per minute, not requests); weighted fair queueing across tiers with reserved capacity for paid; degrade free tier (smaller model, queue) under pressure; per-user concurrency caps.

**5. Long conversations vs cost**
Context window growth = linear cost per turn. Options: sliding window + summarization of older turns, retrieval over conversation history (embeddings) to include only relevant past turns, prompt/prefix caching for the stable prefix, memory features stored separately. Explain the quality/cost tradeoff.

**Cancellation:** cancel endpoint sets a flag the worker checks between decoding steps → frees GPU batch slot immediately.

## Staff+ signals
- Frame it as capacity management: GPU-hours are the cost driver; utilization, batching, and admission control are the design.
- Separate control plane (conversations, billing, routing) from data plane (token streaming).
- Safety filtering inline on input/output with latency budget; abuse detection.
- Multi-region GPU pools and model rollout (canary new model versions, version pinned per conversation).

## Traps
Holding the HTTP request open as the only channel (lost on disconnect); WebSockets by default; request-count rate limits; ignoring GPU scheduling entirely.
