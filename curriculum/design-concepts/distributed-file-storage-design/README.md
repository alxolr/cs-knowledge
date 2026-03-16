# Distributed File Storage Design

**Track:** Design Concepts
**Difficulty Tier:** Advanced
**Prerequisites:** [Data Partitioning](../data-partitioning/README.md), [Replication Strategies](../replication-strategies/README.md)

## Concept Overview

Distributed file storage systems allow organizations to store and retrieve files across a cluster of commodity machines, providing capacity, throughput, and fault tolerance far beyond what a single server can offer. Systems like the Google File System (GFS), the Hadoop Distributed File System (HDFS), and cloud object stores such as Amazon S3 are built on these principles. At their core, they split files into fixed-size chunks, spread those chunks across many storage nodes, and replicate each chunk so that data survives individual disk or machine failures.

Designing such a system requires balancing several competing concerns. Metadata management — knowing which chunks belong to which file and where each chunk is stored — must be fast and highly available, yet the metadata store itself can become a bottleneck or single point of failure. Chunk placement must distribute load evenly while respecting rack-awareness and failure-domain constraints so that a single rack power failure does not destroy all copies of a chunk. Consistency guarantees range from strong (every read sees the latest write) to eventual (replicas converge over time), and the choice affects both performance and application correctness.

Beyond the basics of storage and retrieval, production systems must handle concurrent writers, large-file appends, garbage collection of orphaned chunks, rebalancing when nodes join or leave the cluster, and efficient support for both small random reads and large sequential scans. This module explores these challenges through scenario-based design problems that exercise different aspects of building a distributed file storage system from the ground up.

---

## Problem 1 — High-Level Architecture of a Chunk-Based File System

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Design the high-level architecture for a distributed file storage system that splits files into fixed-size chunks and stores them across a cluster of storage nodes. The design should cover the core components — a metadata service, storage nodes, and a client library — and describe the end-to-end flow for writing a file and reading it back.

### Scenario

**Context:** A mid-size analytics company processes 50 TB of log data daily. Their single NFS server is running out of capacity and cannot keep up with the read throughput required by their batch processing jobs. The team wants to build an internal distributed file system using 100 commodity servers, each with 10 TB of disk. Files range from 100 MB to 10 GB. The system should tolerate the failure of any two servers without data loss. The team needs a clear architectural blueprint before implementation begins.

**Requirements:** Identify the major components of the system and their responsibilities. Define the chunk size and justify the choice. Describe the write path (client uploads a file) and the read path (client retrieves a file) step by step. Explain how the metadata service tracks file-to-chunk mappings and chunk-to-node locations. State what happens when a storage node fails.

**Expected Approach:** Introduce a metadata service (master) that manages the namespace and chunk mappings, storage nodes (chunk servers) that store the actual data, and a client library that coordinates reads and writes through the metadata service. Choose a chunk size of 64 MB to balance metadata overhead against parallelism. Replicate each chunk to three nodes for fault tolerance.

<details>
<summary>Hints</summary>

1. A centralized metadata service (often called the master or name node) stores the file namespace (directory tree), the mapping from each file to its ordered list of chunk IDs, and the mapping from each chunk ID to the set of storage nodes holding replicas. This metadata is small enough to fit in memory for millions of files.
2. A chunk size of 64 MB keeps the total number of chunks manageable (a 10 GB file produces ~160 chunks) while still allowing parallel reads across multiple nodes. Smaller chunks increase metadata overhead; larger chunks reduce parallelism.
3. On the write path, the client asks the metadata service for chunk placement (which three nodes should store each chunk), then streams data directly to the storage nodes — the metadata service is not on the data path. This avoids making the metadata service a throughput bottleneck.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Design a three-component architecture — metadata service, chunk servers, and client library — with 64 MB chunks replicated three ways.

