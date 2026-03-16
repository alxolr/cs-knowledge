# Consistent Hashing

**Track:** Design Concepts
**Difficulty Tier:** Intermediate
**Prerequisites:** [Load Balancing](../load-balancing/README.md)

## Concept Overview

Consistent hashing is a distributed hashing technique that minimizes the number of keys that must be remapped when the set of servers (or nodes) changes. In a naive modular hashing scheme — `hash(key) % N` — adding or removing a single server changes the modulus and forces nearly every key to move to a different server. Consistent hashing solves this by mapping both keys and servers onto a fixed circular hash space (a "hash ring"). Each key is assigned to the first server encountered when walking clockwise around the ring from the key's hash position. When a server is added or removed, only the keys in the affected arc of the ring need to move, leaving the vast majority of mappings untouched.

The hash ring is typically the full output range of a hash function such as SHA-1 or MD5, treated as a circle where the maximum value wraps around to zero. Each server is hashed to one or more positions on this ring. A key is hashed to a point on the ring and then assigned to the nearest server in the clockwise direction. This simple geometric model makes it easy to reason about how keys redistribute when the topology changes: only keys that fall between a removed server and its predecessor are affected, and they simply move to the next server clockwise.

A well-known challenge with basic consistent hashing is load imbalance. If servers are hashed to only one position each, the arcs between them can vary wildly in size, causing some servers to own far more keys than others. The standard solution is virtual nodes (vnodes): each physical server is represented by many points on the ring, spreading its ownership across many small arcs. With enough virtual nodes — typically 100 to 200 per physical server — the key distribution becomes statistically uniform, and adding or removing a physical server redistributes keys evenly across the remaining servers rather than dumping them all onto a single neighbor.

Consistent hashing is a foundational building block in distributed systems. It underpins the data placement strategies in systems like Amazon DynamoDB, Apache Cassandra, and Akamai's CDN. Understanding how the hash ring works, how virtual nodes improve balance, and how node changes affect key ownership is essential for designing any system that distributes data or load across a dynamic set of servers.

---

## Problem 1 — Mapping Keys on a Basic Consistent Hash Ring

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a basic consistent hash ring for distributing cache keys across a small cluster of servers. Given a set of servers and a set of keys, explain how each entity is placed on the ring, how key-to-server assignment works, and trace through the assignment for a concrete example. Your design should clearly describe the ring structure, the clockwise lookup rule, and what happens when a key hashes to a position past the last server on the ring (the wrap-around case).

### Scenario

**Context:** A development team is building an in-memory cache layer with four servers (A, B, C, D). They currently use `hash(key) % 4` to assign keys to servers. They want to understand how a consistent hash ring would work as a replacement before adopting it, starting with the simplest possible version — one position per server on the ring.

**Requirements:** Describe how to construct a hash ring with the four servers. Place the servers at specific positions on a ring of size 360 (using degrees as a simplified hash space). Assign five cache keys to servers using the clockwise lookup rule. Show the full assignment trace. Explain the wrap-around case where a key's hash position is greater than the highest server position.

**Expected Approach:** Hash each server name to a position on the 0–359 ring. Sort the server positions. For each key, hash it to a ring position and walk clockwise to find the first server. If no server exists clockwise before wrapping, assign to the first server on the ring (wrap-around). Present the assignments in a clear table.

<details>
<summary>Hints</summary>

1. Think of the ring as a sorted circular array of server positions. For any key position, you need the first server position that is greater than or equal to the key position. If none exists, wrap around to the first server in the sorted list.
2. In a real system the hash space is 0 to 2^32 − 1 (or similar). Using 0–359 simplifies the example while preserving the same logic.
3. The wrap-around case is the defining feature that makes the hash space a "ring" rather than a line — the position after 359 is 0.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Construct a ring of size 360, place four servers, and assign five keys using clockwise lookup.

1. **Place servers on the ring:**
   - Hash each server identifier to a position: A → 45, B → 130, C → 220, D → 310 (example positions).
   - Sorted server positions: [45, 130, 220, 310].

2. **Clockwise lookup rule:**
   - For a key with hash position `p`, find the smallest server position `s` such that `s >= p`.
   - If no such `s` exists (i.e., `p > 310`), wrap around to the first server (position 45, server A).

