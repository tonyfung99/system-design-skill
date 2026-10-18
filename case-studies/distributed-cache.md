# Design a Distributed Cache (Redis/Memcached-like)

Source: Hello Interview outline (premium); answers are independent · Patterns: infrastructure, scaling reads · Difficulty: medium

## Scope
- **FR:** `set/get/delete` key-value pairs; per-key TTL; LRU eviction. Below the line: complex data types, persistence, transactions.
- **NFR:** highly available, fault tolerant; scales to ~1 TB data and ~100k+ req/s (state the numbers); < 10 ms (target ~1 ms) latency; eventual consistency acceptable.

## Crux
This is an infra interview — consistent hashing, replication, hot keys and the node internals *are* the deep dives. Don't hide them behind "use Redis".

## Skeleton
Per node: hash map + doubly linked list (O(1) LRU) + TTL handling (lazy check on read + periodic active sampling sweep). Cluster: N nodes; clients (or a proxy tier) route by consistent hashing; config service (ZooKeeper/etcd) holds membership.

## Deep dives

**1. High availability & fault tolerance**
Replicate each shard: primary + replicas. Async replication (fast, may lose recent writes on failover) vs sync (durable, slower) — for a cache, async is usually right. Failure detection via heartbeats/gossip; failover promotes a replica (consensus/coordinator avoids split brain). Optional snapshots/AOF to warm restarts.

**2. Scalability**
Do the math: 1 TB / ~64–256 GB usable per node → ~8–16 primaries (+ replicas); 100k req/s per node is comfortable, so memory is the binding constraint. Add nodes → rebalance only affected ranges.

**3. Even key distribution**
Consistent hashing with **virtual nodes** (100s per node) — or fixed hash slots (Redis Cluster's 16,384). Moving keys on membership change: migrate slot ranges in the background; clients follow redirects (`MOVED`) or refresh topology.

**4. Read-hot key**
Replicate the key to several nodes (`key#1..k`, readers pick randomly), serve reads from replicas, client-side near-cache with short TTL, request coalescing. Detect hot keys with sampling / count-min sketch.

**5. Write-hot key**
Harder: replicas don't help. Batch/coalesce writes client-side, shard the value (split counters into k sub-keys and sum on read), or rate-limit the writer. Question the use case (is a cache the right home?).

**6. Performance**
Single-threaded event loop per core (or shared-nothing threads per shard), pipelining/batching, connection pooling, avoid large values (fragmentation), memory allocator choice (jemalloc slab classes), compact encodings, eviction sampling (approximate LRU).

## Staff+ signals
- Articulate client-side routing vs proxy (Twemproxy/Envoy/mcrouter): fewer hops vs simpler clients and central control.
- Thundering herd on cold node replacement → warm from replica before taking traffic.
- Consistency semantics clearly stated (no read-your-writes across failover).
- Memory overhead per key, eviction policy tuning (LFU for stable popularity), observability (hit rate, evictions, latency).

## Traps
Modulo hashing; ignoring rebalancing; claiming replicas solve write-hot keys; O(n) eviction.
