# Design a Chat App (WhatsApp / Messenger)

Source: full breakdown · Patterns: real-time updates · Difficulty: medium

## Scope
- **FR:** group chats (≤ 100 participants); send/receive messages; receive messages sent while offline (≤ 30 days); media attachments. Below the line: calls, E2E key management detail, reactions.
- **NFR:** delivery < 500 ms to online users; guaranteed delivery; billions of users; don't keep messages on servers longer than needed; resilient to component failures.

## Crux
Routing a message to whichever server holds each recipient's connection, at billions of connections, without losing messages when anything in the middle fails.

## Skeleton
Entities: User, Client (device), Chat, ChatParticipant, Message, Inbox (undelivered message ids per client). Transport: WebSocket per client. Commands: `createChat`, `sendMessage`, `createAttachment` (presigned URL), `modifyParticipants`; server→client `newMessage`, `chatUpdate`; client acks.
Flow: Chat server receives → persist Message + Inbox rows for each recipient client (DynamoDB/Cassandra) → ack sender → push to online recipients → client ack → delete Inbox row. Offline clients drain Inbox on connect. TTL deletes old messages.

## Deep dives

**1. Billions of simultaneous users**
- ✗ Plain horizontal scaling — the receiving server doesn't hold recipients' sockets.
- ✗ Kafka topic per user — Kafka isn't built for billions of topics (~tens of KB overhead each).
- ◐ Consistent hashing of users to chat servers (ring in ZooKeeper/etcd); servers forward to owners; full-mesh inter-server connections; rebalancing on scale events.
- ✓ **Redis Pub/Sub** channel per user id: chat servers subscribe for their connected users; senders publish. Lightweight, decoupled; at-most-once, single-digit ms extra latency → durability comes from the Inbox.

**2. Multiple devices per user**
Clients table; Inbox per *client*; deliver to all clients of each participant; cap devices per account; deactivate stale clients.

**3. Dead WebSockets**
- ✗ Rely on TCP keepalive (minutes).
- ◐ Ack timeouts with server retry — only detects during sends.
- ✓ App-level heartbeats every 10–30 s; close on missed pong; client reconnects and syncs from Inbox. (200M users / 10 s = 20M tiny frames/s — fine.)

**4. Pub/Sub drops a message**
◐ Periodic client sync polls (cost ∝ connected users). ◐ Per-chat sequence numbers + gap detection (needs a next message to notice). ✓ **Piggyback the latest per-user sequence on heartbeats** → client detects gaps within one interval and resyncs.

**5. Out-of-order messages**
Don't engineer strict ordering; stamp server-receive time (NTP-synced servers) and display by it; occasional pop-in is acceptable to users.

**6. "Last seen"**
✗ DB write per heartbeat (millions of writes/s). ✓ Derive from connection lifecycle: write on disconnect, report "online" while a connection exists (presence in Redis with TTL).

## Staff+ signals
- Go 2–3 levels deep on failure modes (server crash mid-send, Redis node loss, inbox write succeeded but push failed).
- Partitioning choice: messages by chat_id (history reads) vs inbox by client_id (delivery).
- Large groups/broadcast channels change fan-out math.
- Regionalization and cell-based architecture; E2E encryption implications (server can't index content; multi-device key distribution).
- Media: presigned upload, CDN download, separate retention.

## Level bar
Mid: functional API/HLD, rough scaling with awareness of flaws. Senior: scaling and robustness in detail, partition-by-chat vs by-user tradeoffs. Staff: 2–3 levels deep on failure modes; fault tolerance, DB optimization, regionalization, cells.
