# Delivery Framework

The most common reason candidates fail is not lack of knowledge — it's **not delivering a working system**, usually because time went to the wrong things. A fixed track also gives you somewhere to fall back to when nervous.

```
Requirements (5) → Core entities (2) → API (5) → [Data flow (5)] → High-level design (10–15) → Deep dives (10–20)
```

At staff+ the first four steps compress (aim ~8–10 min total) so most of the hour goes to deep dives you choose.

## 1. Requirements

### Functional — "Users/clients should be able to…"
- Ask like a PM: "Does it need X?", "What happens when Y?"
- Land on **the top 3**. Long lists hurt; many FAANG rubrics grade focus directly.
- Write an explicit **below-the-line** list (shows product sense, invites the interviewer to re-prioritize). Check in: "Anything you want moved above the line?"

### Non-functional — "The system should be…"
Pick the 3–5 that actually shape the design, each **in context and quantified**:
- Bad: "low latency". Good: "search returns in < 500 ms p99"; "redirect < 100 ms".

Checklist to scan:

| Dimension | Question to ask yourself |
|---|---|
| Consistency vs availability | Would a stale read be *catastrophic*? Decide **per feature** (booking = C, browsing = A). |
| Scale & shape | DAU, peak vs average, bursts (on-sale, breaking news, New Year), read:write ratio, fan-out |
| Latency | Which path, what percentile, what target |
| Durability | Can we lose anything? (social like: maybe; payment: never) |
| Fault tolerance | What must keep working when a dependency dies? |
| Environment | Mobile battery/bandwidth, offline, devices, regions |
| Security / compliance | Authn/z, PII, PCI, GDPR, data residency, abuse |

### Capacity estimation
Skip it upfront; say you'll estimate **when it decides something**. Examples where it does: Top-K (does the counter set fit in one heap?), rate limiter (how many Redis nodes?), crawler (machines to finish in 5 days), chat (connections per host). "It's a lot" is not a conclusion. Numbers: `numbers.md`.

## 2. Core entities
Bulleted nouns the API exchanges and the DB persists (User, Event, Ticket, Booking). Not columns yet — you'll discover entities while designing. Good names matter; some interviewers watch for it. Ask: who are the actors? what resources satisfy the FRs?

## 3. API / system interface
- **REST** by default (plural resource nouns, HTTP verbs). **GraphQL** if many clients with divergent data needs. **gRPC** for internal latency-critical service calls.
- Current user comes from the auth token (JWT/session) — never `userId` in the body.
- Mention pagination (cursor for live data), idempotency keys for unsafe retries, rate limits — briefly.
- Real-time? Design the core REST API first, then add SSE/WebSocket for the push channel.
- 4–5 endpoints in a couple of minutes. Still on APIs at minute 10 = too deep.

```
POST /v1/events/{id}/bookings  { ticketIds }  -> Booking
GET  /v1/events/{id}           -> Event + Venue + seat map
GET  /v1/events?q=&start=&end=&cursor=
```

## 4. Data flow (optional)
For pipeline problems only: a numbered list (fetch → parse → extract URLs → store → enqueue). Drives the HLD.

## 5. High-level design
- Boxes and arrows that satisfy the API, **one endpoint at a time**.
- Narrate the request path and **what state changes** at each hop.
- Write only the *interesting* columns next to each DB (status, expiresAt, partition/sort keys). Nobody needs `email` on User.
- Simple first. When you see a future problem (cache, queue, fan-out), say it, note it, move on. Layering complexity early is the #1 way to never finish.
- Load balancer: draw one at the front or just say "stateless, horizontally scaled". API gateway: one box, "routing + auth + rate limiting", move on.

## 6. Deep dives
Use the remaining time to:
- Prove each NFR is met (scale, latency, consistency, durability).
- Resolve the earmarked crux.
- Handle edge cases and failure modes.
- Respond to interviewer probes.

Proactivity scales with level: mid-level can be driven by the interviewer; senior should surface most issues; **staff+ should lead, pick 2–3 areas, and the interviewer should only need to focus, not steer**. Leave room for the interviewer to probe — they're hunting for specific signals; talking over them costs you on collaboration.

Update the diagram as you go. Interviewers write feedback the next day from your board; make it legible.

## Time-boxing by level

| Phase | Mid | Senior | Staff+ |
|---|---|---|---|
| Reqs + entities + API | 12–15 min | ~10 min | ~8 min |
| HLD | 15 min | 10–15 min | ~10 min, crux earmarked |
| Deep dives | interviewer-led, 1–2 | candidate surfaces most, 2+ | candidate-led, 2–3+, multiple levels deep |
| Breadth : depth | 80 : 20 | 60 : 40 | 40 : 60 |