3. **Assign five keys:**
   - Key "user:1001" → hash position 70 → next server clockwise is B (130).
   - Key "product:42" → hash position 200 → next server clockwise is C (220).
   - Key "session:abc" → hash position 315 → no server between 315 and 359, wrap around → A (45).
   - Key "cart:777" → hash position 45 → exact match → A (45).
   - Key "order:55" → hash position 131 → next server clockwise is C (220).

4. **Assignment summary:**

   | Key | Hash Position | Assigned Server | Server Position |
   |-----|--------------|-----------------|-----------------|
   | user:1001 | 70 | B | 130 |
   | product:42 | 200 | C | 220 |
   | session:abc | 315 | A (wrap) | 45 |
   | cart:777 | 45 | A (exact) | 45 |
   | order:55 | 131 | C | 220 |

5. **Observations:**
   - The distribution is uneven: A owns 2 keys, B owns 1, C owns 2, D owns 0. This imbalance is expected with only one position per server and a small number of keys.
   - The wrap-around for "session:abc" demonstrates the circular nature of the ring.
   - With more keys, the distribution would still be uneven because the arc sizes differ (A owns 45° + 95° of wrap-around = 95°, B owns 85°, C owns 90°, D owns 90°). Virtual nodes address this imbalance.

</details>

---

## Problem 2 — Introducing Virtual Nodes for Uniform Key Distribution

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Design a virtual node strategy for a consistent hash ring that achieves near-uniform key distribution across a heterogeneous cluster. The cluster contains servers with different capacities, and the virtual node count per server should reflect its relative capacity. Your design should explain how virtual nodes are created, how they map back to physical servers, and quantify the improvement in distribution uniformity compared to a single-node-per-server approach.

### Scenario

**Context:** An e-commerce platform runs a distributed session store on six servers. Two are high-capacity machines (64 GB RAM) and four are standard machines (32 GB RAM). With one hash ring position per server, the key distribution is highly skewed — one server holds 30% of all sessions while another holds only 8%. The operations team wants to rebalance the ring so that high-capacity servers store roughly twice as many sessions as standard servers, and the overall distribution is smooth.

**Requirements:** Assign virtual nodes to each server proportional to its capacity. Describe how virtual node identifiers are generated (e.g., "server-A-vnode-0", "server-A-vnode-1", …). Explain how a key lookup resolves a virtual node back to its physical server. Calculate the expected key ownership percentage for each physical server. Compare the standard deviation of key ownership with 1 node per server vs. 150/75 virtual nodes per server. The design should also address what happens to virtual nodes when a physical server is removed.

**Expected Approach:** Assign 150 virtual nodes to each high-capacity server and 75 to each standard server. Total virtual nodes = 2×150 + 4×75 = 600. Each high-capacity server owns 150/600 = 25% of the ring, each standard server owns 75/600 = 12.5%. Explain that virtual node identifiers are hashed independently to spread positions across the ring. When a physical server is removed, all its virtual nodes are removed, and the keys from each virtual node's arc move to the next clockwise virtual node (which belongs to a different physical server), distributing the load across multiple servers rather than one.

<details>
<summary>Hints</summary>

1. Virtual node identifiers are typically formed by appending an index to the server name: `hash("server-A#0")`, `hash("server-A#1")`, …, `hash("server-A#149")`. Each produces an independent position on the ring.
2. With 600 virtual nodes on the ring, the average arc size is 360°/600 = 0.6° (or in a real 2^32 ring, about 7.1 million hash positions per arc). The more virtual nodes, the smaller and more uniform the arcs become.
3. When looking up a key, you find the nearest virtual node clockwise, then map that virtual node back to its physical server. The mapping is a simple lookup table: virtual node ID → physical server.
4. Removing a physical server removes all its virtual nodes simultaneously. Because those virtual nodes are scattered across the ring, the freed keys are absorbed by many different neighboring virtual nodes belonging to different physical servers — this is the key advantage over single-node hashing.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use capacity-proportional virtual nodes to achieve uniform distribution, with a virtual-to-physical mapping table for lookups.

