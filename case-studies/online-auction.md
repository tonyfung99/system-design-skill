# Design an Online Auction (eBay)

Source: outline only; answers are independent · Patterns: contention, real-time updates · Difficulty: medium

## Scope
- **FR:** post an item with starting price and end time; bid (accepted only if higher than current highest); view an auction with the current highest bid. Below the line: proxy/auto-bidding, payments, search.
- **NFR:** **strong consistency for bids** (no two "winning" bids, no lost accepted bid); fault tolerant and durable; current highest bid shown in real time; 10M concurrent auctions; bursty sniping at the end.

## Crux
Serializing concurrent bids on the same item correctly and durably, then broadcasting the new high bid to all watchers quickly.

## Skeleton
Entities: Auction (item, start_price, end_time, max_bid, max_bidder, version, status), Bid (auction_id, user, amount, created_at). API: `POST /auctions`, `POST /auctions/{id}/bids {amount}`, `GET /auctions/{id}`, SSE `/auctions/{id}/stream`.

## Deep dives

**1. Strongly consistent bids**
- ✗ Read max, compare in app, write — race.
- ◐ Pessimistic `SELECT … FOR UPDATE` on the auction row, insert bid, update max — correct, serializes per item (fine; contention is per auction).
- ✓ **Conditional update / OCC:** `UPDATE auctions SET max_bid=:a, max_bidder=:u WHERE id=:id AND max_bid < :a AND end_time > now()` + insert Bid in the same transaction; rows affected = 0 → reject with current max. Single statement, no lock held across app logic.
- Alternative at extreme contention: route all bids for an auction to one partition (Kafka key = auction_id) and process sequentially; the consumer is the single writer.

**2. Fault tolerance & durability**
Never lose an accepted bid: persist before acking (DB commit or Kafka `acks=all` then process). If using a queue-first design, give the bidder an async result (pending → accepted/rejected) and make processing idempotent (bid id). Replicated DB with synchronous standby for zero-loss failover.

**3. Real-time highest bid**
SSE to watchers; on accepted bid publish to pub/sub channel `auction:{id}`; servers holding watchers forward. Popular auctions: co-locate watchers; throttle UI updates (latest value wins, so coalescing is safe).

**4. 10M concurrent auctions**
Shard auctions by auction_id; contention is per item, so sharding parallelizes cleanly. Auction end: timer service / scheduled job closes auctions (status transition guarded by condition), determines winner, kicks off payment workflow.

## Staff+ signals
- End-time semantics: bids at the deadline — use server receive time; consider anti-sniping extension (product decision).
- Clock: single authority (DB `now()`), not app servers.
- Proxy bidding turns bid processing into a small state machine — still single-writer per auction.
- Abuse: shill bidding detection, bid retraction rules.

## Traps
Distributed lock as the correctness mechanism; caching the current max as source of truth; WebSockets when SSE + POST suffices.
