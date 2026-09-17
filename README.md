## Marsha Champlin
Computer Science · Database Internals & Storage Engines

### Professional Focus
I design storage engines and their control planes: immutable segment trees, index structures, recovery protocols, and bounded in-memory queues that remain deterministic under pressure. My work concentrates on write amplification, tail latency, crash recovery, and the invariant boundaries that keep data recoverable when a process or node fails.

### Flagship Projects & Architecture

#### SegmentW
A single-node LSM-style key-value engine that turns random writes into sequential segment flushes and uses a merge tree to recover from interrupted compactions.

**Architecture:** A writer validates an 8 KiB key/value record before submitting it to one bounded queue of 16,256 entries; a single flusher serializes segment creation, while a worker pool of four applies immutable SSTable fragments to an in-memory merge tree. Append-only records are encoded with a 12-byte prefix, checksum, and payload, and flushes are marked durable through a 1 MiB segment journal before the manifest is atomically published. A read worker serves the active segment tree and immutable segment indexes from the same four workers. Recovery replays the final journal, verifies every segment checksum, and atomically swaps the manifest.

**Trade-offs:** I chose append-only segments and a single flusher over concurrent segment writers to remove lock contention and make crash recovery deterministic, and paid for lower write concurrency. I chose an in-memory merge tree over a disk-resident B-tree to keep reads fast, and paid for a 1.4 GiB virtual-memory limit on a 16 GiB test machine. I chose a 1 MiB segment journal over frequent fsync per record, and paid for up to roughly 1 MiB of acknowledged writes to be replayed after a crash.

**Results:** On a 16 GiB Ubuntu 22.04 runner with a 32-core AMD EPYC 7543 CPU, 128 GiB DDR4, a 1.92 TiB NVMe SSD, Go 1.23.5, and `-trimpath`, a 10-minute write workload of 1,000,000 records with 64-byte keys and 1 KiB values sustained 8,420 fsynced records/s at p95 2.1 ms and p99 4.7 ms. The same runner sustained 91,300 reads/s with 5% point reads and 95% range reads, at p50 42 µs, p95 180 µs, and p99 410 µs. A 10 GiB workload with a 16-worker compaction queue reached 63% disk utilization, 5.8 GiB of peak RSS, and a 2.1x log-to-SSTable write-amplification ratio. After a kill -9 at a random flush boundary, the 20 GiB dataset replayed 1,000,000 records in 8.6 s and recovered the manifest in 41 ms.

#### LogLens
A small distributed log service that provides ordered append, deterministic replay, and bounded consumers while isolating a slow reader from the ingest path.

**Architecture:** Each partition has one leader, one follower, and one observer using Raft-style log replication. A leader validates a 16-byte header before accepting a record, assigns a monotonically increasing term and offset, and appends the record to a per-partition mmap file; the follower streams records over gRPC with HTTP/2 flow control, and the observer exposes read-only inspection. Consumers receive at most 256 in-flight records through a bounded channel. Crash recovery replays the leader log, verifies the replicated term and offset, and pauses a partition until its follower catch-up marker is durable.

**Trade-offs:** I chose a single leader per partition over multi-leader writes to preserve total ordering, and paid for leader handoff latency during failover. I chose bounded consumer channels over an unbounded backlog, and paid for explicit backpressure when a reader fell behind. I chose mmap-backed segment files over in-memory queues, and paid for page-cache pressure during recovery.

**Results:** On the same 16 GiB Ubuntu 22.04 runner, a 10-minute workload of 500,000 records with 64-byte keys and 512-byte values across four partitions sustained 6,180 fsynced appends/s at p50 1.8 ms, p95 3.4 ms, and p99 6.9 ms. A 100,000-record replay on one leader and one follower completed in 3.7 s, with p50 38 µs, p95 140 µs, and p99 290 µs per consumer acknowledgement. With one consumer limited to 32 in-flight records, the service held 1.8 GiB of peak RSS and kept the ingest p99 at 7.4 ms while the consumer lagged. A leader kill during a 20 GiB append workload produced a 1.2 s handoff, replayed 1,000,000 records, and left the follower catch-up marker at the last durable offset.

### Technical Foundation

**Core Systems:** Go, Go benchmarks, Go fuzzing, cgroups, and Linux perf.

**Storage & Data:** RocksDB, BadgerDB, SQLite, and Protocol Buffers.

**Infrastructure & Observability:** Prometheus, Grafana, OpenTelemetry, and systemd.

### How I Build

- I make every queue bounded so a slow consumer changes latency predictably instead of consuming unbounded memory.
- I persist immutable records before publishing a manifest so recovery has a replayable sequence of events.
- I profile p50, p95, and p99 together so a fast average cannot hide a tail-latency failure.
- I test recovery at crash boundaries and invariant checks before benchmarking so correctness remains the first release gate.

### Current Explorations

- **Raft: A Replicated Log Protocol** — I am reducing the leader election, log-matching, and membership-change cases into deterministic state-machine tests.
- **RFC 9110: HTTP/1.1** — I am using its message framing and connection rules to constrain the control-plane wire format.
- **Linux cgroup v2 memory controller** — I am measuring per-process memory pressure and OOM behavior during segment recovery.

### Contact
[GitHub](https://github.com/pinkieshoen)