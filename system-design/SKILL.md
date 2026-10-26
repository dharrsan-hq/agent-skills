---
name: System Design
description: System design principles distilled from Designing Data-Intensive Applications — storage engines, replication, partitioning, consistency, consensus, and stream processing tradeoffs
---

# System Design

Authoritative source: *Designing Data-Intensive Applications* (Martin Kleppmann).
Apply the rules and tradeoff tables below when designing or reviewing
data-intensive architectures.

## Core mental model

- **A data-intensive system's primary challenge is not computation at scale, but
  the amount and complexity of data — how to store it, process it, distribute it.**
- **Reliability, scalability, and maintainability are the three concerns.**
  - Reliability: the system continues to work correctly (faults happen;
    tolerating them is the design goal, not preventing them).
  - Scalability: describe load with numbers (QPS, hit rate, dataset size) —
    never " scalable" without a load profile. Architecture that works at 100k
    QPS may fail at 10M.
  - Maintainability: operability (easy to operate), simplicity (abstractions,
    not accidental complexity), evolvability (change-friendly).
- **Latency is a percentile problem, not an average problem.** Tail latency
  (p95/p99/p99.9) matters because the slowest requests are often the ones with
  the most data (heavy users) — and in serial chains, a single slow request
  propagates. Head-of-line blocking, queueing, and GC pauses create tails.
- **Most systems are not "big data"; they are "big complexity."** DDIA's lens:
  single-node systems that scale vertically are fine until they aren't —
  distributed systems bring costs that only pay off at scale.

## Data models

- **Relational model:** data as relations; SQL; joins; good for normalized,
  structured data; strong declarative query language (query optimizer decides
  *how*).
- **Document model (MongoDB, CouchDB):** schema-flexible JSON-like docs.
  Pros: locality (whole object in one place, one read), matches app structure.
  Cons: joins are weak (denormalize or multi-query), referential integrity
  is the app's problem, updates are awkward for deeply nested data.
- **Graph model (Neo4j, Datomic):** everything is vertices and edges; good for
  highly connected data (social networks, recommendations, fraud). Local,
  index-free adjacency makes traversals O(1) per hop.
- **Key-value / wide-column (Dynamo, Cassandra, Bigtable):** simple, scalable,
  partition-tolerant. No joins, no secondary indexes in the general case.
- **Schema-on-write vs schema-on-read:** relational = schema-on-write (strict);
  document = schema-on-read (flexible). Schemas are documentation, enforced
  where possible — a schema is a contract, not a burden.
- **Rule of thumb:** pick the model by *access patterns*, not by fashion.
  If the app is "load one object, show it" → document. If it's "query many
  entities by relation" → relational or graph. If it's "get/put by key at
  extreme scale" → key-value.

## Storage engines

- **Two dominant families: B-trees (read-optimized) and LSM-trees
  (write-optimized).** Most databases are one of these under the hood.

| Aspect | B-tree (e.g., MySQL/InnoDB, Postgres) | LSM-tree (e.g., LevelDB, RocksDB, Cassandra) |
|---|---|---|
| Write path | In-place update, write-ahead log first | Append to memtable + WAL, flush to SSTables |
| Read path | O(log n) tree lookup, single page | Check memtable + all SSTables (newest → oldest) |
| Write amplification | Lower (overwrite one page) | Higher (compaction rewrites) |
| Read amplification | Lower (one lookup) | Higher (may check many levels) |
| Compaction | Background page split/merge | Background merge of SSTables |
| Fragmentation | Some (page splits) | None while writing (append-only) |
| Best for | Read-heavy, point lookups | Write-heavy, high ingest |

- **LSM-trees are the default for write-heavy workloads** (analytics, time-series,
  messaging); **B-trees remain the default for read-heavy OLTP** (Postgres,
  MySQL).
- **Bloom filters** mitigate LSM read amplification: cheaply check "does this
  key *possibly* exist in this SSTable?" — skip it if definitely not.
- **Hash indexes** (key → offset) are the simplest: fast, but range queries are
  impossible. Only for pure key-value workloads.
- **Column-oriented storage** (Parquet, ClickHouse, Druid): store values of each
  column together. Excellent for analytics: high compression ratios, scan only
  needed columns. Row-oriented storage is the default for OLTP.
