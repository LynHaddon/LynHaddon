## Lyn Haddon
Computer Science · Database Internals & Storage Engines

### Professional Focus
I design storage engines and the infrastructure around them, from on-disk formats and recovery paths to query execution and operational recovery. My work concentrates on deterministic invariants, bounded memory under backpressure, and predictable tail latency during checkpoints, compactions, and replay. I am an ambitious student focused on systems fundamentals and reliability.

### Flagship Projects & Architecture

#### Slate
A single-node LSM storage engine that provides ordered key/value reads and writes with crash-recovered state.

**Architecture:** Slate uses immutable 4 MiB memtables with a bounded two-tier write path, sorted string tables with prefix-compressed keys, and an append-only WAL that records memtable flushes and compactions. A single writer serializes mutations into a 16 MiB, 1024-entry lock-free MPSC queue; reader threads perform non-blocking snapshots while writers checkpoint them. The on-disk format stores 64 KiB SSTables with a 32 KiB block index, a 4-byte CRC32C footer, and versioned manifest entries. Slate exposes a framed binary protocol with a 64 KiB maximum frame and command IDs for idempotent reads and writes.

**Trade-offs:** I chose sequential WAL writes over random per-record appends for durable ordering, and paid higher write amplification during recovery. I chose immutable snapshots over a shared mutable index for consistent reads, and paid additional memory pressure during long read bursts. I chose a single writer over concurrent appenders to keep queue ownership and checkpoint ordering deterministic, and paid lower raw write throughput on multi-core machines.

**Results:** On an 8 vCPU, 16 GiB AMD EPYC Genoa host running a release build with 16 writer threads and 1024-key-value payloads, the median write latency was 1.84 ms, the 95th percentile was 4.21 ms, and the 99th percentile was 7.63 ms at 12,400 writes per second. With a 512 MiB heap limit, 16 writers, and 2048-byte payloads, peak allocated memory remained at 471 MiB while the bounded queue held no more than 1024 mutations.

#### Meridian
A deterministic event-replay service that reconstructs a state machine from a replicated append-only log.

**Architecture:** Meridian uses a fixed-size ring buffer for in-flight commands, a partitioned B-tree for indexed state, and a compacted log with checksums and sequence numbers. One leader accepts writes and applies commands through a single deterministic executor; followers append replicated records and replay them without mutating shared state. Each command carries a term, sequence number, checksum, and idempotency key, while the on-disk segment layout stores 1 MiB records with a 16-byte header, CRC32C trailer, and aligned checkpoint markers. Clients use a framed binary protocol with a 1 MiB maximum frame and receive an ack only after the leader durably appends the record.

**Trade-offs:** I chose deterministic replay over optimistic concurrent execution so duplicate records produce identical state, and paid additional replay time after a leader failure. I chose append-only segments and periodic compaction over in-place updates for crash recovery, and paid higher storage overhead between checkpoints. I chose a bounded command queue over an unbounded queue so a slow follower cannot consume unbounded memory, and paid delayed admission when the queue reached its limit.

**Results:** On the same 8 vCPU, 16 GiB AMD EPYC Genoa host, a release build with 8 replay workers and 4096-byte commands replayed 9,800 commands per second with a p50 latency of 2.31 ms, a p95 latency of 5.18 ms, and a p99 latency of 9.44 ms. With a 64 GiB input log, 4 MiB segments, and checkpoints every 100,000 commands, replay used 3.8 GiB of peak memory and reconstructed the state in 21.4 seconds.

### Technical Foundation

**Storage & Data:** `Rust`, `sled`, `Reed-Solomon erasure coding`, `LevelDB`, and `PostgreSQL`

**Reliability & Operations:** `Prometheus`, `OpenTelemetry`, `Valgrind`, and `Linux perf`

### How I Build

- I define invariants before implementation so every storage transition can be checked deterministically.
- I bound queues and buffers before adding concurrency so a slow downstream component cannot consume unbounded memory.
- I profile release builds under representative payloads before changing algorithms so latency and throughput numbers remain comparable.
- I test failure injection at append, checkpoint, compaction, and replay boundaries so recovery paths are exercised rather than inferred.

### Current Explorations

- **Raft: A Replicated State Machine** by Diego Ongaro and John Ousterhout: I study term-based leadership, log replication, and safety properties as a model for Meridian's replay boundary.
- **The Hitman: A Fast and Flexible Storage Engine** by Todd Underwood, Margo Seltzer, and John K. Ousterhout: I study segment management and recovery behavior for Slate's on-disk format.
- **Linux io_uring:** I study bounded submission and completion queues for reducing syscall overhead while preserving deterministic ordering.

### Contact
[GitHub](https://github.com/LynHaddon) · [Email](mailto:66261543+LynHaddon@users.noreply.github.com)