1. **Virtual node assignment:**
   - High-capacity servers (H1, H2): 150 virtual nodes each.
   - Standard servers (S1, S2, S3, S4): 75 virtual nodes each.
   - Total virtual nodes on the ring: 2×150 + 4×75 = 600.

2. **Virtual node identifier generation:**
   - For server H1: hash("H1#0"), hash("H1#1"), …, hash("H1#149") → 150 positions on the ring.
   - For server S1: hash("S1#0"), hash("S1#1"), …, hash("S1#74") → 75 positions on the ring.
   - Each hash produces an independent, pseudo-random position, spreading the virtual nodes uniformly across the ring.

3. **Key lookup process:**
   - Hash the key to a ring position.
   - Find the first virtual node clockwise from that position (binary search on the sorted list of 600 virtual node positions).
   - Look up the virtual node's physical server in the mapping table (e.g., "H1#47" → server H1).
   - Route the request to that physical server.

4. **Expected key ownership:**
   - H1: 150/600 = 25.0% of keys.
   - H2: 150/600 = 25.0% of keys.
   - S1 through S4: 75/600 = 12.5% of keys each.
   - High-capacity servers hold exactly 2× the keys of standard servers, matching the 2:1 capacity ratio.

5. **Distribution uniformity comparison:**
   - **1 node per server (6 positions):** Arc sizes vary widely. Standard deviation of key ownership across servers is approximately 10–15% of total keys (some servers own 30%, others 8%).
   - **150/75 virtual nodes (600 positions):** Arc sizes are nearly uniform. Standard deviation drops to approximately 1–2% of total keys. Each server's actual ownership is within a few percent of its expected share.

6. **Removing a physical server:**
   - If S3 is removed, all 75 of its virtual nodes are deleted from the ring.
   - Each of those 75 arcs is absorbed by the next clockwise virtual node, which belongs to a variety of other physical servers (H1, H2, S1, S2, S4).
   - The 12.5% of keys formerly on S3 are spread across roughly 5 other servers, each absorbing about 2–3% more keys. No single server is overwhelmed.

**Why virtual nodes work:** By replacing each physical server with many ring positions, virtual nodes convert the discrete, lumpy distribution of a few points into a smooth, near-continuous distribution. The law of large numbers ensures that with enough virtual nodes, each server's share of the ring converges to its expected proportion.

</details>

---

## Problem 3 — Adding and Removing Nodes with Minimal Key Redistribution

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design the operational procedure for adding a new server to and removing an existing server from a consistent hash ring that is actively serving traffic. Your design should quantify exactly which keys move during each operation, describe how to migrate data without downtime, and compare the redistribution cost against naive modular hashing. The focus is on the mechanics of key movement and the operational steps to execute a topology change safely.

### Scenario

**Context:** A content delivery network (CDN) caches web assets across 12 edge servers using a consistent hash ring with 100 virtual nodes per server (1,200 total virtual nodes). The system stores 6 million cached objects. During a traffic surge, the team needs to add a 13th server to absorb load. Later, they plan to decommission an aging server for hardware replacement. Both operations must happen without cache downtime — clients should continue to receive cached content throughout the transition, with only a minimal, temporary increase in cache misses for the keys that are being migrated.

**Requirements:** For the add-server operation: calculate how many of the 6 million keys must move, identify which existing servers lose keys and how many each loses, and describe a step-by-step migration procedure that avoids downtime. For the remove-server operation: calculate the redistribution and describe the drain procedure. Compare the number of keys moved in both operations against what `hash(key) % N` would require. The design should include a rollback plan if the new server fails health checks during the add operation.

**Expected Approach:** Adding a 13th server with 100 virtual nodes increases the total to 1,300. The new server claims 100/1,300 ≈ 7.7% of the ring, so approximately 462,000 keys move — and they come from all 12 existing servers proportionally, not from a single neighbor. For removal, the departing server's 100/1,200 ≈ 8.3% of keys (about 500,000) are redistributed across the remaining 11 servers. Compare this to modular hashing where changing N from 12 to 13 moves roughly 11/13 ≈ 84.6% of all keys (about 5.08 million).

<details>
<summary>Hints</summary>

