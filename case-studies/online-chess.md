# Design Online Chess (chess.com / lichess)

Source: Hello Interview outline (premium); answers are independent · Patterns: real-time updates, contention · Difficulty: hard

## Scope
- **FR:** skill-based matchmaking and game start; real-time play; global leaderboard with own rank. Below the line: chat, analysis engine, tournaments, spectators.
- **NFR:** a player is never matched into two games; move latency < ~100 ms; fair clocks despite network jitter; 500K concurrent games; leaderboard for 10M players, fast and correct.

## Crux
Atomic matchmaking, stateful game servers with authoritative clocks, and an exact rank query over 10M players.

## Skeleton
Entities: Player (rating), MatchRequest, Game (players, moves, clocks, status), Rating history. Matchmaking service; Game servers (WebSocket, one owner per game); Game DB (moves appended); Leaderboard (Redis sorted set).

## Deep dives

**1. Never matched into two games**
Matchmaking queue bucketed by rating band (and time control); pairing must atomically claim both players: Postgres `SELECT … FOR UPDATE SKIP LOCKED` on queued requests (claim two, mark matched, create game in one transaction) — or a single-writer matcher per bucket (partitioned consumers), or Redis Lua that pops two compatible entries atomically. Widen rating band with wait time.
*If matchmaking outgrows Postgres:* partition by (time control, rating band) so each partition has a single matcher; edge-of-band players handled by periodic cross-band sweeps.

**2. 500K concurrent games**
Each game owned by one game server (consistent hashing on game_id; registry in etcd); both players' WebSockets routed there; moves validated server-side, appended durably (Cassandra/DynamoDB per game, or batched) before broadcast; ~500K games × move every few s ≈ 100k+ moves/s — fine sharded. Server failure → new owner rebuilds from the move log; clients reconnect.

**3. Fair clocks despite latency**
Server is authoritative; measure each client's RTT and apply **lag compensation** (credit back estimated network time, bounded per move); clients render local countdown for UX; flag games on suspicious timing.

**4. Correct, fast leaderboard at 10M players**
Redis sorted set (`ZADD` on rating change, `ZREVRANK` for own rank — O(log n)); top N via `ZREVRANGE`. Durable ratings in DB; rebuild the set from DB if lost. If it must shard: bucket counts by rating range to compute approximate global rank, or keep one big set (10M entries ≈ ~1 GB — fits on one node).

## Staff+ signals
- Single authoritative owner per game makes ordering trivial — argue it explicitly.
- Rating update (Glicko/Elo) transactional with game result; idempotent on game id.
- Disconnect/abandon policies; anti-cheat (engine detection) async.
- Show that 10M-entry leaderboard fits one Redis node before proposing sharding.

## Traps
Matching with read-then-write races; client-authoritative clocks; peer-to-peer moves; sharded leaderboard on day one.
