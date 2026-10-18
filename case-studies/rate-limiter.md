# Design a Distributed Rate Limiter

Source: Hello Interview breakdown (full) · Patterns: scaling writes, contention, scaling reads · Difficulty: medium

## Scope
- **FR:** identify clients by user id, IP, or API key; enforce configurable rules (e.g., 100 req/min/user, per endpoint); reject with 429 + headers (`X-RateLimit-Remaining`, `Retry-After`/reset). Below the line: billing quotas, analytics.
- **NFR:** < 10 ms overhead per check; highly available (eventual consistency across nodes OK); 1M req/s, 100M DAU.

## Crux
Atomic read-modify-write of per-client state at 1M checks/s with single-digit-ms latency, and deciding what happens when the limiter itself fails.

## Skeleton
Placement: in the API gateway/edge (not each service). Rules in a config store, cached in gateways. State in Redis keyed by `{clientId}:{rule}`.

Algorithm choice (say why):
| Algorithm | Notes |
|---|---|
| Fixed window counter | `INCR`+`EXPIRE`; cheap; 2× burst at window edges |
| Sliding window log | sorted set of timestamps; exact; memory ∝ requests |
| Sliding window counter | weighted current + previous window; cheap, good approximation |
| **Token bucket** | (tokens, last_refill) in a hash, refilled lazily; allows controlled bursts; **default** |
| Leaky bucket | smooths output rate; queue semantics |

Token bucket in Redis via a **Lua script** (read, refill, decrement, write atomically — avoids the read-then-write race between gateways).

## Deep dives

**1. 1M req/s**
One Redis ≈ 100–200k ops/s; each check ≈ 2 ops → ~50–100k checks/s per node → **~10–20 shards** (Redis Cluster). Shard by client identifier so each client's state lives on one shard.

**2. Availability & fault tolerance**
◐ Fail-closed (reject when Redis is down) — takes the API offline, triggers retry storms. ◐ Fail-open — loses protection; if Redis failed *because* of load, everything floods downstream. Staff answer: fail-open with **local in-memory fallback limits** per gateway (approximate, conservative), replicas with fast failover, circuit breakers; choose per endpoint (fail-closed for expensive/abuse-prone endpoints like login/SMS).

**3. Latency**
Connection pooling (no per-check handshake); co-locate Redis with gateways per region (accept cross-region inconsistency); Lua to make it one round trip; avoid local caching of counters unless approximation is acceptable.

**4. Hot keys**
Legit heavy clients: client-side rate limiting via headers, batching APIs, premium tiers/dedicated capacity. Abuse: auto-blocklist after repeated violations (checked cheaply before Redis), DDoS protection at the edge (Cloudflare/Shield). Shared IPs (NAT): higher IP limits, prefer authenticated identities.

**5. Dynamic rules**
◐ Gateways poll config every ~30 s. ✓ Push via ZooKeeper/etcd watches or pub/sub for near-instant updates; keep last-known-good config locally.

## Staff+ signals
- Global limits across regions: per-region budgets (split quota) or async counter sync — exact global limits are rarely worth cross-region latency.
- Approximate local limiting (each of N gateways enforces limit/N) as a zero-network-hop design, and when it's good enough.
- Clock issues: use Redis server time (`TIME`) in the script.
- Observability: rejected-rate per rule, false-positive complaints, shadow mode for new rules.

## Traps
Non-atomic GET/SET; rate limiting inside each microservice; ignoring what happens when Redis dies; using WebSocket/queue anywhere on this path.