1. When a new server's virtual nodes are inserted into the ring, each new virtual node "steals" a portion of the arc from the existing virtual node that was previously responsible for that region. The keys in the stolen arc move to the new server. Since the 100 new virtual nodes are scattered across the ring, the stolen keys come from many different existing servers.
2. The migration can be done in two phases: (a) add the new server to the ring so new writes go to it, but also check the old server on cache misses (double-read), (b) background-migrate existing keys from old servers to the new one, (c) once migration is complete, stop the double-read.
3. For removal, the reverse applies: before removing the server, migrate its keys to their new owners (the next clockwise virtual nodes). Once migration is confirmed, remove the server from the ring.
4. Rollback for a failed add: remove the new server's virtual nodes from the ring. Keys that were already migrated are now orphaned on the new server, but the old servers still have them (if you used a copy-then-delete migration strategy). Clients fall back to the old servers seamlessly.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Execute topology changes in phased migrations with double-read fallback, quantifying key movement at each step.

1. **Adding server 13 — key movement calculation:**
   - Current ring: 12 servers × 100 virtual nodes = 1,200 virtual nodes, 6 million keys.
   - New ring: 13 servers × 100 virtual nodes = 1,300 virtual nodes.
   - Server 13 claims 100/1,300 ≈ 7.69% of the ring ≈ 461,538 keys.
   - These keys come proportionally from all 12 existing servers. Each existing server loses approximately 461,538 / 12 ≈ 38,462 keys (about 0.64% of total keys per server).
   - **Comparison with modular hashing:** Changing from `% 12` to `% 13` would remap approximately 5,076,923 keys (84.6%). Consistent hashing moves 12× fewer keys.

