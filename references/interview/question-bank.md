# Question Bank

Reference answers live in this skill at `case-studies/<file>`. In mock mode, read the file before the interview starts and never show it to the candidate.

Columns: **Prompt** (what you say) · **Flavor** (P = product, I = infra) · **Crux** (what the candidate must find) · **Staff escalations** (constraints to add once the base design is solid, to find the ceiling).

## Easy (warm-up, or mid-level)

| File | Prompt | Flavor | Crux | Staff escalations |
|---|---|---|---|---|
| `bitly.md` | Design a URL shortener | P | unique short codes; fast redirects | multi-region writes; guessable codes; analytics without slowing redirects; 301 vs 302 |
| `dropbox.md` | Design Dropbox | P | large resumable uploads off the server path; delta sync | dedupe across users; conflict resolution offline; revoking shared links |
| `gopuff.md` | Design 1-hour local delivery availability + ordering | P | drive-time nearby DCs; consistent multi-item orders | inventory drift; holding carts; region failover |
| `yelp.md` | Design Yelp search + reviews | P | geo+text search; derived ratings | search by neighborhood polygons; review fraud; ranking with few reviews |

## Medium

| File | Prompt | Flavor | Crux | Staff escalations |
|---|---|---|---|---|
| `ticketmaster.md` | Design Ticketmaster | P | reservation without double booking; on-sale surge | 10M users one event; bots/scalpers; lock store outage; payment webhook ordering |
| `fb-news-feed.md` | Design Facebook News Feed | P | fan-out on read vs write; celebrities | ranking stage; edits/deletes in precomputed feeds; feed rebuild; storage sizing |
| `whatsapp.md` | Design WhatsApp | P | routing to connection holders; guaranteed delivery | multi-device; pub/sub drops; regionalization/cells; E2E encryption implications |
| `leetcode.md` | Design LeetCode | P | sandboxed execution; contest bursts | sandbox tech by threat model; contest fairness; per-test streaming |
| `rate-limiter.md` | Design a distributed rate limiter | I | atomic per-client state at 1M rps | Redis down: open vs closed; global limits across regions; dynamic rules |
| `youtube.md` | Design YouTube | P | ABR processing DAG; resumable uploads | transcode cost ladder; long-tail CDN strategy; pipeline versioning |
| `fb-live-comments.md` | Design Facebook Live comments | P | broadcast fan-out to millions | mega-stream mode switch; moderation latency; reconnect storms |
| `tinder.md` | Design Tinder | P | atomic mutual-swipe match; filtered stacks | hot profiles; Bloom filter for history; location privacy |
| `instagram.md` | Design Instagram | P | feed fan-out + media pipeline | publish ordering (process before fan-out); storage cost; moderation |
| `distributed-cache.md` | Design a distributed cache | I | consistent hashing, replication, hot keys, LRU internals | write-hot keys; proxy vs client routing; warm failover |
| `job-scheduler.md` | Design a distributed job scheduler | I | due-job discovery; 2 s precision; at-least-once | missed-fire policy; DST/cron semantics; tenant fairness |
| `google-news.md` | Design Google News | P | freshness SLA ingestion; cacheable feed | story clustering; personalization; breaking-news spike |
| `price-tracker.md` | Design CamelCamelCamel | P | prioritized crawling of 500M items; untrusted reports | legality/ToS; Prime-Day notification storm |
| `notification-system.md` | Design a notification system | I | never drop; priority isolation; dedupe | provider failover; campaign pacing; compliance |
| `online-auction.md` | Design an online auction | P | serialized bids; real-time high bid | sniping/end-time semantics; proxy bidding; durability on failover |
| `strava.md` | Design Strava | P | offline-first, batch uploads | live sharing; cheating detection; privacy zones |

## Hard (staff loops)

| File | Prompt | Flavor | Crux | Staff escalations |
|---|---|---|---|---|
| `top-k.md` | Design top-K most-viewed videos per window | I | 700k writes/s; exact windowed counts | sliding windows; approximation negotiation; OLAP tradeoffs; late events |
| `uber.md` | Design Uber | P | location writes + proximity; one-driver-one-ride workflow | stadium surge; batched matching; Redis partition; surge pricing cells |
| `web-crawler.md` | Design a web crawler (10B pages / 5 days) | I | durable frontier; politeness; dedupe; throughput math | priority/recrawl; giant-domain long pole; traps |
| `ad-click-aggregator.md` | Design an ad click aggregator | I | stream pre-aggregation; no loss; idempotency | hot ads; reconciliation; exactly-once semantics; late clicks |
| `fb-post-search.md` | Design Facebook post search (no Elasticsearch) | I | inverted index + like-churn writes | phrase queries; storage tiering; privacy filtering |
| `google-docs.md` | Design Google Docs | P | OT/CRDT convergence; doc-owner routing | hot document; server failover mid-session; offline |
| `metrics-monitoring.md` | Design Datadog-style metrics + alerting | I | write-heavy TSDB; sub-minute alerts | cardinality explosion; monitoring during incidents; percentiles |
| `payment-system.md` | Design a payment system | P/I | idempotency; unknown states with processors; audit | reconciliation; ledger; webhooks; PCI scope |
| `robinhood.md` | Design Robinhood | P | price fan-out; order state vs external exchange | market-open herd; cancel/fill races; buying power contention |
| `online-chess.md` | Design online chess | P | atomic matchmaking; authoritative clocks; rank at 10M | matchmaking beyond Postgres; lag compensation; game server failover |
| `chatgpt.md` | Design ChatGPT | P/I | resumable token streaming; GPU scheduling | tier fairness; long-context cost; model rollout |
| `flash-sale.md` | Design a flash sale | P | never oversell under extreme contention; admission | fairness/bots; gate vs truth reconciliation; isolating the main site |

## Choosing a question
- **Staff product loop:** Ticketmaster, Uber, WhatsApp, Google Docs, Flash Sale, ChatGPT.
- **Staff infra loop:** Top-K, Ad Click Aggregator, Metrics Monitoring, Rate Limiter (with escalations), Job Scheduler, Distributed Cache.
- **Test transfer:** after a practice round, pick a question that shares the crux but not the domain (Ticketmaster → Flash Sale or Online Auction; News Feed → Instagram; Top-K → Ad Clicks).

## Escalation technique
Once a base design works, change one constraint at a time ("now 100× the traffic on one item", "the cache cluster just failed", "legal says EU data stays in the EU", "budget cut 50%") and watch whether the candidate localizes the change or redesigns everything. Staff candidates localize and name the new crux.