- **Aggregation and materialized views**: precompute expensive results; accept
  staleness; data cubes are a common OLAP shortcut.
- **Time-series databases** are LSM-based with aggressive compression and
  retention policies; they excel at "append a lot, read by range, aggregate."

## Encoding and evolution

- **Data formats outlive code.** Design for schema evolution: field
  additions/renames must be backward *and* forward compatible.
- **Language-specific formats** (Java serialization, Python pickle) are a
  security and compatibility hazard — never use them across trust boundaries.
- **JSON, XML, CSV:** human-readable but ambiguous (numbers, binary, unicode).
  JSON is fine for external APIs; binary schemas (Protobuf, Avro, Thrift) are
  better for internal service-to-service.
- **Protobuf:** required/optional fields; never reuse a removed field number —
  that's the compatibility contract. **Avro:** schema registry is the norm
  (producer and consumer look up the writer schema).
- **The flow of data:**
  - Via a database: old readers must tolerate new writers' data and vice versa
    (rolling upgrades; add columns as nullable).
  - Via services: REST/RPC with explicit versioned schemas; gRPC/Protobuf is
    a good default.
  - Via message passing: log-based (Kafka) or broker-based (RabbitMQ).
- **Message brokers vs logs:** a broker deletes messages after delivery
  (temporary); a log (Kafka) retains them indefinitely (replayable). Logs are
  the backbone for event-driven systems; use log-based integration when
  consumers need to replay or when multiple consumers need the same stream.

## Replication

- **Three replica problems:**
  1. **Leader-based replication** (single-leader): writes go to the leader;
     replicas follow the change log (Postgres, MySQL, MongoDB default). Most
     common; simple but write bottleneck and failover complexity.
  2. **Multi-leader replication**: writes to any of several leaders. Useful
     across data centers or offline clients; introduces *conflict resolution*
     as a first-class problem.
  3. **Leaderless replication** (Dynamo-style): writes go to any of N replicas;
     quorum reads/writes (w + r > n). Examples: Cassandra, Riak, Voldemort.
- **Replication modes:** **synchronous** (leader waits for replica ack — safe
  but slow/fragile), **asynchronous** (fast but risks data loss on leader
  failure — replication lag), **semi-sync** (one replica syncs).
- **Replication lag problems** (async replicas): read-your-own-writes violations,
  monotonic-read violations, consistent-prefix violations. Mitigations:
  - Read-your-own-writes: read from the leader for a short window after write,
    or track the last write timestamp and route reads to a replica that's
    caught up.
  - Monotonic reads: always read from the same replica (sticky per user).
- **Failover:** leader failure requires automatic detection (heartbeat
  timeout) and election — the risks are lost writes (async replication) and
  split brain (two leaders). Use a quorum for failover decisions.
- **Multi-leader conflicts:** convergent resolution strategies —
  last-write-wins (LWW, loses data), merging (union), or application-level
  conflict resolution (most correct). CRDTs (conflict-free replicated data
  types) converge automatically.
- **Topologies:** single leader, leader with trees, multi-leader with full mesh
  (all-to-all) — full mesh risks write cycles.

## Partitioning (sharding)

- **Partitioning distributes data across nodes** — for scale-out and for
  locality. The goal: **skew and hot spots must be avoided**.
- **Partitioning strategies:**
  - **By key range** (e.g., A–M, N–Z): simple, but risks hot spots (e.g., keys
    that all start with "A" or a timestamp-based key).
  - **By hash of key** (consistent hashing or fixed hash buckets): distributes
    evenly, but loses range queries.
  - **Secondary index strategies:** local (document-based) indexes are
    partition-local (scatter-gather queries); global (term-based) indexes are
    centralized (single-query but write amplification).
- **Rebalancing:** add partitions, split, or migrate — must not take the system
  down. **Fixed number of partitions** (e.g., 10x node count) is the common
  practice; **consistent hashing** or **virtual nodes** for leaderless systems.
- **Request routing:** three approaches — client contacts any node (redirect),
  tiered routing (routing tier), or client-side partition-aware driver. The
  catalog service (ZooKeeper/etcd) tracks who owns what.
- **Cross-partition transactions** are the hard part — 2PC is the fallback; use
  it sparingly.