2. **Add-server migration procedure:**
   - **Phase 1 — Ring update with double-read:** Insert server 13's 100 virtual nodes into the ring. New key lookups now resolve to server 13 for its arcs. On a cache miss at server 13, the system performs a fallback read from the previous owner (the server that owned the arc before the insertion). This ensures zero downtime — every key is still accessible.
   - **Phase 2 — Background migration:** A migration worker iterates through the keys that now belong to server 13 (identified by re-hashing keys on the old servers and checking if they fall in server 13's arcs). It copies each key to server 13 and deletes it from the old server.
   - **Phase 3 — Disable double-read:** Once migration is complete (verified by checking that server 13 can serve all its keys without fallback), disable the double-read path. Server 13 is now fully integrated.

3. **Removing a server — key movement calculation:**
   - Removing one server's 100 virtual nodes from the 1,200-node ring.
   - The departing server owns 100/1,200 ≈ 8.33% of keys ≈ 500,000 keys.
   - These keys redistribute to the next clockwise virtual nodes, which belong to various remaining servers. Each of the 11 remaining servers absorbs approximately 500,000 / 11 ≈ 45,455 keys.

4. **Remove-server drain procedure:**
   - **Phase 1 — Drain:** Mark the server as "draining" — it still serves reads but receives no new writes. New key lookups skip its virtual nodes.
   - **Phase 2 — Migrate:** Copy all keys from the draining server to their new owners (determined by the ring without the draining server's virtual nodes).
   - **Phase 3 — Remove:** Once all keys are migrated and verified, remove the server's virtual nodes from the ring and shut down the server.

5. **Rollback plan for failed add:**
   - If server 13 fails health checks during Phase 1 or Phase 2, remove its virtual nodes from the ring.
   - Keys already copied to server 13 are abandoned (they still exist on the original servers because the migration uses copy-then-delete, and deletes have not yet occurred).
   - The ring reverts to 1,200 virtual nodes and all lookups resolve to the original servers. No data is lost.

**Why consistent hashing minimizes redistribution:** Each topology change only affects the arcs adjacent to the added or removed virtual nodes. The rest of the ring — and the vast majority of keys — is untouched. This property makes consistent hashing practical for systems that frequently scale up and down.

</details>

---

## Problem 4 — Consistent Hashing for a Distributed Cache with Replication

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design a distributed caching system that uses consistent hashing for key placement and replicates each cached entry to multiple nodes for fault tolerance. The system must handle node failures gracefully — when a cache node goes down, clients should still be able to read cached data from a replica without a cache miss. Your design should address replica placement on the hash ring, read and write paths, consistency guarantees, and how replication interacts with node addition and removal.

### Scenario

**Context:** A social media platform caches user profile data across 20 cache nodes using a consistent hash ring with 150 virtual nodes per server. The cache serves 200,000 reads per second and tolerates a cache miss rate of under 2%. When a cache node crashes, all keys owned by that node become cache misses until the data is re-fetched from the database and re-cached — this temporarily spikes the miss rate to 7% and overloads the database. The team wants to replicate each cache entry to two additional nodes so that a single node failure causes zero increase in cache misses.

**Requirements:** Design a replication strategy where each key is stored on its primary node (determined by the hash ring) and on the next N−1 distinct physical nodes clockwise on the ring. Define N=3 (one primary + two replicas). Describe the write path (how a new cache entry is written to all three nodes), the read path (how a client reads from the primary and falls back to replicas), and the failure handling (what happens when the primary node crashes). Address the "replica on same physical server" problem when using virtual nodes. Explain how adding or removing a node affects replicated keys and what additional data migration is required beyond the primary key movement.

**Expected Approach:** For replica placement, walk clockwise from the key's position and select the next 3 distinct physical servers (skipping virtual nodes that belong to the same physical server as one already selected). Writes go to all 3 nodes (synchronous to primary, asynchronous to replicas for low latency). Reads go to the primary; on failure, the client retries on the first replica, then the second. When a node fails, its keys are served by replicas with zero miss increase. When a node is added, some keys gain a new primary or new replica positions, requiring both primary migration and replica rebalancing.

<details>
<summary>Hints</summary>

1. The "next N distinct physical servers" rule is critical when using virtual nodes. If you simply take the next 2 virtual nodes clockwise, they might belong to the same physical server as the primary — giving you no fault tolerance. Always skip virtual nodes until you find ones belonging to different physical servers.
2. For the write path, a common pattern is synchronous write to the primary (the client waits for confirmation) and asynchronous replication to the two replicas (fire-and-forget or background queue). This keeps write latency low while still achieving replication.
3. When a node is added, some existing keys change their primary assignment (as in basic consistent hashing). But replicas also shift: a key that was replicated to servers B and C might now be replicated to the new server and B. The migration must update both primary and replica placements.
4. Consider what happens if two nodes fail simultaneously. With N=3, the system can tolerate one failure with zero miss increase. Two simultaneous failures may cause misses for keys where both replicas are on the failed nodes — quantify this probability.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use consistent hashing with N=3 replication (primary + 2 replicas) placed on distinct physical servers clockwise on the ring, with synchronous primary writes and asynchronous replica writes.

1. **Replica placement:**
   - For a key at ring position `p`, walk clockwise through virtual nodes.
   - The first virtual node encountered determines the primary physical server (e.g., server A).
   - Continue clockwise, skipping any virtual node belonging to server A, until a virtual node belonging to a different physical server is found (e.g., server B) — this is replica 1.
   - Continue clockwise, skipping virtual nodes belonging to A or B, until a third distinct physical server is found (e.g., server C) — this is replica 2.
   - The key is stored on servers A, B, and C.

2. **Write path:**
   - Client sends a write (cache set) to the primary node (server A).
   - Server A stores the entry locally and acknowledges the write to the client (synchronous).
   - Server A asynchronously forwards the entry to replicas B and C via a replication queue.
   - Replicas store the entry and acknowledge back to A (for monitoring, not for client latency).
   - Write latency ≈ primary write time only (e.g., 1–2 ms). Replication completes within 5–10 ms in the background.

3. **Read path:**
   - Client hashes the key, determines the primary (server A), and sends a read request.
   - If server A responds, return the cached value.
   - If server A is unreachable (timeout after 50 ms), retry on replica 1 (server B).
   - If server B is also unreachable, retry on replica 2 (server C).
   - If all three are unreachable, fall back to the database (cache miss).

4. **Node failure handling:**
   - Server A crashes. All keys where A is the primary are now served by replica B (the client's retry logic handles this automatically).
   - Cache miss rate remains near 0% for A's keys because B and C have copies.
   - A background process detects A's failure and promotes B to primary for affected keys. A new replica is created on the next eligible server clockwise to restore N=3 replication.

5. **Interaction with node addition/removal:**
   - **Adding a node:** Some keys gain a new primary (as in basic consistent hashing). Additionally, some keys' replica lists change — the new node may become a replica for keys it was not previously responsible for. Migration must copy both primary and replica data to the new node.
   - **Removing a node:** The node's primary keys move to the next server (as in basic consistent hashing). Its replica keys must be re-replicated to a new third server to maintain N=3. The drain procedure must ensure all replicas are rebuilt before the node is shut down.

6. **Dual failure probability:**
   - With 20 nodes and N=3, a key's replicas are on 3 out of 20 servers.
   - Probability that both replicas fail simultaneously (given one failure): the second failed node must be one of the 2 remaining replicas out of 19 remaining nodes = 2/19 ≈ 10.5%.
   - For the overall cache: if 2 random nodes fail, approximately 10.5% of the failed nodes' keys (about 0.5% of total keys) lose all replicas. This is acceptable for a cache (data can be re-fetched from the database).

**Why N=3 replication on the hash ring works:** It provides single-node fault tolerance with zero cache miss increase, the replica placement is deterministic (any client can compute it from the ring), and the replication cost is modest (3× storage, but cache entries are typically small). The consistent hash ring ensures that replica assignments shift minimally when the topology changes.

</details>

---

## Problem 5 — Consistent Hashing for Database Sharding with Hotspot Mitigation

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a database sharding strategy based on consistent hashing that distributes rows across multiple database instances while handling hotspot keys — keys that receive disproportionately high read or write traffic. Your design should cover shard assignment, query routing, handling of range queries across shards, and a strategy for detecting and mitigating hotspots without resharding the entire cluster.

### Scenario

**Context:** A ride-sharing platform stores trip records in a sharded database cluster of 8 instances. Each trip record is keyed by `trip_id`. The system processes 50,000 writes per second (new trips) and 300,000 reads per second (trip lookups, driver history, rider history). The team chose consistent hashing on `trip_id` to distribute data across shards. However, two problems have emerged. First, certain celebrity drivers generate extreme read traffic on their trip records — a single shard holding a popular driver's recent trips receives 10× the read load of other shards. Second, the analytics team needs to run range queries (e.g., "all trips in the last hour") that span all shards, and the current design requires scatter-gather across all 8 instances for every range query, which is slow.

**Requirements:** Design the sharding scheme using consistent hashing with virtual nodes. Address the hotspot problem by designing a read-replica or key-splitting strategy that distributes hot key reads across multiple shards without duplicating the entire shard's data. Design an approach for range queries that minimizes the number of shards contacted. Explain how new shards are added to the cluster (scaling from 8 to 12 instances) and how data is migrated. The design must maintain consistency for writes (a trip record is written to exactly one authoritative shard) while allowing reads to be served from replicas or caches.

**Expected Approach:** Use consistent hashing with 200 virtual nodes per shard for uniform distribution of trip records by `trip_id`. For hotspots, implement a hot-key detection layer that identifies keys exceeding a read threshold and creates read replicas of those specific keys on adjacent shards (not full shard replication). For range queries, maintain a secondary time-based index that partitions trips by time bucket, allowing the query router to contact only the shards that contain data for the requested time range. For scaling, add new shards with virtual nodes and migrate only the affected key ranges using the consistent hashing redistribution property.

<details>
<summary>Hints</summary>

1. Hotspot mitigation at the key level is more efficient than shard-level replication. If only 0.1% of keys are hot, replicating the entire shard wastes storage and complicates consistency. Instead, identify hot keys (e.g., keys exceeding 1,000 reads/second) and replicate only those keys to 2–3 additional shards as read-only copies.
2. For range queries, consistent hashing by `trip_id` scatters trips randomly across shards — there is no locality by time. A secondary index that maps time buckets (e.g., 1-hour windows) to the set of shards containing trips from that window allows the query router to skip shards that have no relevant data. This does not eliminate scatter-gather entirely but reduces the fan-out.
3. When adding shards, consistent hashing ensures only ~1/N of keys move (where N is the new total). But for a database (unlike a cache), the migration must be transactional — the old shard must stop accepting writes for migrating keys, transfer the data, and then the new shard takes over. A two-phase approach (copy data, then switch ownership) minimizes downtime.
4. Write consistency is maintained by the rule that each key has exactly one authoritative shard (the primary on the hash ring). Read replicas of hot keys are asynchronously updated and may serve slightly stale data — this is acceptable for read-heavy workloads like trip history lookups.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Consistent hashing with virtual nodes for shard assignment, per-key hot replica promotion for hotspot mitigation, a time-bucketed secondary index for range query optimization, and phased migration for cluster scaling.

1. **Shard assignment:**
   - 8 database instances, each with 200 virtual nodes on the hash ring (1,600 total virtual nodes).
   - Each trip record is assigned to a shard by hashing `trip_id` and finding the next clockwise virtual node.
   - Each shard owns approximately 1/8 of all trip records (about 6,250 writes/second and 37,500 reads/second per shard under uniform distribution).

2. **Hotspot detection and mitigation:**
   - A monitoring layer tracks per-key read rates on each shard. Keys exceeding a threshold (e.g., 1,000 reads/second) are flagged as hot.
   - For each hot key, the system creates read-only replicas on 2 additional shards (chosen as the next distinct physical shards clockwise on the ring, reusing the same replica placement logic as Problem 4).
   - The query router maintains a hot-key table: `{trip_id → [primary_shard, replica_shard_1, replica_shard_2]}`. Read requests for hot keys are load-balanced across the primary and replicas using round-robin.
   - Replicas are updated asynchronously from the primary shard's write-ahead log. Staleness is bounded to under 1 second, which is acceptable for trip history reads.
   - When a key's read rate drops below the threshold for a sustained period (e.g., 10 minutes), its replicas are removed to free resources.

3. **Range query optimization:**
   - Maintain a secondary index that maps time buckets (1-hour windows) to the set of shard IDs that contain trips created during that window.
   - When a trip is written, the shard reports its shard ID and the trip's timestamp to a lightweight index service, which updates the time-bucket → shard mapping.
   - For a range query like "all trips in the last hour," the query router consults the index to find which shards have trips in that time bucket (e.g., shards 2, 4, 5, 7) and sends the query only to those shards, skipping the other four.
   - This reduces fan-out from 8 shards to an average of 4–6 shards per range query (depending on write distribution during the time window). For narrow time ranges (e.g., "last 5 minutes"), fan-out may drop to 2–3 shards.

4. **Scaling from 8 to 12 shards:**
   - Add 4 new shard instances, each with 200 virtual nodes. The ring grows from 1,600 to 2,400 virtual nodes.
   - Each new shard claims approximately 200/2,400 ≈ 8.3% of the ring. Total keys migrated: 4 × 8.3% ≈ 33.3% of all trip records.
   - **Migration procedure:**
     - Phase 1: Add new shards to the ring in read-only mode. They accept no writes yet.
     - Phase 2: For each key range that moved to a new shard, copy the data from the old shard to the new shard. The old shard continues to serve reads and writes during the copy.
     - Phase 3: Once a key range is fully copied, atomically switch ownership: the old shard stops accepting writes for those keys, the new shard starts accepting writes. A brief write pause (< 100 ms per key range) ensures no writes are lost.
     - Phase 4: After all ranges are migrated, the old shards delete the migrated data to reclaim storage.
   - Rollback: If a new shard fails during migration, its virtual nodes are removed from the ring and the old shards resume full ownership. Copied data on the failed shard is discarded.

5. **Write consistency guarantee:**
   - Every key has exactly one authoritative shard at any point in time (the primary on the hash ring).
   - All writes go to the primary shard. Read replicas (for hot keys) are asynchronous and eventually consistent.
   - During migration, the ownership transfer is atomic per key range — there is no window where two shards accept writes for the same key.

**Why consistent hashing fits database sharding:** It provides predictable, minimal data movement when scaling (only ~1/N of keys move per added shard), deterministic shard assignment that any query router can compute independently, and a natural framework for replica placement. The per-key hotspot mitigation layer and time-bucketed secondary index address the two main limitations of hash-based sharding (hotspots and lack of range locality) without abandoning the consistent hashing foundation.

</details>