1. **Components:**
   - **Metadata Service (Master):** Maintains the file namespace (hierarchical directory tree), the file-to-chunk mapping (ordered list of chunk handles per file), and the chunk-to-node mapping (which chunk servers hold each chunk's replicas). Stores all metadata in memory for fast lookups, with a write-ahead log and periodic checkpoints to disk for durability.
   - **Chunk Servers:** Commodity machines that store chunks as regular files on local disks. Each chunk is identified by a globally unique chunk handle (64-bit ID). Chunk servers send periodic heartbeats to the metadata service, reporting which chunks they hold and their disk utilization.
   - **Client Library:** A library linked into applications that translates file operations (open, read, write) into metadata service RPCs (to locate chunks) and direct data transfers with chunk servers (to read/write data).

2. **Chunk size justification (64 MB):**
   - A 10 GB file produces 160 chunks → 160 metadata entries. With millions of files, total metadata fits in tens of GB of RAM.
   - 64 MB is large enough to amortize the overhead of establishing a network connection per chunk, yet small enough to allow parallel reads from multiple chunk servers.

3. **Write path (uploading a 1 GB file):**
   ```
   1. Client calls metadata service: CREATE("/data/logs/2024-01-15.log", file_size=1GB)
   2. Metadata service allocates 16 chunk handles (1 GB / 64 MB) and for each chunk selects 3 chunk servers (primary + 2 secondaries), returning the list to the client.
   3. For each chunk, the client streams 64 MB of data to the primary chunk server.
   4. The primary chunk server forwards the data to the two secondary servers (pipeline replication: primary → secondary1 → secondary2).
   5. Once all three replicas acknowledge, the primary reports success to the client.
   6. After all 16 chunks are written, the client confirms completion to the metadata service, which marks the file as complete.
   ```

4. **Read path (reading the same file):**
   ```
   1. Client calls metadata service: OPEN("/data/logs/2024-01-15.log")
   2. Metadata service returns the ordered list of chunk handles and, for each chunk, the set of chunk servers holding replicas.
   3. For each chunk, the client picks the closest (or least loaded) chunk server and reads 64 MB directly from it.
   4. The client assembles the chunks in order and returns the file content to the application.
   ```

5. **Failure handling:**
   - When a chunk server misses heartbeats for 30 seconds, the metadata service marks it as dead and identifies all chunks that now have fewer than 3 replicas.
   - The metadata service instructs healthy chunk servers holding the remaining replicas to copy the under-replicated chunks to new servers, restoring the replication factor to 3.
   - Because the system tolerates 2 simultaneous server failures (3 replicas), losing one server does not cause data loss as long as re-replication completes before a second failure.

</details>

---

## Problem 2 — Metadata Service Scalability and High Availability

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a highly available and scalable metadata service for a distributed file storage system that manages the namespace and chunk location data for over 500 million files. The metadata service must survive the failure of any single node without downtime, handle thousands of metadata operations per second, and avoid becoming a bottleneck as the cluster grows. Address how metadata is persisted, how failover works, and how the system handles the memory limits of a single machine when the metadata set grows very large.

### Scenario

**Context:** The distributed file system from Problem 1 has grown to 500 million files comprising 200 billion chunks. The single-node metadata service holds all metadata in memory, which now requires 120 GB of RAM. The team is hitting two problems: (1) if the metadata service crashes, the entire cluster is unavailable until it restarts and replays its write-ahead log (which takes 15 minutes), and (2) the single node is approaching its memory ceiling and cannot accommodate projected growth to 2 billion files within two years. The team needs a metadata service redesign that eliminates the single point of failure and scales beyond a single machine's memory.

**Requirements:** Design a failover mechanism that provides sub-second recovery when the active metadata node fails. Decide whether to use active-passive replication, multi-raft consensus, or another approach, and justify the choice. Address how the metadata set can be partitioned across multiple machines when it exceeds single-node memory. Describe how clients discover the current active metadata node after a failover. Analyze the consistency guarantees of the metadata service — can a client ever see stale chunk locations?

**Expected Approach:** Use a Raft-based consensus group of 3–5 metadata nodes with one leader handling all writes and followers replicating the log. On leader failure, a follower is elected within seconds. For scaling beyond single-node memory, partition the namespace by directory subtree or hash-based sharding across multiple Raft groups. Clients discover the leader via a lightweight service registry or by querying any node which redirects to the current leader.

<details>
<summary>Hints</summary>

1. Raft consensus with 3 or 5 nodes provides automatic leader election. The leader processes all metadata mutations and replicates them to followers. If the leader fails, a new leader is elected in 1–5 seconds, and the cluster resumes serving requests. This eliminates the 15-minute restart window.
2. Chunk location data (which chunk servers hold which chunks) does not need to be persisted in the Raft log — it can be reconstructed from chunk server heartbeats after a failover. Only the namespace (file/directory tree) and file-to-chunk mappings need to be in the replicated log, which significantly reduces the log size.
3. For namespace partitioning, split the directory tree at the top level — e.g., `/data/team-a/` is managed by Raft group 1, `/data/team-b/` by Raft group 2. This keeps related files together and avoids cross-partition operations for most workloads.
4. Clients can discover the leader by contacting any metadata node. If the contacted node is a follower, it returns a redirect to the current leader's address. Alternatively, use a service registry (e.g., ZooKeeper or etcd) that always points to the current leader.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Deploy a Raft-based replicated metadata service with namespace partitioning across multiple Raft groups for horizontal scaling.

1. **Raft-based replication (eliminating single point of failure):**
   - Deploy 5 metadata nodes forming a Raft consensus group. One node is the leader; the other 4 are followers.
   - All metadata write operations (create file, delete file, allocate chunk) go through the leader, which appends them to the Raft log. The operation is committed once a majority (3 of 5) acknowledge.
   - Read operations can be served by the leader (strong consistency) or by followers (slightly stale but lower latency). For chunk location lookups, eventual consistency is acceptable because the client will retry on a different chunk server if the location is stale.
   - **Failover:** If the leader crashes, the remaining 4 nodes detect the missing heartbeat within 1–2 seconds and elect a new leader. Clients experience a brief pause (1–5 seconds) during election, then resume operations against the new leader.

2. **Separating persistent metadata from ephemeral metadata:**
   - **Persistent metadata (replicated via Raft):** File namespace (directory tree), file-to-chunk-handle mappings, chunk replication factor, access control lists.
   - **Ephemeral metadata (reconstructed from heartbeats):** Chunk-handle-to-chunk-server mappings. Each chunk server reports its chunk inventory via heartbeats every 10 seconds. After a leader failover, the new leader rebuilds the chunk location map within 10–30 seconds as heartbeats arrive.
   - This separation reduces the Raft log size by ~60%, since chunk locations are the most frequently changing metadata.

3. **Namespace partitioning for horizontal scaling:**
   - When the metadata set exceeds single-node memory (~200 GB), partition the namespace across multiple Raft groups.
   - **Partitioning strategy:** Split by top-level directory. Each top-level directory (e.g., `/data/`, `/models/`, `/logs/`) is assigned to a Raft group. A lightweight router service maps path prefixes to Raft groups.
   - Each Raft group manages an independent subset of the namespace and can be hosted on different sets of machines.
   - **Cross-partition operations** (e.g., renaming a file from `/data/` to `/logs/`) require a two-phase commit across Raft groups — these are rare and can be handled as an expensive special case.

4. **Client leader discovery:**
   - The client library maintains a cached address of the current leader for each Raft group.
   - If a request fails or returns a "not leader" response, the client contacts any node in the group, which responds with the current leader's address.
   - Alternatively, a service registry (etcd or ZooKeeper) stores the current leader address, updated on every election. Clients watch this key for changes.

5. **Consistency guarantees:**
   - **Namespace operations (create, delete, rename):** Strongly consistent — all mutations go through the Raft leader and are committed by majority.
   - **Chunk location lookups:** Eventually consistent — the location map is rebuilt from heartbeats and may be briefly stale after a failover or when a chunk server has just moved a chunk. Clients handle stale locations by retrying on a different replica.
   - **Read-after-write for file creation:** Guaranteed if the client reads from the leader. If reading from a follower, there may be a replication lag of a few milliseconds.

6. **Memory optimization:**
   - Compact in-memory representation: store file names as interned strings, use 64-bit chunk handles, and represent the directory tree as a trie.
   - With 500 million files averaging 400 chunks each, the namespace and chunk mappings require approximately 80–120 GB of RAM per Raft group. Partitioning into 4 groups reduces this to 20–30 GB per group, well within commodity server capacity.

</details>

---

## Problem 3 — Chunk Placement, Rack Awareness, and Rebalancing

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design the chunk placement policy and rebalancing mechanism for a distributed file storage system deployed across multiple data center racks. The placement policy must distribute chunks to maximize fault tolerance (surviving rack-level failures), balance storage utilization across nodes, and minimize network traffic during reads. The rebalancing mechanism must handle nodes joining or leaving the cluster, redistribute chunks to maintain even utilization, and do so without disrupting ongoing read/write operations.

### Scenario

**Context:** The distributed file system is deployed across 500 chunk servers organized into 50 racks (10 servers per rack). Each rack has its own top-of-rack switch, and all racks connect to a core switch. If a rack loses power or its top-of-rack switch fails, all 10 servers in that rack become unavailable simultaneously. The current placement policy randomly selects 3 chunk servers for each chunk's replicas, but this occasionally places all 3 replicas in the same rack — a rack failure then causes data loss. Additionally, as servers are added and removed over time, storage utilization has become uneven: some servers are 90% full while others are only 40% full. The team needs a smarter placement policy and an automated rebalancing system.

**Requirements:** Design a rack-aware placement policy that ensures no two replicas of the same chunk are in the same rack (or at most one replica per rack when replication factor is 3). Define how the metadata service learns the rack topology. Design a rebalancing algorithm that migrates chunks from over-utilized servers to under-utilized servers while respecting rack-awareness constraints. Specify how rebalancing is throttled to avoid saturating network bandwidth. Explain how the system handles a new rack of 10 servers joining the cluster — how quickly are chunks migrated to the new servers, and how is the migration prioritized?

**Expected Approach:** Implement a placement policy that selects replicas from different racks, preferring racks with lower utilization. Chunk servers report their rack ID during registration. For rebalancing, run a background process that identifies imbalanced servers and schedules chunk migrations, throttled to a configurable bandwidth limit (e.g., 100 MB/s per server). When new servers join, prioritize migrating chunks from the most over-utilized servers first.

<details>
<summary>Hints</summary>

1. A common rack-aware policy for 3 replicas: place the first replica on a server in rack A, the second replica on a server in a different rack B, and the third replica on another server in rack B (different from the second). This survives any single rack failure while keeping two replicas close for efficient reads.
2. The metadata service maintains a rack topology map: `rack_id → [list of chunk servers]`. Chunk servers report their rack ID (configured at deployment or inferred from IP address) during heartbeat registration.
3. Rebalancing should be gradual. A "rebalancing score" for each server can be computed as `(actual_utilization - target_utilization) / target_utilization`. Servers with high positive scores are sources; servers with high negative scores are destinations. Migrate chunks from sources to destinations until scores converge.
4. Throttle migrations by limiting the number of concurrent chunk transfers per server (e.g., 2 outbound + 2 inbound) and by capping the bandwidth allocated to rebalancing (e.g., 100 MB/s). This ensures rebalancing does not starve client read/write traffic.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement a rack-aware placement policy with a two-rack strategy, and a background rebalancer that uses utilization scores and bandwidth throttling.

1. **Rack topology management:**
   - Each chunk server is configured with a `rack_id` (e.g., via environment variable or derived from its IP subnet).
   - During heartbeat registration, the chunk server reports its `rack_id` to the metadata service.
   - The metadata service maintains a map: `rack_id → [chunk_server_ids]` and `chunk_server_id → rack_id`.

2. **Rack-aware placement policy (for replication factor 3):**
   ```
   function selectReplicaLocations(chunk_handle):
       # Get all racks sorted by average utilization (ascending)
       racks = getRacksByUtilization()
       
       # Select two racks with lowest utilization
       rack_A = racks[0]
       rack_B = racks[1]
       
       # Select one server from rack_A (lowest utilization in rack)
       server_1 = selectLowestUtilizationServer(rack_A)
       
       # Select two servers from rack_B (lowest utilization, different servers)
       server_2 = selectLowestUtilizationServer(rack_B)
       server_3 = selectSecondLowestUtilizationServer(rack_B)
       
       return [server_1, server_2, server_3]
   ```
   - **Fault tolerance:** If rack_A fails, 2 replicas remain in rack_B. If rack_B fails, 1 replica remains in rack_A — the system can still serve reads and will re-replicate to restore the replication factor.
   - **Read locality:** Two replicas in the same rack allow clients in that rack to read locally without crossing the core switch.

3. **Rebalancing algorithm:**
   ```
   function rebalance():
       target_utilization = total_used_space / total_capacity  # e.g., 60%
       
       for each chunk_server:
           score = (server.utilization - target_utilization) / target_utilization
           # score > 0 means over-utilized (source)
           # score < 0 means under-utilized (destination)
       
       sources = servers where score > 0.1, sorted by score descending
       destinations = servers where score < -0.1, sorted by score ascending
       
       for each source in sources:
           for each chunk on source:
               if migration_budget_exhausted(): return
               
               # Find a destination in a different rack that doesn't already have this chunk
               dest = selectDestination(chunk, destinations)
               if dest:
                   scheduleMigration(chunk, source, dest)
                   update destination's projected utilization
   ```

4. **Migration execution:**
   - The metadata service schedules migrations by sending a `COPY_CHUNK(chunk_handle, source_server, dest_server)` command to the destination server.
   - The destination server pulls the chunk from the source server, verifies the checksum, and reports completion to the metadata service.
   - The metadata service updates the chunk location map to include the new replica, then (optionally) instructs the source server to delete its copy if the source is over-replicated.

5. **Throttling mechanisms:**
   - **Per-server concurrency limit:** Each chunk server handles at most 2 outbound migrations and 2 inbound migrations concurrently.
   - **Bandwidth cap:** Migration traffic is rate-limited to 100 MB/s per server, leaving the remaining bandwidth for client traffic.
   - **Cluster-wide pacing:** The metadata service limits total concurrent migrations to 5% of chunk servers (e.g., 25 servers migrating at once in a 500-server cluster).

6. **Handling new servers joining:**
   - When a new rack of 10 servers joins, they register with the metadata service with 0% utilization.
   - The rebalancer immediately identifies them as high-priority destinations (score = -1.0 if utilization is 0% and target is 60%).
   - Migrations are scheduled from the most over-utilized servers to the new servers, respecting rack-awareness (new servers are in a new rack, so they are valid destinations for chunks from any existing rack).
   - **Prioritization:** Chunks on servers with >85% utilization are migrated first to prevent those servers from filling up completely.
   - **Timeline:** With 10 new servers each accepting 100 MB/s of migration traffic, the cluster can migrate 1 GB/s. To fill each new server to 60% of 10 TB (6 TB), the migration takes approximately 6000 seconds (~1.7 hours) per server, but all 10 servers fill in parallel.

7. **Rebalancing during failures:**
   - If a server fails, the metadata service first prioritizes re-replication (restoring chunks to 3 replicas) over utilization rebalancing.
   - Re-replication uses the same migration mechanism but with higher priority and relaxed throttling (e.g., 200 MB/s per server) to restore fault tolerance quickly.

</details>

---

## Problem 4 — Consistency Model for Concurrent Writes and Appends

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design the consistency model and write protocol for a distributed file storage system that supports concurrent appends from multiple clients to the same file. The system must define what guarantees clients receive when multiple writers append to a shared log file simultaneously, how replicas are kept consistent when network partitions or slow nodes cause divergence, and how readers can determine which portions of a file contain valid, committed data.

### Scenario

**Context:** The distributed file system is used by a data pipeline where hundreds of worker processes append records to shared log files. Each worker opens the same file (e.g., `/logs/events/2024-01-15.log`) and appends batches of event records. Because workers run on different machines and append concurrently, the system must handle the case where two workers try to append to the same chunk at the same time. The team has observed that under high concurrency, some appended records appear duplicated or interleaved in unexpected ways. Additionally, when a chunk server is slow to acknowledge a write, the replicas can diverge — one replica has the appended data while another does not. The team needs a well-defined consistency model that clients can reason about, along with a protocol that keeps replicas consistent.

**Requirements:** Define the consistency semantics for concurrent appends — will the system guarantee that each append is atomic (all-or-nothing), or can partial appends occur? Design the write protocol for appending data to a chunk that is replicated across three servers, including how a primary replica coordinates with secondaries. Explain what happens when one secondary is slow or unreachable during an append — does the write succeed or fail? Define how the system detects and repairs inconsistent replicas. Describe how readers distinguish committed data from uncommitted or corrupted data within a chunk.

**Expected Approach:** Designate one replica as the primary for each chunk, responsible for serializing all appends and assigning byte offsets. The primary forwards each append to secondaries and waits for all replicas to acknowledge before confirming success. If a secondary fails, the primary reports the failure to the metadata service, which re-replicates the chunk. Use record-level checksums and a commit marker so readers can identify valid data. Define "at least once" append semantics — a failed append may be retried and result in a duplicate record, which readers must handle.

<details>
<summary>Hints</summary>

1. A primary-based write protocol serializes concurrent appends: the primary assigns each append a byte offset within the chunk, ensuring no two appends overlap. The primary then instructs all secondaries to write the data at the same offset, maintaining replica consistency.
2. If a secondary fails to acknowledge an append, the primary can either (a) abort the append and return an error to the client, or (b) succeed the append on the primary and the responsive secondary, then mark the failed secondary as stale. Option (b) provides higher availability but requires a mechanism to repair the stale replica later.
3. Each appended record should include a checksum and a record length header. Readers validate each record by checking the checksum. If a record is corrupted or incomplete (partial write due to a crash), the reader skips it and moves to the next record boundary.
4. "At least once" semantics mean that if a client does not receive an acknowledgment (e.g., network timeout), it retries the append, potentially creating a duplicate. Clients that need exactly-once semantics must include a unique record ID and deduplicate on the read side.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement a primary-based append protocol with serialized offset assignment, record-level checksums for integrity, and a stale-replica repair mechanism.

1. **Primary replica designation:**
   - For each chunk, the metadata service designates one of the three replicas as the primary. The primary holds a lease (e.g., 60 seconds, renewable) that grants it exclusive authority to serialize writes.
   - All append requests for a chunk are routed to the primary. If a client contacts a secondary, the secondary redirects to the primary.

2. **Append protocol (happy path):**
   ```
   1. Client sends APPEND(chunk_handle, data) to the primary.
   2. Primary assigns the next byte offset for this append:
      offset = chunk.current_size
      chunk.current_size += len(data)
   3. Primary writes the record to its local disk at the assigned offset:
      record = [record_length | checksum | data]
   4. Primary forwards WRITE(chunk_handle, offset, record) to secondary_1 and secondary_2.
   5. Both secondaries write the record at the specified offset and acknowledge.
   6. Primary receives both acknowledgments and sends SUCCESS(offset) to the client.
   ```
   - The primary serializes all concurrent appends by assigning offsets sequentially. Two clients appending simultaneously receive different offsets — their data does not interleave.

3. **Handling secondary failure during append:**
   - If secondary_1 acknowledges but secondary_2 does not respond within 5 seconds:
     1. The primary completes the append on itself and secondary_1 (2 of 3 replicas have the data).
     2. The primary reports secondary_2 as stale to the metadata service.
     3. The client receives SUCCESS — the data is durable on 2 replicas.
     4. The metadata service schedules a re-replication: a healthy chunk server copies the chunk from the primary to replace the stale secondary.
   - If both secondaries fail, the primary returns an error to the client. The client retries, and the metadata service may reassign the primary to a different set of servers.

4. **Record format and reader validation:**
   ```
   +----------------+----------------+------------------+
   | record_length  | CRC32 checksum | data payload     |
   | (4 bytes)      | (4 bytes)      | (variable)       |
   +----------------+----------------+------------------+
   ```
   - Readers scan the chunk sequentially. For each record:
     1. Read `record_length` to know how many bytes to read next.
     2. Read `checksum` and `data`.
     3. Compute CRC32 of `data` and compare with `checksum`. If they match, the record is valid. If not, the record is corrupted (partial write) — skip to the next record boundary.
   - A zero `record_length` indicates the end of valid data in the chunk.

5. **At-least-once append semantics and deduplication:**
   - If the client sends an append but does not receive an acknowledgment (network timeout), it retries with the same data. The primary treats this as a new append and assigns a new offset — the data appears twice in the chunk.
   - **Client-side deduplication:** Clients that need exactly-once semantics include a unique `record_id` (e.g., UUID) in the data payload. Downstream readers or processing pipelines deduplicate by `record_id`.
   - **Why not server-side deduplication:** Tracking every record ID on the server is expensive and adds complexity. Since the file system is a storage layer (not an application layer), deduplication is left to the application.

6. **Stale replica repair:**
   - When the metadata service detects a stale replica (secondary missed writes), it initiates a repair:
     1. The stale secondary is removed from the chunk's replica set.
     2. A new chunk server is selected (rack-aware) and copies the entire chunk from the primary.
     3. The new server is added to the replica set, restoring the replication factor to 3.
   - **Optimization:** For small divergences (secondary missed only the last few appends), the primary can send just the missing records instead of the entire chunk. The secondary applies them and catches up.

7. **Consistency guarantees summary:**
   - **Within a single append:** Atomic — the entire record is written or not. Partial records are detectable via checksum and skipped by readers.
   - **Across concurrent appends:** Serialized by the primary — each append occupies a distinct byte range. No interleaving.
   - **Across replicas:** Consistent after successful append (all acknowledging replicas have identical data). Stale replicas are detected and repaired.
   - **Client-visible semantics:** At-least-once delivery. Clients may see duplicate records after retries. Exactly-once requires application-level deduplication.

</details>

---

## Problem 5 — Garbage Collection, Compaction, and Storage Reclamation

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design the garbage collection and storage reclamation system for a distributed file storage system where files are frequently created, appended to, and deleted. The system must reclaim disk space from deleted files, handle orphaned chunks (chunks that are no longer referenced by any file due to partial failures during file creation), compact fragmented chunks to improve read performance, and do all of this without impacting the availability or performance of ongoing read and write operations.

### Scenario

**Context:** The distributed file system stores data for a machine learning platform. Training jobs create thousands of temporary checkpoint files (each 1–5 GB), use them for a few hours, and then delete them. Over six months of operation, the team notices that although 200 TB of files have been deleted, only 50 TB of disk space has been reclaimed. Investigation reveals three issues: (1) deleted files are marked as deleted in the metadata service but their chunks still occupy disk space on chunk servers, (2) some chunks are orphaned — they were allocated during file creation but the creation failed midway, leaving chunks that no file references, and (3) chunks that have been partially overwritten or had records deleted contain "holes" of wasted space that fragment sequential reads. The team needs a comprehensive garbage collection and compaction system.

**Requirements:** Design a lazy deletion mechanism where deleting a file does not immediately free disk space but instead marks the file for garbage collection. Explain why lazy deletion is preferred over immediate deletion in a distributed system. Design the garbage collection process that identifies and removes chunks belonging to deleted files. Design an orphan chunk detection mechanism that finds chunks not referenced by any file. Design a compaction process for chunks with significant wasted space. Specify how all of these processes run without blocking or degrading client operations. Address the race condition where a chunk is being garbage collected while a client is still reading it.

**Expected Approach:** Implement lazy deletion by renaming deleted files to a hidden trash namespace with a retention period. A background garbage collector periodically scans for files past their retention period, removes their chunk references from the metadata service, and instructs chunk servers to delete the physical data. Orphan detection compares the set of chunks reported by chunk servers (via heartbeats) against the set of chunks referenced in the metadata — any chunk in the former but not the latter is an orphan. Compaction rewrites fragmented chunks into new, dense chunks. All background processes are throttled and run at low priority.

<details>
<summary>Hints</summary>

1. Lazy deletion (moving to trash instead of immediate removal) provides a safety net against accidental deletions — files can be recovered within the retention period. It also simplifies the distributed deletion problem: the metadata service only needs to update the namespace (fast, local operation), and the expensive work of contacting chunk servers to delete data happens asynchronously in the background.
2. Orphan chunks arise when a file creation fails after some chunks have been allocated and written but before the file metadata is committed. The metadata service has no record of these chunks, but the chunk servers hold them. Detecting orphans requires comparing chunk server inventories against the metadata service's chunk registry.
3. For the race condition between garbage collection and active reads: use a reference-counting or lease-based mechanism. Before deleting a chunk, the garbage collector checks if any client holds an active read lease on it. If so, deletion is deferred until the lease expires.
4. Compaction is most valuable for chunks that serve as append logs (Problem 4) where records have been logically deleted or expired. The compactor reads valid records from the old chunk, writes them sequentially into a new chunk, and atomically swaps the chunk reference in the metadata service.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement a three-phase storage reclamation system — lazy deletion with trash namespace, background garbage collection with orphan detection, and low-priority chunk compaction — all running as throttled background processes.

1. **Lazy deletion mechanism:**
   - When a client deletes a file, the metadata service does not remove the file's metadata or chunk references. Instead, it moves the file to a hidden trash namespace:
     ```
     DELETE("/data/checkpoints/model-v3.ckpt")
     → Rename to "/.trash/data/checkpoints/model-v3.ckpt"
     → Set deletion_timestamp = now()
     → Set retention_period = 7 days (configurable)
     ```
   - The file is no longer visible in the normal namespace, but its chunks remain on disk.
   - During the retention period, the file can be recovered by moving it back to its original path (an "undelete" operation).
   - **Why lazy deletion:** (a) Immediate deletion would require the metadata service to synchronously contact all chunk servers holding the file's chunks — a slow, failure-prone operation that blocks the client. (b) Lazy deletion makes the delete operation O(1) — just a metadata rename. (c) It provides accidental deletion recovery.

2. **Background garbage collector:**
   ```
   function garbageCollect():
       # Phase 1: Identify expired trash files
       expired_files = metadata.query("SELECT * FROM trash WHERE deletion_timestamp + retention_period < now()")
       
       for each file in expired_files:
           # Phase 2: Remove chunk references
           chunk_handles = metadata.getChunks(file.id)
           for each chunk_handle in chunk_handles:
               metadata.removeChunkReference(file.id, chunk_handle)
               
               # If no other file references this chunk, mark it for deletion
               if metadata.getReferenceCount(chunk_handle) == 0:
                   metadata.markChunkForDeletion(chunk_handle)
           
           # Phase 3: Remove file metadata
           metadata.deleteFileRecord(file.id)
       
       # Phase 4: Instruct chunk servers to delete marked chunks
       chunks_to_delete = metadata.getChunksMarkedForDeletion()
       for each chunk in chunks_to_delete:
           for each server in chunk.replica_locations:
               server.deleteChunk(chunk.chunk_handle)
           metadata.removeChunkRecord(chunk.chunk_handle)
   ```
   - The garbage collector runs every hour (configurable) and processes files in batches of 1,000 to avoid overwhelming the metadata service.
   - **Throttling:** Chunk deletion requests to chunk servers are rate-limited to 50 deletions per second per server, ensuring garbage collection does not compete with client I/O.

3. **Orphan chunk detection:**
   - Every chunk server reports its full chunk inventory during periodic heartbeats (every 10 minutes for the full inventory, every 10 seconds for incremental updates).
   - The metadata service maintains the authoritative set of valid chunk handles (referenced by at least one file or in the trash namespace).
   - **Detection:** `orphan_chunks = chunk_server_inventory - metadata_valid_chunks`
   - Orphan chunks are not deleted immediately — they are placed in a quarantine list with a grace period (e.g., 24 hours). This handles the race condition where a file creation is in progress: chunks have been written to chunk servers but the file metadata has not yet been committed.
   - After the grace period, if the chunk is still not referenced, it is deleted.

4. **Chunk compaction:**
   - Applicable to chunks used as append logs (Problem 4) where records have been logically deleted or expired.
   - The compactor identifies chunks where wasted space exceeds a threshold (e.g., >30% of the chunk is dead records).
   - **Compaction process:**
     ```
     1. Read the source chunk sequentially, filtering out dead/expired records.
     2. Write valid records into a new chunk on the same set of servers (or a new set if rebalancing is needed).
     3. Atomically update the metadata service to replace the old chunk handle with the new chunk handle in the file's chunk list.
     4. Mark the old chunk for deletion (processed by the garbage collector).
     ```
   - Compaction runs at the lowest priority — it is paused entirely if the cluster is under heavy client load (e.g., >80% I/O utilization).

5. **Race condition: garbage collection vs. active reads:**
   - Before deleting a chunk, the garbage collector checks the metadata service for active read leases on that chunk.
   - When a client opens a file for reading, the metadata service issues a short-lived read lease (e.g., 1 hour) for each chunk. The lease is renewed if the read is still in progress.
   - The garbage collector skips chunks with active leases and retries in the next cycle.
   - **Alternative approach:** Use a reference count on each chunk. The count is incremented when a client starts reading and decremented when the read completes. Chunks with a reference count > 0 are not deleted.

6. **Storage reclamation timeline:**
   - **Immediate (on delete):** File disappears from the namespace. No disk space freed.
   - **After retention period (7 days):** Garbage collector removes chunk references and marks chunks for deletion.
   - **Within 1 hour of marking:** Chunk servers delete the physical data and report freed space.
   - **Total time from delete to disk reclamation:** 7 days + ~1 hour. For urgent reclamation, an admin can set the retention period to 0 or force-empty the trash.

7. **Monitoring and safety:**
   - Track metrics: trash size (bytes), orphan chunk count, compaction backlog, disk reclamation rate.
   - Alert if trash size grows faster than the garbage collector can process (indicates the GC is falling behind).
   - Alert if orphan chunk count spikes (indicates a bug in file creation or metadata corruption).
   - All deletion operations are logged to an audit trail for forensic analysis.

</details>
