# Numbers to Know (modern hardware)

The biggest tell of book-only knowledge is reasoning from 2010–2015 hardware: sharding at 100 GB, fearing big objects in memory, caching everything. Today's single boxes are enormous; many "distributed" problems fit on one machine.

## Hardware envelope

| Resource | Rough modern ceiling (cloud) |
|---|---|
| RAM per instance | 512 GiB standard large; 4–24 TB memory-optimized |
| vCPU | 64–192+ |
| Local NVMe | tens of TB per instance (e.g., ~60 TB) |
| Network | 25 Gbps common, 50–100+ Gbps high-end |
| Object storage (S3) | effectively unlimited; 11 nines durability; don't discuss its scaling |

## Latency ladder

| Operation | Time |
|---|---|
| L1 / RAM access | ~1 ns / ~100 ns |
| Redis GET over network (same AZ) | ~0.2–1 ms |
| NVMe random read | ~10–100 µs |
| Sequential read 1 MB from SSD | ~0.1–1 ms |
| Round trip same AZ / cross-AZ | < 1 ms / 1–2 ms |
| Indexed DB point query (warm) | 1–5 ms; uncached/complex 10–50 ms |
| Cross-region / NYC↔London | 50–150 ms / ≥ ~70–80 ms (speed of light) |
| TCP+TLS handshake (far) | tens of ms per new connection → pool connections |

Random access is still slower than sequential on SSDs — the reason LSM trees, append-only logs and batching win for writes.

## Component capacities and scale triggers

| Component | Healthy single-unit capacity | Consider scaling out when… |
|---|---|---|
| Redis / Memcached node | 100k+ ops/s (more with pipelining), sub-ms, up to ~hundreds of GB–1 TB memory | hit rate < 80%, latency > 1 ms, memory > 80%, one key hot |
| Postgres/MySQL primary | 10k–50k simple TPS; several TB comfortably (managed limits ~64 TiB) | sustained writes > ~10k/s, working set ≫ RAM, multi-region needs |
| App server | 100k+ concurrent connections (WebSockets: 100k–1M with tuning) | CPU > 70%, latency > SLA, connection count near limit |
| Kafka broker | ~1M small msgs/s, TBs of retention; ~200k partitions/cluster ceiling | sustained ~800k msg/s, growing consumer lag |
| Elasticsearch | read-heavy; struggles with high update rates (e.g., like counts) | write amplification, refresh pressure |
| Vector index (HNSW) | 1M × 1536-d float32 ≈ 6 GB raw, ~2× with index; 1–10 ms queries | RAM, recall vs latency |

## Estimation shortcuts

- **~100k seconds/day** (86,400 rounded). 1M/day ≈ 10/s; 1B/day ≈ 10k/s; 70B/day ≈ 700k/s.
- Peak ≈ 2–10× average (event-driven: 100×+; design for the spike if the product has one).
- Row sizing: count only the big fields, round up (URL row ≈ 500 B → 1B rows ≈ 500 GB → *one* Postgres).
- ID+count for 4B items ≈ 16 B × 4B ≈ 64 GB → fits in RAM on one box (but think about HA).
- Connections: 200M online users / 100k–500k per host → 400–2,000 hosts.
- Heartbeats: 200M users / 10 s interval = 20M msgs/s — fine for tiny frames, not fine if each writes a DB row.

Do the math only to decide something; narrate the implication ("so a single heap won't hold it → shard by videoId").

## Common mistakes

- **Premature sharding**: 500 GB and 5k writes/s is one well-tuned primary + replicas.
- **Overestimating latency**: a cache is ~1 ms, not "fast-ish"; cross-region is the real cost.
- **Over-engineering writes**: 10k writes/s is within one Postgres; 1M/s needs batching/partitioning.
- **Ignoring cost**: DynamoDB at hundreds of thousands of writes/s gets expensive; RAM is the priciest place to store data; egress dominates video/CDN bills.
