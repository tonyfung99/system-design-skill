# Rubric

Company rubrics use different words but converge on four dimensions. Score each 1–4 **relative to the target level**, citing evidence. Then decide the level.

Scale: **1** clear gap · **2** below bar (some signal, significant gaps) · **3** at bar · **4** above bar (strong, would raise the hire bar).

## 1. Problem navigation
Breaking an ambiguous problem down, prioritizing what matters, making steady progress, delivering a working system. Often the most important dimension.

| Level | At bar looks like |
|---|---|
| Mid | Asks clarifying questions, lands a focused requirement list, follows a structure, completes a working design with some guidance |
| Senior | Quantified NFRs tied to features; spots the main challenges; moves through basics fast; manages own time |
| Staff | Scopes decisively (top 3 + explicit below-the-line); names the crux in the first ~10 min and earmarks it; spends most time on highest-risk areas; consistency decided per feature; asks where scale is *not* needed |
| Principal | Reframes the problem (product/organizational levers), identifies multi-year risks, sets the agenda for the conversation |

Failure signals: shallow requirement gathering; long feature lists; rabbit-holing on trivia (user table fields, LB algorithms); stuck without a way forward; **no working end-to-end system**.

## 2. Solution design
Designing each piece and composing a coherent whole that meets the requirements and scales.

| Level | At bar looks like |
|---|---|
| Mid | Sensible components; data flows clear; reaches "good" solutions with probing |
| Senior | Reaches "great" solutions for 2+ deep dives; explicit tradeoffs; identifies bottlenecks proactively |
| Staff | Simplest design that meets quantified needs; every component justified; failure modes analyzed 2–3 levels deep (detection, blast radius, recovery); correctness under concurrency/retries; covers deep dives the interviewer didn't plan |
| Principal | Evolution path (v1 → v3), migration strategy, multi-region/cell design, cost model as first-class constraints |

Failure signals: ignoring scale/perf; "spaghetti" design; over-engineering (distributed everything at modest scale); queues/caches added without reason; correctness hand-waved ("the lock handles it").

## 3. Technical excellence
Knowing current technologies, patterns and best practices, and applying them correctly.

| Level | At bar looks like |
|---|---|
| Mid | Knows a reasonable option per category at a high level (ES is a search index, Kafka is a log) |
| Senior | Explains mechanics (inverted index, partitions/consumer groups, LSM vs B-tree, cache stampede) and limits; recognizes standard patterns |
| Staff | Practical, experience-backed depth: configuration that matters, operational failure modes, modern hardware numbers; knows when *not* to use a tool; uses estimation to decide, not to decorate |
| Principal | Can design the underlying system itself (the cache, the queue, the consensus layer) and critique industry defaults |

Failure signals: unaware of standard tech; outdated constraints (sharding at 100 GB); misapplying tech (Kafka topic per user, ES as source of truth, Redis lock as correctness guarantee); not recognizing the pattern.

## 4. Communication & collaboration
Clarity, responsiveness to feedback, and what it would be like to work together.

| Level | At bar looks like |
|---|---|
| Mid | Explains clearly; accepts hints; diagram readable |
| Senior | Structured narration; checks in at transitions; defends or updates positions with reasoning |
| Staff | Peer register (no 101 explanations); high-bandwidth; **makes decisions** instead of listing options; gives the interviewer room to probe; the interviewer learns something |
| Principal | Shapes the discussion; crisp framing of tradeoffs for non-specialists and specialists alike |

Failure signals: can't explain complex ideas; defensive or argumentative; talks over the interviewer; option menus without choice; lost in the weeds.

## Level decision

- **Hire at target level:** no dimension at 1; at most one at 2 (and not Problem navigation); at least one 4 for staff+.
- **Down-level:** solid at the level below (e.g., senior-bar evidence in a staff loop: good deep dives, but interviewer-driven, shallow failure analysis, option lists).
- **No hire:** no working system, or any dimension at 1 at the level below target.

Staff-specific must-haves (absence of any two → not staff): crux identified unprompted · simplicity with justified escalations · decisions with reasons · failure modes beyond the happy path · candidate-led deep dives.

## Red flags at any level
- Memorized answer that collapses under a "why?" or a changed constraint.
- SQL-vs-NoSQL generalities as justification.
- Capacity math that doesn't change any decision, or none when it clearly would.
- Treating a cache, search index, or replica as consistent without saying so.
- Ignoring idempotency on retried, money-moving or counting paths.
- Caller identity taken from the request body or path instead of the auth token.
- Derived stores (search index, cache, warehouse) synced by dual writes with no story for partial failure.

## Evidence capture template (fill during the interview)

```
[mm:ss] dimension  +/-  quote or observation
[03:10] NAV  +  "Top 3 are view, search, book; dynamic pricing below the line"
[18:40] SOL  -  Redis lock presented as the only double-booking protection
[27:05] TECH +  explained ES refresh interval → near-real-time search, CDC lag
[33:30] COMM -  listed 3 queue options, didn't pick until asked
```
