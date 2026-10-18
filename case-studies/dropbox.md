# Design File Storage & Sync (Dropbox / Google Drive)

Source: full breakdown · Patterns: large blobs · Difficulty: easy

## Scope
- **FR:** upload from any device; download from any device; share files with users and view shared files; automatic sync across devices. Below the line: version history UI, editing, quotas.
- **NFR:** availability > consistency; files up to 50 GB; secure and recoverable; low-latency upload/download/sync.

## Crux
Moving very large files reliably (resumable, parallel, verified) without routing bytes through your servers, and keeping devices in sync cheaply (delta sync).

## Skeleton
Entities: File (id, owner, name, size, status, s3_key, chunks[]), FileShare, Device. API: `POST /files` (metadata → presigned URLs), `GET /files/{id}` (→ signed CDN URL), `POST /files/{id}/shares`, `GET /files/changes?since=cursor`.
Metadata DB (Postgres/DynamoDB) + S3 + CDN. Client sync agent watches the local FS and polls/subscribes for remote changes.

## Deep dives

**1. Large files (50 GB)**
- Single POST fails: timeouts, no resume, no progress. Chunk into 5–100 MB parts via S3 multipart; upload parts in parallel; fingerprint each chunk (hash).
- ◐ Client PATCHes chunk status to the backend — trusts the client (could mark unuploaded chunks done).
- ✓ **Server-side verification:** client reports part numbers + ETags; backend verifies with `ListParts`; completion via `CompleteMultipartUpload` triggers an `ObjectCreated` event that flips status to uploaded. Resume = fetch file metadata, upload missing parts only. Lifecycle rule aborts stale incomplete uploads.

**2. Fast upload/download/sync**
- Download via CDN (signed URLs), range requests, parallel.
- Upload: parallel parts, adaptive chunk size.
- Sync: only changed chunks move. Fixed-size chunks break on insertions (every boundary shifts) → **content-defined chunking** (rolling Rabin hash) so edits only touch nearby chunks. Chunk-level dedupe across files/users by fingerprint.
- Compression client-side, only when it pays (skip already-compressed media).
- Change notification: cursor-based `GET /changes` polling (simple) or SSE/WebSocket push for active clients; conflicts → keep both ("conflicted copy") or last-writer-wins by policy.

**3. Security**
HTTPS in transit; S3 server-side encryption at rest; ACL via share table checked on every signed-URL request; signed URLs are bearer tokens → short expiry (minutes), optionally IP-bound or combined with auth cookies.

## Staff+ signals
- Dedupe economics (content-addressed chunk store, refcounts, GC of unreferenced chunks).
- Metadata vs bytes consistency: status state machine + reconciliation sweeper.
- Sharing at scale: "shared with me" query → index by recipient; permission changes revoke outstanding URLs (short TTL).
- Conflict resolution and offline edits.
- Cost tiers (hot → infrequent access → archive by last access).

## Traps
Bytes through API servers; trusting client-reported upload state; fixed-size chunking for delta sync; long-lived download URLs.
