# Probe Bank

Probes test reasoning, not recall. Ask one at a time; let silence work. For each area: the probe, what a **weak** answer sounds like, and what a **staff** answer contains.

## Requirements & scope
| Probe | Weak | Staff |
|---|---|---|
| "What's most important for this system?" | lists every feature | top 3 + why; explicit below-the-line; checks with you |
| "Consistency or availability?" | one answer for the whole system | per feature, with the business consequence of staleness |
| "How big is this?" | recites DAU/QPS with no consequence | estimates only what changes a decision, says what it changes |
| "What if we don't need X to scale?" | ignores | simplifies aggressively (in-memory, single node) |

## Data & storage
| Probe | Weak | Staff |
|---|---|---|
| "Why this database?" | SQL-vs-NoSQL generalities | specific property needed (transactions, partition model, write path) + what's given up |
| "What's your partition key?" | "user id" without checking queries | key derived from dominant access pattern; names the scatter-gather query it makes expensive |
| "This table is 10 TB now." | "shard it" | checks whether it's actually needed; picks strategy; migration plan (dual write, backfill, cutover) |
| "How does the index stay in sync?" | "it just does" | CDC/outbox, lag, drift detection, rebuild path |

## Scale & performance
| Probe | Weak | Staff |
|---|---|---|
| "Traffic goes 10×." | add servers everywhere | finds which component breaks first, with numbers |
| "One key gets 100× the traffic." | consistent hashing fixes it | distinguishes structural vs workload imbalance; replicate/salt/split/cache; detection |
| "The cache entry expires under load." | — | stampede protection: single-flight, stale-while-revalidate, jitter |
| "Latency budget is 100 ms end to end." | — | per-hop budget; removes a hop (edge, cache, co-location) |

## Correctness & concurrency
| Probe | Weak | Staff |
|---|---|---|
| "Two users do this at the same instant." | "we lock it" | atomic conditional write / OCC / single writer; where the invariant is enforced |
| "The client retries." | — | idempotency key; dedupe store; conditional state transitions |
| "Your lock holder pauses for 30 s." | — | lease expiry, fencing tokens, invariant in the DB |
| "Is this exactly-once?" | "yes, Kafka" | at-least-once + idempotent sink; boundary where exactly-once holds |

## Failure & resilience
| Probe | Weak | Staff |
|---|---|---|
| "This box dies." (pick the most critical) | "we have replicas" | detection, failover, data loss window, user impact, recovery, how you'd test it |
| "The queue backs up for an hour." | "autoscale" | backpressure, priority, shed, DLQ, catch-up rate math |
| "A downstream dependency is slow, not down." | — | timeouts < caller timeout, circuit breaker, bulkheads, degraded mode |
| "A region goes away." | — | which data is regional vs global, RPO/RTO, what still works |

## Real-time & async
| Probe | Weak | Staff |
|---|---|---|
| "Why WebSockets?" | "real-time" | push direction and frequency justify SSE vs WS vs polling |
| "Connection drops for 10 s." | — | heartbeat detection, resume token/sequence, catch-up from durable store |
| "One update to 5M clients." | loop over sockets | fan-out tiers, co-location, sampling, CDN pull |
| "The workflow crashes at step 3 of 5." | — | persisted state / durable execution, idempotent steps, compensation |

## Operability, evolution, cost (staff+ ceiling)
| Probe | Weak | Staff |
|---|---|---|
| "How do you know it's healthy?" | "monitoring" | SLOs on the critical path, specific metrics (lag, hit rate, error budget), alert on symptoms |
| "Ship v2 of the schema/pipeline with no downtime." | — | expand/contract, dual writes, backfill, verification, rollback |
| "Finance says cut cost 50%." | — | identifies the dominant cost driver with rough numbers; tiering, sampling, cheaper storage, smarter caching |
| "What would you build first, and what would you defer?" | everything now | sequencing by risk; explicit YAGNI |

## Pushback probes (test conviction)
Use on a *correct* decision occasionally:
- "Isn't that overkill?" / "Wouldn't X be simpler?"
- "I've seen that fail in production."
- "Why not just use <popular tool>?"

Good: holds position with reasoning, or updates with reasoning and says what changed. Bad: caves instantly, or argues without engaging.

## Hint ladder (record every rung used)
1. Restate the requirement at risk: "Remember we said no double booking."
2. Point at the component: "Walk me through what happens in the booking service when two requests arrive."
3. Name the problem class: "This is a race condition — how do you prevent it?"
Rung 3 on a crux item caps that area at "below bar" for staff.
