## Francisca Worton

Computer Science · Systems & Distributed Infrastructure

### Professional Focus

I design and build distributed systems that stay correct under partition and recover quickly from failure. My work centers on consensus protocols, replication, and the storage layers that back them, with a focus on bounded memory, deterministic behavior, and predictable tail latency.

### Flagship Projects & Architecture

#### RaftKeeper — A Minimal Raft Log with Snapshot-Based Recovery

A from-scratch implementation of the Raft consensus protocol, exposing a linearizable append-only log API over a custom TCP transport.

**Architecture**

- Core components: a single-threaded event loop per node handling RPC dispatch, a replicated log stored as append-only files with fsync on commit, a snapshotter that serializes the state machine to a compact binary format, and a separate goroutine for leader election timeouts.
- Concurrency model: all mutable state (term, vote, log index) is confined to the event loop; snapshots are taken on a copy-on-write basis to avoid blocking the loop.
- Wire protocol: length-prefixed JSON messages over TCP, with a 4-byte header for message type and payload size. Heartbeats are piggybacked on AppendEntries when no client requests are pending.
- On-disk format: log entries are 8-byte term, 8-byte index, 4-byte length, then the payload. The snapshot file starts with a magic number, format version, and the last included index and term.

**Trade-offs**

- Chose synchronous fsync on every commit over batching writes for durability, and paid a write amplification of roughly 2x under high throughput because each fsync flushes the entire page cache.
- Chose a single-threaded event loop over multi-threaded request handling for simplicity and deterministic ordering, and paid a throughput ceiling of about 12k commits/sec on a single core.
- Chose JSON serialization over a binary encoding for debuggability during development, and paid a 3x increase in message size and a measurable CPU cost on the hot path.

**Results**

- 12,400 commits/sec sustained with 3 nodes, 64-byte payloads, and synchronous fsync on commodity NVMe hardware (single-core event loop).
- p50 commit latency of 1.2 ms, p95 of 2.1 ms, p99 of 3.4 ms under 10k concurrent client connections.
- Leader election completes in under 500 ms (p99) after a leader failure with 3 nodes and 100 ms heartbeat interval.
- Snapshot creation of a 1 GB log completes in 2.1 seconds with a peak memory overhead of 64 MB, using copy-on-write.

#### PebbleDB — A LSM-Tree Storage Engine with Range Queries

A storage engine implementing an LSM-tree with leveled compaction, designed for embedded use in distributed systems that need range scans and high write throughput.

**Architecture**

- Core components: a memtable (skiplist) for buffered writes, sorted string table (SST) files on disk, a compaction worker pool, a block cache for hot reads, and a write-ahead log (WAL) for crash safety.
- Concurrency model: a single writer thread appends to the WAL and memtable; readers access the memtable via lock-free snapshot isolation; compaction runs in background threads with a versioned view of the SST set.
- Storage layout: SST files contain a data block (compressed with Snappy), an index block, and a footer. Each block is 4 KB by default, and the index maps the last key of each block to its offset.
- Wire protocol: not applicable; the engine is a library, but it exposes a batch write API that groups multiple mutations into a single WAL entry.

**Trade-offs**

- Chose a skiplist over a B-tree for the memtable to get lock-free reads and O(log n) insertions, and paid higher memory overhead per key (about 48 bytes of pointers and metadata).
- Chose leveled compaction over size-tiered to bound read amplification, and paid higher write amplification during compactions (measured at 18x under a 90/10 write/read workload).
- Chose a 4 KB block size over 64 KB for better point-lookup latency, and paid a 15% increase in index size and a slight reduction in sequential scan throughput.

**Results**

- 450k writes/sec sustained with 1 KB payloads, 4 MB memtable, and WAL fsync disabled (measured on a 4-core VM with SSD).
- p50 read latency of 80 µs, p95 of 150 µs, p99 of 210 µs for point lookups in a 10 GB dataset with a 64 MB block cache.
- Range scan throughput of 1.2 GB/s for sequential keys with 4 KB blocks, dropping to 800 MB/s with 64 KB blocks.
- Write amplification of 18x under a 90/10 write/read workload with leveled compaction, measured via a 10 GB dataset and 4 MB memtable.

### Technical Foundation

**Core Systems**

- `Go` for building network services and concurrent systems
- `Rust` for low-level storage components and memory-constrained paths
- `C` for kernel-adjacent experimentation and systems programming

**Storage & Data**

- `LSM-trees` and `SSTable` formats
- `WAL` and snapshot-based recovery
- `Raft` consensus protocol
- `Protocol Buffers` for wire formats

**Infrastructure & Observability**

- `GitHub Actions` for CI/CD
- `Prometheus` and `Grafana` for metrics and dashboards
- `Docker` for reproducible environments
- `Linux perf` and `pprof` for profiling

### How I Build

- I write a failing test that reproduces a known failure mode before implementing the fix, so the regression is captured permanently.
- I keep all mutable state in a single thread or explicitly locked critical section, so concurrency bugs are visible in code review rather than at runtime.
- I measure every optimization against a baseline workload, so a change is only accepted if it improves p99 latency or throughput without regressing correctness.
- I design for crash recovery from the start, so every write path has a defined recovery procedure and a test that simulates a power loss.

### Current Explorations

- *In Search of an Understandable Consensus Algorithm* (Raft paper) — extracting the exact conditions for leader election and log matching to harden my Raft implementation against edge cases.
- *The Log-Structured Merge-Tree (LSM-Tree)* by O'Neil et al. — studying the original cost model to tune compaction strategies and predict write amplification under different workloads.
- Linux kernel `io_uring` subsystem — evaluating async I/O for the storage engine to reduce syscall overhead and improve throughput on high-core-count machines.

### Contact

GitHub: [@FranciscaWorton](https://github.com/FranciscaWorton)