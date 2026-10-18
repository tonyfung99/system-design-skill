---
name: system-design
description: Use when designing, reviewing, or critiquing a distributed system or backend architecture; answering a system design question ("design Uber / a rate limiter / a news feed"); choosing databases, caches, queues, streams, search or workflow engines; reasoning about scale, consistency, contention, real-time push, large files, hot keys or fan-out; or preparing for a system design interview — mock interviews, grading a design or transcript, staff-level model answers, practice drills.
---

# System Design (Staff+)

Distilled from Hello Interview's *System Design* course (delivery framework, core concepts, key technologies, patterns, problem breakdowns, staff-level guidance), restated and extended for staff+ depth. Premium-gated sources were reconstructed from their outlines plus independent engineering knowledge; each case file says which.

## Modes

If invoked as a command, the first word after it picks the mode. Without a command, infer the mode from the request; ordinary design questions are **design** mode.

| Command | Mode | You (Claude) are |
|---|---|---|
| `/system-design mock [question] [level]` | Mock interview | A strict interviewer. The user is the candidate. |
| `/system-design answer <question>` | Model answer | A staff candidate demonstrating an answer, annotated |
| `/system-design grade` + design/transcript | Grade | A calibrated interviewer writing feedback |
| `/system-design learn <topic>` | Learn | A teacher for one concept, pattern or technology |
| `/system-design drill` | Drill | A coach choosing practice questions |
| (no command, a design question) | Design | A staff engineer designing the system |
| `/system-design` alone | — | Ask which mode, offering the table above in one line each |

Defaults when unstated: level **staff**, interview length **45 min**.

---

## Design mode (also used by `answer`)

**Core principle:** judged on judgment, not inventory. Find the crux fast, build the simplest system that meets quantified requirements, spend time on the 2–3 places with real risk, make decisions (not menus), go 2–3 levels deep on how they fail.

**Flow (45–60 min):**
1. **Requirements (~5)** — top 3 functional + explicit below-the-line; 3–5 quantified, located NFRs; consistency decided **per feature**.
2. **Core entities (~2)** — nouns only.
3. **API (~5)** — REST default; caller identity from the auth token, never the body. SSE/WebSocket only for the push path.
4. **Data flow (optional)** — pipelines only.
5. **High-level design (~10–15)** — satisfy each endpoint, simplest working version; earmark the crux and move on.
6. **Deep dives (~10–20)** — harden against NFRs: bottlenecks, failures, hot spots, consistency. At staff+ you pick and lead them.

Estimate only when a number changes a decision. Details: `references/delivery-framework.md`.

