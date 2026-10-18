# The Staff+ Lens

Staff-level design isn't a senior answer done faster. It's a different conversation, usually with a staff+ interviewer looking for evidence you can own the hardest technical risk of a product area. Part A is the source course's staff guidance; Part B extends it with the dimensions staff+ engineers are expected to bring unprompted.

## Part A — Five keys

1. **Communicate at the interviewer's level.** Don't explain first-year material (what a load balancer does, what an index is) unless asked. Over-explaining wastes time, sounds freshly learned, and signals you don't usually talk with staff peers. Maximize bandwidth: named patterns, precise vocabulary, numbers.
2. **Find the crux.** Most of any problem is ordinary (CRUD, a search box, a workflow at modest traffic). Grok it fast, say so ("standard CRUD service, moving on"), **earmark** the hard parts ("keeping hour-long transactions fault tolerant is tricky; coming back to it"), and spend your time there.
3. **Ruthlessly cut complexity.** Ask what's actually required before building the sophisticated version. *10k places, rarely updated → sync periodically and search in memory.* Single-node and in-memory systems have far fewer failure modes. Start minimal, fix pinch points as they appear, justify every component (YAGNI). Choose your battles with company culture (don't spend the hour arguing a microservices shop into a monolith).
4. **Demonstrate depth from experience.** You haven't built Uber, but you've built workflow-heavy or high-write systems. Bring concrete lessons: what broke, what you'd configure, numbers you've seen.
5. **Show your instincts — decide.** Don't offer 2–4 options and wait for the interviewer to choose. "Postgres, because I need multi-table transactions and durability; not a bottleneck here, revisit if it becomes one." Justify enough, not exhaustively.

**Calibration from the breakdowns:** breadth:depth ≈ 40:60; go deep on 2–3+ areas; the interviewer should intervene "only to focus, not to steer"; the strongest signal is the interviewer **learning something** from you; staff candidates often add deep dives the interviewer didn't plan and articulate the whole solution space with clear judgment.

Judgment beats recall: e.g., in an ad-click aggregator with 1-minute windows on a replayable stream, questioning whether Flink checkpointing is even needed reads as more senior than reciting it.

## Part B — Dimensions to bring unprompted

Pick the 2–3 that matter for *this* problem; don't recite all of them.

### Failure modes, 2–3 levels deep
For each critical component: how does it fail (crash, partition, slow, overloaded, stale), how is that detected (health checks, heartbeats, lag metrics), what's the blast radius, how do you recover (failover, replay, reconciliation), and what does the user see? Examples: Redis lock primary fails after granting → duplicate holder → fencing token or DB-level conditional write; pub/sub drops a message → sequence gaps → client resync from durable inbox.

### Correctness under concurrency and retries
Idempotency keys, conditional writes, exactly-once *effects* via at-least-once + dedupe, outbox instead of dual writes, ordering guarantees per key, reconciliation against a source of truth.

### Evolution and migration
How does v1 become v2 without downtime? Dual-write + backfill + verify + cut-over; schema evolution; resharding; versioned workflows; feature flags; replay pipelines from the log to rebuild derived stores.

### Operability
SLOs per critical path (e.g., booking success p99), golden signals, consumer lag alerts, DLQ alarms, runbooks, safe deploys (canary, connection draining for WebSocket fleets), load testing for known peaks (on-sales, New Year's Eve), backpressure and load shedding policies.

### Multi-region, cells, blast radius
Active-passive vs active-active; what data must be global (usernames, balances) vs regional (rides, live comments); cell-based architecture (shard users into independent cells to cap blast radius); data residency.

### Cost
Back-of-envelope cost when it drives a choice: DynamoDB vs Cassandra at 500k writes/s, RAM vs SSD for indexes, CDN egress for video, GPU utilization for LLM serving, hot vs cold storage tiers.

### Security and abuse
Authz on every resource, signed URLs as bearer tokens, rate limiting, bot/fraud (click fraud, scalping in ticketing), PII/PCI scope reduction (tokenization), audit logs.

### Product and organizational judgment
Question requirements that don't pay for themselves ("does the 100k-follow user need second-level freshness?"; cap friends at 5,000; show a representative subset of comments). Name ownership boundaries (which team owns the search index sync) when it affects the design.

## Staff-level phrasing patterns

- Scoping: "Top three above the line are X, Y, Z. I'm deliberately leaving out A and B — shout if either matters to you."
- Crux: "Most of this is a standard read-heavy service. The real problem is preventing double booking when ten million people arrive at once; I'll spend most of our time there."
- Simplicity: "Before sharding — 1B rows × 500 B is 500 GB. One Postgres primary with replicas covers it."
- Decision: "I'll go with an expiring reservation checked inline in the transaction, not a cron sweep — the sweep becomes cleanup, not correctness."
- Depth: "Where this bites in practice is…"
- Tradeoff closure: "We give up X; acceptable because Y; if Z changes I'd switch to W."

## Anti-signals at staff

- Waiting for the interviewer to find the problems.
- Comprehensive option lists with no choice.
- Tutorial-mode explanations of basics.
- Distributed everything at modest scale; or the reverse — no answer when numbers genuinely demand distribution.
- Box-drawing without request paths, state changes, or failure behavior.
- Defensive or argumentative when probed; or caving immediately without reasoning.
