# Bigtable: A Distributed Storage System for Structured Data

**Date:** 2026-07-26
**Sources:**
- [Bigtable: A Distributed Storage System for Structured Data — Chang, Dean, Ghemawat et al. (Google, OSDI 2006)](https://storage.googleapis.com/gweb-research2023-media/pubtools/4443.pdf)

**Related entries:**
- [008-lsm-trees.md](008-lsm-trees.md) — Bigtable's tablet storage is an LSM tree (memtable + SSTables + compaction), the pattern that LevelDB/RocksDB later made widely available
- [005-facebook-tao.md](005-facebook-tao.md) — TAO's sharding (shard ID embedded in object ID) parallels Bigtable's tablet-based partitioning; both use single-master assignment

---

## What Bigtable Is

Bigtable is Google's internal distributed storage system for managing structured data at massive scale — petabytes across thousands of commodity servers. It powers Google Analytics, Google Earth, Google Finance, Personalized Search, the web crawl index, and over sixty other Google products. In production since April 2005.

Bigtable is not a full relational database. It provides a simpler data model giving clients dynamic control over data layout and format, letting them reason about locality properties directly. Think of it as a sparse, distributed, persistent multi-dimensional sorted map.

---

## The Data Model

Every value is indexed by three dimensions:

```
(row:string, column:string, time:int64) → string
```

### Rows

Arbitrary strings (typically 10-100 bytes). Every read or write to a single row key is atomic, regardless of how many columns are involved. Rows are maintained in lexicographic order, and the row range for a table is dynamically partitioned into **tablets** — the unit of distribution and load balancing (each ~100-200 MB).

Crucial design trick: clients choose row keys to exploit locality. In the Webtable, URLs are reversed — `com.google.maps/index.html` instead of `maps.google.com/index.html` — so pages from the same domain are stored adjacent, enabling efficient domain-level analyses.

### Column Families

Column keys are grouped into sets called column families, which form the unit of access control. A column key is `family:qualifier` — families are a small fixed set (rarely change), qualifiers are unbounded. In Webtable, `anchor:cnnsi.com` stores the anchor text from Sports Illustrated's link to CNN.

### Timestamps

Each cell can have multiple versions indexed by timestamp (64-bit integers). Garbage collection is configurable per column family — keep last N versions, or only versions from the last N days.

---

## Architecture

Three main components: a client library, one master server, and many tablet servers.

```
┌─────────────────────────────────────────────────────────────┐
│                        Chubby                                │
│        (distributed lock service, Paxos-based)               │
│        - master election                                     │
│        - tablet server discovery                             │
│        - schema storage                                      │
└──────────────────────────┬──────────────────────────────────┘
                           │
                    ┌──────┴──────┐
                    │   Master    │
                    │ (assigns    │
                    │  tablets to │
                    │  servers)   │
                    └──────┬──────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐
  │Tablet Server│  │Tablet Server│  │Tablet Server│
  │  (10-1000   │  │             │  │             │
  │   tablets)  │  │             │  │             │
  └──────┬──────┘  └─────────────┘  └─────────────┘
         │
    ┌────┴────┐
    │  GFS    │   (Google File System — stores SSTables and logs)
    └─────────┘
```

The **master** assigns tablets to tablet servers, detects server additions/expirations, balances load, handles garbage collection. Client data does NOT flow through the master — clients talk directly to tablet servers. Master is lightly loaded.

**Chubby** (distributed lock service, Paxos-based) provides: master election, tablet server discovery (each server acquires exclusive lock in a Chubby directory), schema storage, access control. If Chubby unavailable → Bigtable unavailable (measured at 0.0047% of server-hours).

### Tablet Location

Three-level hierarchy (analogous to B+ tree):

```
Chubby file → Root tablet (never split) → METADATA tablets → User tablets
```

Root tablet contains locations of all METADATA tablets. Each METADATA tablet stores user tablet locations. With 128MB METADATA tablets, addresses 2^34 tablets. Client library caches locations and recursively moves up on misses (3 network round-trips if cache empty, 6 if stale).

### Tablet Assignment

Each tablet is assigned to one tablet server at a time. The master tracks live servers and assignments:

- Tablet servers register by acquiring exclusive lock on a file in Chubby's servers directory
- Master monitors that directory to discover servers
- If a server loses its lock (network partition, death), master tries to acquire it — if successful, the server is dead, master reassigns its tablets
- Master kills itself if its own Chubby session expires (doesn't change assignments, just stops making new ones)

On master startup: grab master lock → scan servers directory → ask each server what tablets it has → scan METADATA table → assign any unassigned tablets.

---

## Tablet Serving: The LSM Tree Inside

Each tablet's persistent state lives in GFS:

```
┌─────────────────────────────────┐
│          In Memory              │
│  ┌─────────────┐  ┌─────────┐  │
│  │  memtable   │  │ Read Op │  │
│  │ (sorted)    │  │         │  │
│  └─────────────┘  └─────────┘  │
├─────────────────────────────────┤
│          GFS (Disk)             │
│  ┌────────────┐                 │
│  │ tablet log │ (commit log)    │
│  └────────────┘                 │
│  ┌───┐ ┌───┐ ┌───┐            │
│  │SST│ │SST│ │SST│ (SSTables)  │
│  └───┘ └───┘ └───┘            │
└─────────────────────────────────┘
```

**Write path:** Check authorization → append to commit log (group commit for throughput) → insert into memtable.

**Read path:** Merged view of memtable + SSTables. Both are lexicographically sorted, so merging is efficient.

**Compaction types:**
- **Minor compaction:** Freeze memtable → flush to new SSTable in GFS. Shrinks memory, shortens recovery.
- **Merging compaction:** Read a few SSTables + memtable → write one new SSTable. Bounds total file count.
- **Major compaction:** Merge ALL SSTables into exactly one. Produces SSTable with no deletion markers or obsolete data — important for reclaiming space and ensuring deleted data disappears (privacy).

---

## Key Refinements

**Locality groups:** Clients group column families into locality groups. Separate SSTable per locality group per tablet. Columns not typically accessed together are segregated for more efficient reads. Locality groups can be declared in-memory for small, frequently-accessed data.

**Compression:** Per-SSTable-block, two-pass: Bentley-McIlroy (long common strings) + fast 16KB window compression. Achieves 10:1 on web pages because reversed-URL keys place same-host pages adjacent, exposing shared boilerplate.

**Bloom filters:** Reduce disk I/Os for lookups on non-existent rows/columns. Most lookups are for non-existent keys, so Bloom filters eliminate most unnecessary seeks.

**Single commit log per tablet server:** Co-mingles mutations for all tablets on one server. Better write throughput via group commit. Recovery: sort log entries by `(table, row, sequence_number)`, partition into 64MB segments, sort in parallel across servers.

**Dual log-writing threads:** Two threads each writing its own log file, only one active. If active thread stalls on GFS latency, switch to the other. Prevents latency spikes from stalling mutations.

**Exploiting immutability:** SSTables are immutable → no synchronization for concurrent reads. Only the memtable is mutable (copy-on-write for rows allows parallel reads and writes). Tablet splits are cheap — children share parent's SSTables.

**Speeding tablet recovery:** Before unloading a tablet, source server does a minor compaction (reduces uncompacted log state). Then a second fast minor compaction for any mutations that arrived during the first. Receiving server needs no log replay.

---

## Performance (2006)

With 1000-byte values, commodity hardware (dual-core Opteron, 1Gbps):

| Operation | 1 server | 50 servers | 500 servers |
|-----------|----------|-----------|-------------|
| Random reads | 1,212/s | 479/s per | 241/s per |
| Random reads (mem) | 10,811/s | 8,000/s per | 6,250/s per |
| Random writes | 8,850/s | 3,745/s per | 3,425/s per |
| Sequential reads | 4,425/s | 2,463/s per | 2,625/s per |
| Sequential writes | 8,547/s | 3,623/s per | 2,451/s per |
| Scans | 15,385/s | 10,526/s per | 9,524/s per |

Aggregate throughput scales to over 4M ops/sec for scans at 500 servers. Random reads scale worst due to 64KB block transfers for 1KB values saturating network.

---

## Real Applications at Google (2006)

- **Google Analytics:** ~200TB raw click table (sessions per site, chronological), compresses to 14% of original. ~20TB summary table (MapReduce-generated), compresses to 29%.
- **Google Earth:** ~70TB preprocessing table (raw imagery), plus ~500GB serving table (served with low latency across hundreds of tablet servers with in-memory column families).
- **Personalized Search:** Per-user data (queries, clicks across Google properties). Each user is a row, column families per action type, timestamps for temporal ordering. Replicated across multiple clusters.

---

## Lessons from Google

- Large distributed systems are vulnerable to many failure types beyond partitions: memory corruption, clock skew, hung machines, asymmetric partitions, bugs in dependencies.
- Delay features until their use is clear. Initially planned general transactions — found most apps only need single-row atomicity.
- System-level monitoring is essential. Extended RPC system to trace every request end-to-end.
- **Value simple designs.** The tablet-server membership protocol was redesigned several times. Complex protocols that worked were too hard to debug. Final version relies solely on well-tested Chubby features.
