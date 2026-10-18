# Design a Stock Brokerage (Robinhood)

Source: Hello Interview outline (premium); answers are independent · Patterns: real-time updates, consistency · Difficulty: hard

## Background
A broker doesn't run the exchange: it receives market data (price feed) and routes orders to an exchange/market maker via a FIX-style external API, which reports fills asynchronously.

## Scope
- **FR:** see live stock prices; create, cancel, and view orders (market/limit). Below the line: options, margin, portfolio analytics, deposits.
- **NFR:** prices fresh within ~1 s for millions of viewers; order placement highly consistent — no lost or duplicated orders, accurate status; high availability during market open spikes; ~20M users, a few thousand symbols.

## Crux
Fan-out of a small number of fast-changing values to millions of clients, and keeping internal order state consistent with an external exchange that you only talk to asynchronously.

## Skeleton
Entities: Symbol/Price, Order (user, symbol, side, type, qty, limit, status, external_order_id), Fill, Position. API: `GET /symbols/{s}`, SSE `/prices?symbols=…`, `POST /orders {idempotencyKey,…}`, `DELETE /orders/{id}`, `GET /orders?status&cursor`.
Market data processor (consumes exchange feed) → Redis (latest price per symbol) + pub/sub; Order service → Orders DB (Postgres/DynamoDB, partition by user) → Order gateway → exchange; fills/acks via exchange callback stream → trade processor.

## Deep dives

**1. Scale live price updates**
Only thousands of symbols: the *values* are tiny; the *fan-out* is the problem. Clients open SSE with their watchlist; price servers subscribe to per-symbol channels (Redis pub/sub) only for symbols their clients watch; throttle/coalesce updates per client (latest value wins, e.g., ≤ 1/s per symbol). Non-active symbols: poll.

**2. Track order updates**
Store `external_order_id` ↔ internal order id; the exchange sends execution reports (acks, partial fills, cancels) to a trade processor that updates order status idempotently (exec ids), appends fills, updates positions; notify the user (SSE/push).

**3. Order consistency**
- Write order row (status `pending`) **before** sending to the exchange — if we crash after sending, we know the order exists.
- Idempotency key from client; client order id (ClOrdID) to the exchange so retries don't duplicate.
- On timeout/unknown: query the exchange for order status before retrying.
- Cancels race with fills — final state comes from the exchange's report; UI shows "cancel requested".
- Reconcile against exchange/clearing end-of-day reports.

## Staff+ signals
- Distinguish quote freshness (AP, best-effort, coalesced) from order state (CP, durable, reconciled).
- Market-open thundering herd: pre-scale, admission control, queue orders with fairness.
- Regulatory: best execution, audit trail (append-only order events), trading halts.
- Buying-power checks are a contention problem (reserve funds atomically on order placement).

## Traps
Building an exchange/matching engine when asked for a broker; WebSockets for read-only price streams; sending to the exchange before persisting; treating timeouts as failures.
