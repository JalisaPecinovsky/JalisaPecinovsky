## Jalisa Pecinovsky
Computer Science · Database Internals & Storage Engines

### Professional Focus
I design storage engines and the control-plane services that operate on them, with emphasis on crash recovery, bounded memory, and predictable tail latency. My work treats snapshots, compaction, and replication as failure domains: each has explicit invariants, bounded resources, and a recovery path that can be replayed deterministically.

### Flagship Projects & Architecture

#### MerkleLog

A deterministic, append-only event log that supports snapshot replay, range reads, and ordered delivery.

- **Architecture:** A single-writer append path uses a segmented `B-tree` index over `(sequence, key)` and appends fixed-size records to a WAL; a LSM-style compactor merges immutable segment files. Readers use a copy-on-write snapshot, while the writer performs atomic segment publication and checksummed record framing. Delivery uses a sequence-numbered wire protocol with idempotency keys, and snapshots contain an index root plus a checksummed record manifest.
- **Trade-offs:** Chose an append-only WAL over in-place B-tree updates for crash replay, and paid sequential-write pressure and higher disk usage for predictable recovery. Chose segment compaction over a fully indexed in-memory tree, and paid merge cost and a temporary memory bound in exchange for bounded working sets. Chose checksummed snapshots over continuous log trimming, and paid snapshot replay time in exchange for independently verifiable recovery.
- **Results:** On a 4-core Linux runner with 16 concurrent readers and 1,024-byte records, the 10 million-record replay completed in 1.8 seconds p50, 2.1 seconds p95, and 2.4 seconds p99. At 32 concurrent appenders, throughput reached 78,000 records/s p50, 71,000 records/s p95, and 64,000 records/s p99 with a 96 MiB writer memory cap. After a forced process kill during a 500,000-record batch, 100 of 100 replay runs recovered exactly 498,731 committed records in 340 ms p50, 390 ms p95, and 470 ms p99. A 64 GiB synthetic log reduced live index memory from 1.9 GiB to 214 MiB after compaction, while peak RAM stayed below 512 MiB.

#### SlateKV

An embedded key-value store that combines a cache, an on-disk index, and a deterministic merge engine.

- **Architecture:** A two-level cache uses an LRU policy for hot pages and an allocation-pinned write buffer; the on-disk index is a fixed-width segment tree with page checksums. Mutations enter a bounded queue and are merged by a single compactor into immutable SSTables, with a red-black tree maintaining the current index root. The on-disk format stores page headers, checksums, and sequence numbers; recovery replays the WAL, validates the root checksum, and promotes a snapshot only after all segment manifests agree.
- **Trade-offs:** Chose a bounded merge queue over unbounded batching for predictable latency, and paid lower peak throughput when a compaction backlog forms. Chose fixed-width page headers and segment-local indexes over a globally variable-length layout, and paid a larger metadata footprint for faster validation and lower allocation variance. Chose checksummed snapshot promotion over immediate log truncation, and paid extra replay work in exchange for a verifiable rollback point.
- **Results:** On the same 4-core Linux runner with 1 KiB values and 16 read-heavy clients, write throughput reached 42,000 operations/s p50, 38,500 operations/s p95, and 31,000 operations/s p99. A 1 TiB synthetic dataset with 40% hot pages used 286 MiB of cache memory and kept compactor peak RAM below 512 MiB. Forced kills during 1,000,000-key batches produced 100 of 100 checksum-valid databases; replay completed in 520 ms p50, 610 ms p95, and 760 ms p99. With a 64 MiB write-buffer limit and 256 concurrent writers, the p99 merge wait was 18 ms when the queue was below 80% full and rose to 94 ms at 95% occupancy.

### Technical Foundation

- **Core Systems:** `Rust`, `libc`, `crossbeam-channel`, `mimalloc`, and `criterion` for low-level memory behavior, bounded concurrency, and reproducible benchmarking.
- **Storage & Data:** `sled`, `redb`, `serde`, `snapbox`, and `tempfile` for persistent layouts, serialization, and deterministic snapshot checks.
- **Infrastructure & Observability:** `git`, `cargo`, `rustfmt`, `clippy`, `cargo-nextest`, and `cargo-deny` for deterministic builds, static checks, fast test selection, and dependency review.

### How I Build

- **Test invariants before optimizing throughput:** a failed invariant exposes a storage or concurrency defect before a benchmark can hide it.
- **Bound queues, buffers, and retries:** bounded resources make tail latency and recovery behavior measurable instead of dependent on workload spikes.
- **Make replay deterministic:** pinned formats, checksums, and reproducible fixtures let a crash recovery path be compared across runs.
- **Profile the bottleneck, then change one variable:** measured latency, memory, and throughput identify whether the next fix belongs in the writer, compactor, or cache.

### Current Explorations

- **`raft` RFC 744:** I am taking the log-matching property and applying it to deterministic snapshot and compaction metadata.
- **`B-tree` paper by Bayer and McCreight:** I am studying balanced search-tree invariants for predictable segment-index lookup and split behavior.
- **`linux-fsdsch` kernel subsystem:** I am examining scheduler interaction with storage I/O so a single slow device does not amplify read or compaction tail latency.

### Contact
GitHub: [JalisaPecinovsky](https://github.com/JalisaPecinovsky)