- **Hot partition mitigation:** salt the key (append random suffix to spread
  writes), or move the hot key's partition.

## Transactions

- **ACID is a contract, not a buzzword:** Atomicity (all-or-nothing), Consistency
  (invariants held — app's responsibility), Isolation (concurrent transactions
  don't interfere), Durability (committed data survives crashes).
- **Isolation levels — weakest to strongest:**

| Isolation level | Prevents | Allows (anomalies) | Implementation |
|---|---|---|---|
| Read committed | Dirty reads, dirty writes | Lost updates, read skew, phantoms | Row locks + MVCC |
| Snapshot isolation (repeatable read) | Read skew, non-repeatable reads | Write skew, phantoms | MVCC with snapshot |
| Serializable | All anomalies | None | Actual serial execution, 2PL, or SSI |

- **Write skew** is the signature anomaly of snapshot isolation: two
  transactions read the same data, then both write disjoint rows — each
  satisfying an invariant alone, but violating it together (e.g., on-call
  doctor swap, board game booking, username uniqueness). Fix: serializable
  isolation, or explicit locking (e.g., `SELECT ... FOR UPDATE`), or
  materialized conflicts.
- **Serializable implementations:**
  - **Actual serial execution** (single-threaded, all in one partition or a
    stored procedure): simple, fast, but limited by throughput and partitioning.
  - **Two-phase locking (2PL):** readers block writers; strong but slow and
    deadlock-prone.
  - **Serializable snapshot isolation (SSI):** optimistic concurrency — detect
    conflicts at commit time, abort if violated. Postgres serializable,
    FoundationDB.
- **Two-phase commit (2PC):** the coordinator asks all participants to prepare,
  then commits. **Blocking protocol** — a coordinator failure can hold locks
  indefinitely. It's the only universal way to get atomic commit across
  partitions, but it's a performance and availability liability.
- **Distributed transactions across heterogeneous systems** (XA) are worse —
  avoid. Prefer single-node transactions or carefully designed async
  compensations (sagas).

## Consistency, consensus, and the CAP tradeoff

- **Weak (eventual) consistency:** replicas converge eventually — no read
  guarantee. Strong consistency is expensive: only possible with a quorum.
- **CAP theorem:** in the presence of a network partition, you must choose
  **Consistency** (reject requests) or **Availability** (serve stale data).
  CAP is not "choose 2 of 3" — it only applies during a partition.
- **Linearizability:** the strongest single-object guarantee — every read
  sees the latest write; the system behaves like one copy. Costs: every read
  must reach the leader (or a quorum), making it unavailable during partitions.
  The cost is often unnecessary — only use linearizability when there's a real
  need (locking, leader election, unique constraint checks).
- **Ordering:** causality — if A caused B, A must be ordered before B. **Lamport
  timestamps** capture causal order (not total order). **Version vectors**
  track per-replica versions for concurrent-write detection.
- **Consensus:** getting nodes to agree on a value despite failures. Requires
  **majority quorum** (n/2 + 1) — tolerates f failures with 2f + 1 nodes.
  Paxos, Raft, Zab — all equivalent in intent.
- **Consensus algorithms properties:** unanimous agreement, integrity,
  validity, liveness (with majority up), and an elected **leader** (with
  term/epoch numbers to detect stale leaders).
- **Use cases for consensus:** leader election, unique constraints, linearizable
  storage (ZooKeeper/etcd are the standard building blocks).
- **Fencing tokens:** when a client uses a lock, the lock service issues a
  monotonically increasing token — a stale client's request with an old token
  is rejected. Prevents split-brain lock misuse.
- **Total order broadcast:** all nodes deliver messages in the same order —
  equivalent to consensus; it's the basis for replicated state machines.
- **Never trust the clock:** **NTP drift is real**; distributed systems need
  *logical clocks* (Lamport, version vectors) for correctness, not wall-clock.
  Clock skew can silently corrupt ordering (LWW data loss is a classic).

## Batch and stream processing

- **Batch processing (MapReduce, Spark, Hadoop):** process a bounded dataset,
  write output to a store. **Unix philosophy applies:** small, composable,
  deterministic programs — the same principles as `sort | uniq -c`.
- **Batch is predictable:** rerun it, it produces the same result (no
  nondeterminism). Failures are retried wholesale — no partial-state bugs.
- **Stream processing (Flink, Kafka Streams, Samza):** unbounded, continuous
  processing. **Event time vs processing time** — event time is when the event
  happened; processing time is when it's processed. Use **event time** with
  **watermarks** for out-of-order data.
- **Stream joins:** stream-stream joins (windowed), stream-table joins (enrich
  events), table-table joins (materialized views). State must be kept for
  joins — fault tolerance via checkpointing (Flink) or changelog topics (Kafka).
- **Fault tolerance:** exactly-once semantics requires transactional
  checkpointing (Flink) — expensive; at-least-once with idempotency is usually
  enough; exactly-once is a marketing term for "effectively once."
- **Windowing:** tumbling (fixed non-overlapping), hopping (sliding),
  session (gap-based). Choose by semantics of the question being asked.
- **The lambda architecture** (batch + speed layers) is complex — **Kappa
  architecture** (one stream, replayable log) is simpler when the log supports
  replay.
- **Untrusted input:** schema validation (Avro/Protobuf) at ingestion — bad
  data poisons pipelines downstream.

## Building blocks and estimation

- **Back-of-envelope estimation:** always do the math. Rules of thumb:
  - 1 day ≈ 86,400 seconds ≈ 10⁵ seconds.
  - 1M requests/day ≈ ~12 req/s average; peak ≈ 2–5x average.
  - Memory: 1 GB = 10⁹ bytes; a 64-bit pointer is 8 bytes.
  - Storage: 1 TB = 10¹² bytes; a 10 TB dataset with 3x replication = 30 TB.
- **Cache:** the classic tradeoff — read-through, write-through,
  write-behind, write-around. Invalidation is the hard part (cache aside +
  TTL is the pragmatic default). A cache is *not* a database — it can be lost.
- **CDNs** for static content; **read replicas** for read scale; **sharding**
  for write scale; **denormalization** for read speed at the cost of write
  complexity.
- **Message queues** decouple producers from consumers, absorb bursts, and
  enable async workflows; but they add latency and complexity (retries,
  idempotency, dead-letter queues).
- **Rate limiting:** token bucket / leaky bucket; enforce at the edge
  (API gateway), per user/IP/tenant.
- **Idempotency is not optional** in distributed systems: retries happen.
  Design every consumer with idempotency keys or deduplication.

## Design walkthrough checklist

When designing or reviewing a system, work through these questions:

- [ ] **Load profile:** what is the read/write ratio? QPS? data size? growth
      rate? Without numbers, "scalable" is meaningless.
- [ ] **Access patterns:** what are the 3–5 queries the system must serve
      fast? Design the data model around them, not around the data.
- [ ] **Consistency requirements:** what anomalies can the business tolerate?
      Is read-your-own-writes enough? Is linearizability actually required
      (only for locking, leader election, uniqueness)?
- [ ] **Storage engine:** write-heavy or read-heavy? LSM or B-tree? Column or
      row? Justify the choice against the access pattern.
- [ ] **Replication strategy:** leader-based, multi-leader, or leaderless?
      Sync or async? How is replication lag handled for read-your-own-writes?
- [ ] **Partitioning:** what's the partition key? Hot key risk? Rebalancing
      strategy? Secondary indexes local or global?
- [ ] **Failure modes:** what happens when each component fails? Every
      dependency needs a degradation path, not a hard dependency.
- [ ] **Transactions:** what invariants must hold under concurrency? Is
      snapshot isolation enough, or is write skew possible?
- [ ] **Ordering and clocks:** is any logic relying on wall-clock time? Are
      concurrent writes detected (version vectors) or silently lost (LWW)?
- [ ] **Schema evolution:** will fields be added/removed? Are formats
      backward *and* forward compatible? Is there a schema registry?
- [ ] **Batch vs stream:** does the workload need continuous processing?
      Event time or processing time? How are late events and watermarks handled?
- [ ] **Idempotency and retries:** every write path can be retried — are
      consumers idempotent? Is there a dead-letter queue?
- [ ] **Estimation:** back-of-envelope numbers for storage, QPS, bandwidth,
      memory — and a note of which numbers the design is sensitive to.
- [ ] **Operability:** how is this system observed (golden signals), deployed
      (canary), and recovered (backups, restore-tested)?