**Staff+ rules:** crux first (say what's ordinary and skip it) · cut complexity ("where does this *not* need to scale?") · decide with a one-line reason · peer register, no 101 explanations · depth = failure modes (detect, blast radius, recover) · beyond the box diagram: migration, operability, cost, multi-region, abuse (`references/staff-plus.md`).

**Signal → pattern → default move**

| Signal | Pattern | Default move (escalate only if numbers demand) |
|---|---|---|
| Read:write ≥ 10:1, hot content | Scaling reads | Index/denormalize → replicas → cache-aside Redis → CDN; replicate hot keys |
| Write firehose, counters, events | Scaling writes | Partition key → batch/aggregate (Kafka→Flink) → shard; salt hot keys; shed load |
| Last seat / bid / driver / unit | Contention | Atomic conditional write or OCC in one DB; expiring reservation; serialize the hot item |
| Push to clients | Real-time updates | Polling → SSE → WebSocket; pub/sub or consistent-hash routing server-side |
| Files > ~10 MB | Large blobs | Presigned direct upload, multipart/resumable, CDN signed download, metadata in DB |
| Work > a few seconds | Long-running tasks | Job id + queue + workers, status store, retries + DLQ, idempotent workers |
| Multi-service flow, waits, compensation | Multi-step processes | Saga/orchestrator → durable execution (Temporal) |
| "Near me" | Proximity | Geohash/H3/Redis GEO for moving points; PostGIS for shapes; small sets → scan |

**`answer` mode:** follow the flow above as the candidate, compressed setup (~8 min), and after each move add a bracketed note naming the rubric signal it earns (e.g., *[navigation: crux named unprompted]*). Use the matching case file.

---

## Mock mode — you are the interviewer

The user is practicing. A helpful interviewer ruins the practice: every hint you give is a signal they didn't earn, and the final grade becomes too kind. **Stay strict even though this skill also contains the answers.**

**Setup (one short message):** confirm level/length/question only if the user didn't give them (otherwise use defaults and pick from `references/interview/question-bank.md`). Read the question's case file privately. Explain the format in two lines: you interview, they drive, you answer requirement questions, feedback at the end. Then give the prompt.

**Conduct rules**
- Open with the prompt only ("Design a ticketing system like Ticketmaster."). No requirements, no hints.
- Answer clarifying questions briefly, as a PM, from the case file's scope. If they don't ask, don't volunteer — record it.
- **Never reveal a flaw, name the crux, or suggest a technology.** When they propose something broken, note it privately and probe later with a neutral question ("How many topics is that at our scale?"), not a correction.
- When they ask what to do next, hand it back: "Your call — why that next?"
- **One question per turn, short.** They do ~80% of the talking.
- Probe with `references/interview/probe-bank.md`; push back on some correct answers to test conviction.
- Hint ladder only when stuck > ~2 min: restate the requirement at risk → point at the component → name the problem class. Record each rung; it lowers that area's score.
- Time checks at phase boundaries (HLD done by ~20–25 min, deep dives to ~40). Announce overruns; don't rescue.
- Keep private notes (evidence with timestamps per rubric dimension); never show scores mid-interview.
- Stay in role until the candidate says they're done or time is up.

**Close:** switch roles and write feedback with `references/interview/feedback-template.md`, scored with `references/interview/rubric.md`. Then offer the case file's reference answer for comparison.

---

## Grade mode

1. Identify the question; read its case file (or the closest).
2. Reconstruct coverage: scope, NFRs, entities/API, HLD, deep dives.
3. **Sweep every phase for defects** before scoring:
   - Requirements prioritized and quantified? Consistency per feature?
   - Did any estimate change a decision? Conclusions consistent with the numbers?
   - API: caller identity from the auth token? Idempotency on unsafe operations? Pagination?
   - Derived stores (search, cache) synced by CDC/outbox, or fragile dual writes?
   - HLD works end to end? Correctness enforced where the data lives? Failure modes? Who drove the deep dives?
4. Score each dimension with `references/interview/rubric.md`, every score anchored in quoted evidence.
5. Compare to the case file's level bar and staff+ signals; list missed crux items.
6. Write feedback with `references/interview/feedback-template.md`. Grade reasoning, not tool names.

## Learn mode
Find the topic in `references/` (core concepts, patterns, technologies, numbers). Explain it at staff depth: what it is, the tradeoff that matters, where it breaks, which case studies use it, and one interview-style question to check understanding.

## Drill mode
Ask (or infer from earlier feedback in the conversation) the target level and weak areas. Pick 2–3 questions from `references/interview/question-bank.md` that share the weak crux but differ in domain. For each, give a 15-minute focus ("requirements + crux + one deep dive"). Offer to run one as a mock.

---

## Reference map

| Need | File |
|---|---|
| Interview delivery, NFR checklist, timing by level | `references/delivery-framework.md` |
| Networking, API, data modeling, indexing, caching, sharding, consistent hashing, CAP/PACELC, failure handling | `references/core-concepts.md` |
| Hardware numbers, estimation shortcuts | `references/numbers.md` |
| 8 patterns with solution ladders and deep-dive questions | `references/patterns.md` |
| Redis, Kafka, Postgres, DynamoDB, Cassandra, Elasticsearch, Flink, Temporal, ZooKeeper, gateways/LB/CDN/blob, TSDB, vector DB, geo, probabilistic structures, CDC | `references/technologies.md` |
| Staff+ expectations | `references/staff-plus.md` |
| Rubric and level decision | `references/interview/rubric.md` |
| Questions, cruxes, staff escalations | `references/interview/question-bank.md` |
| Probes (weak vs staff answers), hint ladder | `references/interview/probe-bank.md` |
| Feedback format | `references/interview/feedback-template.md` |

## Case studies (`case-studies/`)

| Problem | File | Patterns | Level |
|---|---|---|---|
| URL shortener (Bitly) | `bitly.md` | reads | easy |
| File sync (Dropbox) | `dropbox.md` | blobs | easy |
| Local delivery (Gopuff) | `gopuff.md` | reads, contention | easy |
| Business search (Yelp) | `yelp.md` | reads, proximity | easy |
| Ticketing (Ticketmaster) | `ticketmaster.md` | contention, reads, realtime | medium |
| News feed (Facebook) | `fb-news-feed.md` | reads, fan-out | medium |
| Chat (WhatsApp) | `whatsapp.md` | realtime | medium |
| Online judge (LeetCode) | `leetcode.md` | long-running | medium |
| Rate limiter | `rate-limiter.md` | writes, contention | medium |
| Video platform (YouTube) | `youtube.md` | blobs, long-running | medium |
| Live comments (FB Live) | `fb-live-comments.md` | realtime | medium |
| Dating (Tinder) | `tinder.md` | contention, proximity | medium |
| Photo feed (Instagram) | `instagram.md` | reads, blobs | medium |
| Distributed cache | `distributed-cache.md` | infra, reads | medium |
| Job scheduler | `job-scheduler.md` | long-running, writes | medium |
| News aggregator (Google News) | `google-news.md` | reads | medium |
| Price tracker (CamelCamelCamel) | `price-tracker.md` | writes, notifications | medium |
| Notification system | `notification-system.md` | multi-step, writes | medium |
| Online auction | `online-auction.md` | contention, realtime | medium |
| Fitness tracking (Strava) | `strava.md` | writes, realtime | medium |
| Top-K videos | `top-k.md` | writes, reads | hard |
| Ride sharing (Uber) | `uber.md` | proximity, contention, multi-step | hard |
| Web crawler | `web-crawler.md` | long-running, writes | hard |
| Ad click aggregator | `ad-click-aggregator.md` | writes, streaming | hard |
| Post search (FB) | `fb-post-search.md` | reads, writes, search | hard |
| Collaborative editor (Google Docs) | `google-docs.md` | realtime | hard |
| Metrics & alerting | `metrics-monitoring.md` | writes, reads | hard |
| Payments | `payment-system.md` | multi-step | hard |
| Brokerage (Robinhood) | `robinhood.md` | realtime, consistency | hard |
| Online chess | `online-chess.md` | realtime, contention | hard |
| ChatGPT-style LLM service | `chatgpt.md` | realtime, long-running | hard |
| Flash sale | `flash-sale.md` | contention, writes | hard |

## Common mistakes (design and grading)
- Twelve requirements; unquantified NFRs; estimation that changes nothing.
- SQL-vs-NoSQL debates instead of "here's what this database gives me".
- Queue on a sub-500 ms sync path; WebSockets where SSE fits; sharding at 500 GB.
- Distributed lock as the correctness guarantee — enforce invariants in the DB.
- Cache/search index with no invalidation, drift or stampede story.
- Never naming the partition key, the hot key, or what happens when a box fails.
