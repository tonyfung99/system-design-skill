# Design a Collaborative Editor (Google Docs)

Source: Hello Interview outline (premium); answers are independent · Patterns: real-time updates · Difficulty: hard

## Scope
- **FR:** create documents; multiple users edit concurrently; see others' changes in real time; see others' cursors and presence. Below the line: comments, permissions UI, offline mode, rich formatting detail.
- **NFR:** edits propagate < ~100–200 ms; edits never lost; eventual convergence (all replicas reach the same document); millions of concurrent editors, but ≤ ~100 per document; storage under control.

## Crux
Concurrent edits must converge (OT or CRDT), which needs a per-document authority or ordering point — so routing every editor of a document to the same stateful server is the architecture.

## Skeleton
Entities: Document (metadata), DocumentOperation (doc_id, version, op), Snapshot, Presence (ephemeral). API: `POST /docs`, `GET /docs/{id}` (snapshot + version), WebSocket `/docs/{id}` for ops, acks, cursor updates.
Document service (metadata, Postgres) + **Document servers** (stateful, own documents via consistent hashing) + op log store (Cassandra/DynamoDB partitioned by doc_id) + snapshot store (S3).

## Concurrency control
- **Operational Transformation (OT):** server is the single sequencer per document; clients send ops against a base version; server transforms against concurrent ops, assigns the next version, broadcasts; clients transform pending local ops. Proven (Google Docs), needs a central server.
- **CRDTs** (Yjs, Automerge, RGA/Logoot): commutative ops, converge without a central sequencer; great for offline/P2P; metadata overhead (tombstones) and harder intent preservation.
- For a centralized product, OT (or CRDT with a server relay) is the pragmatic answer — pick one and defend it.

Flow: client applies op locally (optimistic) → sends to owning doc server → server transforms, appends to op log (durable before ack), broadcasts to other connections → cursors/presence sent as ephemeral messages (not persisted).

## Deep dives

**1. Millions of WebSocket connections**
Connections must reach the server owning the document: consistent hashing of doc_id → server (ring in ZooKeeper/etcd); L4/L7 routing by doc id or a thin gateway forwarding to owners. Rebalancing on scale events: hand off ownership (flush state, new owner loads snapshot + tail of op log, clients reconnect). Heartbeats + reconnect with last version → server replays missed ops.

**2. Storage under control**
Op log grows forever → periodic **compaction into snapshots** (e.g., every N ops or when idle); keep recent ops for undo/version history; move old snapshots to cold storage; version history sampled (named versions kept).

## Staff+ signals
- State why sharding by document makes concurrency simple (single writer per doc) and what happens when one doc is very hot (hundreds of editors → read-only viewers via fan-out tier).
- Failure of a document server mid-session: durable op log before ack; new owner rebuilds; clients resend unacked ops idempotently (client op ids).
- Offline editing implications (pushes toward CRDTs).
- Permissions enforced at connect and on share changes (kick sessions).

## Traps
Stateless servers + "last write wins"; locking paragraphs; persisting cursor movements; no snapshotting.
