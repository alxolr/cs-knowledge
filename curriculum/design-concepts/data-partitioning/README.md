# Data Partitioning

**Track:** Design Concepts
**Difficulty Tier:** Intermediate
**Prerequisites:** [Database Schema Design](../database-schema-design/README.md), [Consistent Hashing](../consistent-hashing/README.md), [CAP Theorem](../cap-theorem/README.md)

## Concept Overview

Data partitioning (also called sharding) is the practice of splitting a large dataset across multiple storage units — servers, disks, or database instances — so that no single unit is responsible for the entire dataset. As systems grow beyond the capacity of a single machine in terms of storage, memory, or throughput, partitioning becomes the primary mechanism for horizontal scalability. Each partition holds a subset of the data and can be read from and written to independently, allowing the system to serve more traffic in parallel and store more data than any individual node could handle alone.

There are two fundamental axes of partitioning. Vertical partitioning splits a table by columns — frequently accessed columns stay in a "hot" partition while rarely used or large columns move to a separate store. Horizontal partitioning splits a table by rows — each partition contains a complete set of columns but only a subset of rows, determined by a partitioning function applied to a chosen partition key. Most distributed systems rely on horizontal partitioning because it scales linearly: doubling the number of partitions roughly doubles the system's capacity for both storage and throughput.

The choice of partitioning strategy has deep implications for query performance, data locality, and operational complexity. Hash-based partitioning distributes rows uniformly but destroys ordering, making range queries expensive. Range-based partitioning preserves ordering and supports efficient range scans but risks hotspots when traffic concentrates on a narrow key range. Composite strategies combine both approaches — for example, partitioning first by a hash of the tenant ID and then by a time range within each hash partition — to balance uniformity with query efficiency. Understanding these trade-offs is essential for designing any system that must scale beyond a single database instance.

Real-world systems rarely partition in isolation. Partitioning interacts with replication (each partition is typically replicated for fault tolerance), with consistency models (cross-partition transactions are expensive under the CAP theorem), and with query routing (the application or a middleware layer must know which partition holds the requested data). Mastering data partitioning means understanding not just how to split data, but how that split affects every other layer of the system.

---

## Problem 1 — Choosing Between Horizontal and Vertical Partitioning

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a partitioning strategy for a database table that suffers from both wide-column bloat and high row volume. Determine which columns should be vertically partitioned into separate stores and how the remaining rows should be horizontally partitioned across multiple database instances. Your design should justify each partitioning decision based on access patterns and explain how the application reconstructs a full record when needed.

### Scenario

**Context:** An e-commerce platform stores product data in a single `products` table with 50 million rows. The table has 35 columns including small, frequently read fields (`product_id`, `name`, `price`, `category`, `stock_count`), medium-frequency fields (`description`, `specifications`), and large, rarely accessed fields (`full_resolution_images` as binary blobs averaging 5 MB each, `user_manual_pdf` averaging 2 MB). The product catalog page reads the small fields at 100,000 queries per second, while the full product detail page (which needs descriptions and specs) is accessed at 10,000 queries per second. Image and PDF downloads happen at only 500 requests per second. The single database instance is running out of both storage (the blob columns consume 90% of disk) and throughput (the catalog queries are bottlenecked by I/O because the large blob columns pollute the buffer pool).

**Requirements:** Design a vertical partitioning scheme that separates columns by access frequency and size. Then design a horizontal partitioning scheme for the high-volume catalog data. Explain how the application layer joins data from the vertical partitions when a full product detail is requested. Quantify the storage and throughput improvements from each partitioning decision.

**Expected Approach:** Vertically partition into three stores: a primary relational database for the 5 hot columns, a secondary store for descriptions and specifications, and an object store (e.g., S3) for binary blobs. Horizontally partition the primary store by hashing `product_id` across 4 database instances. The application joins vertical partitions by `product_id` at the application layer when a full detail view is needed.

<details>
<summary>Hints</summary>

1. Vertical partitioning is most effective when columns have vastly different access frequencies or sizes. Moving the 5 MB image blobs out of the relational database immediately frees 90% of disk and eliminates buffer pool pollution for the hot catalog queries.
2. After vertical partitioning, the primary table shrinks dramatically (only small columns remain), making horizontal partitioning straightforward — each partition holds a manageable number of rows with small row sizes.
3. The application-layer join for full product details is a simple two-step lookup: fetch the core fields from the primary store, then fetch the description from the secondary store using the same `product_id`. This adds one extra network call but is only needed for 10% of queries.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Vertically partition by column access frequency and size, then horizontally partition the hot columns by hash of `product_id`.

1. **Vertical partitioning into three stores:**
   - **Primary store (relational DB):** `product_id`, `name`, `price`, `category`, `stock_count`. Row size ≈ 200 bytes. Total storage: 50M × 200 bytes ≈ 10 GB. Serves 100,000 catalog queries/second.
   - **Secondary store (document DB or relational):** `product_id`, `description`, `specifications`. Row size ≈ 5 KB. Total storage: 50M × 5 KB ≈ 250 GB. Serves 10,000 detail queries/second.
   - **Object store (S3 or equivalent):** `full_resolution_images`, `user_manual_pdf`, keyed by `product_id`. Average object size ≈ 7 MB. Total storage: 50M × 7 MB ≈ 350 TB. Serves 500 download requests/second.

2. **Storage improvement:** The primary relational database drops from ~350 TB (dominated by blobs) to ~10 GB. Buffer pool utilization improves dramatically because every cached page now contains useful catalog data instead of blob fragments.

3. **Horizontal partitioning of the primary store:**
   - Partition key: `product_id`.
   - Strategy: Hash `product_id` modulo 4 (or use consistent hashing with 4 nodes).
   - Each of the 4 instances holds ~12.5 million rows and ~2.5 GB of data.
   - Each instance handles ~25,000 catalog queries/second (100,000 / 4), well within a single instance's capacity.

4. **Application-layer join for full detail view:**
   - Step 1: Query the primary store partition for core fields using `product_id` (< 1 ms).
   - Step 2: Query the secondary store for description and specifications using `product_id` (< 5 ms).
   - Step 3: If the user requests images or PDFs, fetch from the object store using `product_id` as the key (latency depends on object size, typically 50–200 ms).
   - Total detail view latency: ~6 ms for text data, plus optional blob download. This is acceptable because detail views are 10× less frequent than catalog views.

5. **Throughput improvement:** The primary store now fits entirely in memory (10 GB across 4 instances = 2.5 GB each), enabling sub-millisecond catalog lookups. The previous bottleneck — I/O contention from blob columns — is eliminated.

</details>

---

## Problem 2 — Designing a Hash-Based Partitioning Scheme

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Design a hash-based partitioning strategy for a high-throughput user activity logging system. The system must distribute writes uniformly across partitions, support efficient single-key lookups, and handle partition expansion (adding new nodes) with minimal data movement. Your design should specify the hash function, the partition mapping, and the strategy for rebalancing when the cluster grows.

### Scenario

**Context:** A mobile analytics platform ingests 500,000 event records per second from 80 million active users. Each event is keyed by `user_id` and contains a timestamp, event type, and a small JSON payload (average 400 bytes). The system currently runs on 16 database partitions using `hash(user_id) % 16` to assign events to partitions. The team needs to scale to 24 partitions to handle projected growth to 800,000 events per second. However, changing the modulus from 16 to 24 would remap the majority of existing records, requiring a massive and disruptive data migration. The team wants a partitioning scheme that supports incremental scaling without full reshuffling.

**Requirements:** Replace the naive modular hashing with a consistent hashing approach that supports growing from 16 to 24 partitions while moving only the minimum necessary data. Specify how events are assigned to partitions, how the query router locates a user's events, and how the migration from 16 to 24 partitions is executed. Calculate the percentage of data that must move under both the naive modular approach and your proposed approach. Address the risk of uneven partition sizes and describe how to detect and correct imbalance.

**Expected Approach:** Use consistent hashing with virtual nodes (e.g., 256 virtual nodes per partition) to distribute user events. When scaling from 16 to 24 partitions, only ~1/3 of data moves (the 8 new partitions each claim ~1/24 of the ring). Compare this to naive modular hashing where ~62.5% of data would need to move. Include a monitoring strategy for partition size imbalance and a virtual node reweighting mechanism to correct skew.

<details>
<summary>Hints</summary>

1. With naive `hash(user_id) % 16` changing to `% 24`, a key stays on the same partition only if `hash(user_id) % 16 == hash(user_id) % 24` — this happens for only about 37.5% of keys (those where the result modulo the LCM of 16 and 24 maps to the same partition under both). The remaining 62.5% must move.
2. With consistent hashing and 256 virtual nodes per partition, the ring has 16 × 256 = 4,096 virtual nodes initially. Adding 8 partitions adds 8 × 256 = 2,048 virtual nodes, growing the ring to 6,144. Each new partition claims 256/6,144 ≈ 4.17% of the ring. Total data moved: 8 × 4.17% ≈ 33.3%.
3. To detect partition imbalance, track the actual number of records and bytes per partition. If any partition deviates more than 20% from the average, consider adjusting its virtual node count — adding virtual nodes to underloaded partitions and removing them from overloaded ones.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Replace modular hashing with consistent hashing using virtual nodes, enabling incremental scaling with minimal data movement.

1. **Partition assignment with consistent hashing:**
   - Hash ring size: 2^32 positions.
   - 16 partitions, each with 256 virtual nodes = 4,096 virtual nodes on the ring.
   - For each event, hash `user_id` to a ring position and assign to the partition owning the next clockwise virtual node.
   - All events for a given `user_id` land on the same partition, enabling efficient single-user lookups.

2. **Query routing:**
   - The query router maintains a copy of the ring (the sorted list of virtual node positions and their partition mappings).
   - To look up events for a `user_id`: hash the `user_id`, binary search the ring for the next clockwise virtual node, and route the query to that partition.
   - Ring metadata is small (~4,096 entries × 12 bytes ≈ 48 KB) and cached on every query router instance.

3. **Scaling from 16 to 24 partitions:**
   - Add 8 new partitions, each with 256 virtual nodes. Ring grows from 4,096 to 6,144 virtual nodes.
   - Each new partition claims 256/6,144 ≈ 4.17% of the ring.
   - Total data moved: 8 × 4.17% ≈ 33.3% of all records.
   - The moved data comes proportionally from all 16 existing partitions — each existing partition loses about 33.3% / 16 ≈ 2.08% of total data.
   - **Comparison with naive modular hashing:** Changing from `% 16` to `% 24` would move approximately 62.5% of all records — nearly twice as much data.

4. **Migration procedure:**
   - Phase 1: Add new partitions to the ring in read-only mode. Existing partitions continue serving all traffic.
   - Phase 2: For each key range claimed by a new partition, stream the affected records from the old partition to the new one. Old partitions continue serving reads for migrating keys.
   - Phase 3: Atomically switch ownership per key range — new partitions start accepting writes, old partitions stop serving migrating keys.
   - Phase 4: Old partitions delete migrated records to reclaim storage.

5. **Imbalance detection and correction:**
   - Monitor records-per-partition and bytes-per-partition every 5 minutes.
   - If a partition exceeds 120% of the average load, reduce its virtual node count by 10% (remove the least-loaded virtual nodes and reassign their arcs to neighbors).
   - If a partition falls below 80% of the average, increase its virtual node count by 10%.
   - Virtual node adjustments trigger small, targeted data migrations — far less disruptive than a full reshard.

</details>

---

## Problem 3 — Designing a Range-Based Partitioning Scheme

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a range-based partitioning strategy for a time-series data platform that must support efficient range scans over time windows while avoiding hotspot partitions that receive disproportionate write traffic. Your design should define the partition boundaries, explain how range queries are routed, and include a mechanism for splitting partitions that grow too large or too hot.

### Scenario

**Context:** An IoT monitoring platform collects sensor readings from 200,000 devices. Each reading contains a `device_id`, a `timestamp`, and a `value` payload. The system ingests 100,000 readings per second and stores them in a partitioned database. The most common query pattern is a time-range scan: "retrieve all readings for device X between time T1 and T2." The team initially partitioned by hashing `device_id`, which distributes writes evenly but makes time-range queries expensive — every query must scatter-gather across all partitions because readings for a single device are spread across the time axis within one partition, but the partition itself is selected by device hash, not time. The team wants to switch to a range-based scheme that makes time-range queries efficient while controlling the write hotspot problem that arises when all current-time writes concentrate on the latest partition.

**Requirements:** Design a range-based partitioning scheme using a composite key of `(device_id, timestamp)`. Define how partition boundaries are set, how new partitions are created as time advances, and how range queries for a single device's time window are routed to the minimum number of partitions. Address the write hotspot problem where the "current time" partition receives all new writes. Propose a partition-splitting strategy for partitions that exceed a size or throughput threshold.

**Expected Approach:** Partition by `device_id` range first (e.g., devices 0–49,999 on partition group A, 50,000–99,999 on group B, etc.), then within each device range, sub-partition by time range (e.g., 1-day buckets). This two-level scheme ensures that a time-range query for a single device touches at most a few partitions (one per day in the query window). The write hotspot is mitigated because the "current day" partition is split across 4 device-range groups, each handling only 25,000 writes/second. If a partition exceeds a threshold, it is split at the midpoint of its key range.

<details>
<summary>Hints</summary>

1. Pure time-based range partitioning (one partition per day) creates a severe write hotspot: all 100,000 writes/second hit the current day's partition. Combining device-range partitioning with time-range partitioning spreads the current-time writes across multiple partitions.
2. For a time-range query on device X between T1 and T2, the router first identifies which device-range group contains X, then scans only the time-range partitions within that group that overlap [T1, T2]. If each time bucket is 1 day and the query spans 3 days, only 3 partitions are scanned.
3. Partition splitting should be triggered by either size (e.g., partition exceeds 100 GB) or throughput (e.g., partition exceeds 30,000 writes/second). The split point is chosen at the midpoint of the partition's key range to produce two roughly equal halves.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Two-level range partitioning — first by `device_id` range, then by `timestamp` range — with automatic partition splitting for size and throughput control.

1. **Partition scheme:**
   - Level 1 (device range): Divide the 200,000 devices into 4 groups of 50,000 devices each based on `device_id` ranges: [0–49,999], [50,000–99,999], [100,000–149,999], [150,000–199,999].
   - Level 2 (time range): Within each device group, create time-range partitions in 1-day buckets. Each day, a new partition is created for each device group.
   - Total partitions per day: 4 (one per device group). Over 30 days: 120 partitions.

2. **Write distribution:**
   - At any moment, new readings are written to the 4 "current day" partitions.
   - Each current-day partition handles approximately 100,000 / 4 = 25,000 writes/second (assuming devices are evenly distributed across groups).
   - This is 4× better than a single current-time partition handling all 100,000 writes/second.

3. **Range query routing:**
   - Query: "All readings for device 73,412 between Jan 5 and Jan 7."
   - Step 1: Device 73,412 falls in group B [50,000–99,999].
   - Step 2: The query spans 3 days (Jan 5, 6, 7), so 3 partitions within group B are scanned: `B-Jan05`, `B-Jan06`, `B-Jan07`.
   - Step 3: Within each partition, a local index on `(device_id, timestamp)` efficiently locates the readings for device 73,412 in the requested time range.
   - Total partitions scanned: 3 (out of potentially hundreds). This is a dramatic improvement over scatter-gather across all partitions.

4. **Partition splitting:**
   - **Size trigger:** If a partition exceeds 100 GB, split it at the midpoint of its `device_id` range. For example, partition B-Jan05 covering devices [50,000–99,999] splits into B1-Jan05 [50,000–74,999] and B2-Jan05 [75,000–99,999].
   - **Throughput trigger:** If a current-day partition exceeds 30,000 writes/second, split it at the device-range midpoint. The two resulting partitions each handle ~15,000 writes/second.
   - The partition metadata table is updated to reflect the new boundaries, and the query router refreshes its routing map.

5. **Partition lifecycle:**
   - New partitions are pre-created at the start of each day (or lazily on first write).
   - Old partitions beyond the retention window (e.g., 90 days) are archived to cold storage and dropped from the active cluster.
   - This natural lifecycle prevents unbounded partition growth and keeps the active partition count manageable.

6. **Trade-offs vs. hash-based partitioning:**
   - **Advantage:** Time-range queries touch only the relevant day partitions within one device group — O(days in range) partitions instead of O(all partitions).
   - **Disadvantage:** Write hotspotting on current-day partitions (mitigated by device-range grouping). Uneven device activity can still cause imbalance within a device group.

</details>

---

## Problem 4 — Composite Partitioning for a Multi-Tenant SaaS Platform

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design a composite partitioning strategy for a multi-tenant SaaS platform that must isolate tenant data for security and performance, support tenants of vastly different sizes, and allow the system to scale by adding partitions without downtime. Your strategy should combine multiple partitioning techniques — hash, range, or list — into a multi-level scheme that balances data isolation, query efficiency, and operational simplicity.

### Scenario

**Context:** A project management SaaS platform serves 5,000 tenants ranging from small teams (10 users, 1,000 tasks) to enterprise customers (50,000 users, 20 million tasks). The total dataset is 2 billion task records across all tenants. The system runs on 32 database partitions. The current scheme hashes `tenant_id` to assign each tenant to a single partition. This works well for small tenants, but the 10 largest enterprise tenants each have 20+ million tasks, and several of them land on the same partition by chance, creating severe hotspots. Meanwhile, some partitions hold only small tenants and are nearly idle. The team needs a partitioning strategy that (a) keeps small tenants on a single partition for simple queries, (b) spreads large tenants across multiple partitions for throughput, and (c) provides tenant-level data isolation so that one tenant's heavy workload does not degrade another tenant's performance.

**Requirements:** Design a two-level composite partitioning scheme. The first level assigns tenants to partition groups using a tenant-size classification (small, medium, large). The second level partitions data within each group using a strategy appropriate to the tenant size. Define the partition key, the classification thresholds, and the routing logic. Explain how a tenant is migrated from the "small" tier to the "large" tier as it grows. Address cross-tenant query isolation — how the system prevents a large tenant's full-table scan from starving small tenants' queries on the same physical hardware.

**Expected Approach:** Classify tenants into three tiers: small (< 100K tasks), medium (100K–5M tasks), large (> 5M tasks). Small tenants are hash-partitioned by `tenant_id` across a shared pool of 16 partitions. Medium tenants each get a dedicated partition. Large tenants are hash-partitioned by `task_id` across 4 dedicated partitions per tenant. The routing layer maintains a tenant-to-partition mapping that is updated when a tenant crosses a tier threshold. Resource isolation is enforced via per-tenant query quotas and separate connection pools for each tier.

<details>
<summary>Hints</summary>

1. The key insight is that one partitioning strategy does not fit all tenant sizes. Small tenants benefit from co-location (fewer partitions to manage, simpler routing), while large tenants need their data spread across multiple partitions to avoid hotspots.
2. Tenant tier migration (e.g., small → medium) requires moving all of a tenant's data from the shared partition pool to a dedicated partition. This can be done as a background migration with double-write during the transition period — new writes go to both the old and new locations, reads prefer the new location, and once migration is complete, the old data is deleted.
3. For query isolation, consider that a large tenant's analytical query (scanning millions of rows) should not compete for I/O and CPU with small tenants' transactional queries. Physical separation (different partition groups on different hardware) is the strongest isolation, but per-tenant resource quotas (query timeout limits, connection pool limits, I/O bandwidth caps) provide isolation even when tenants share hardware.
4. The routing layer needs a tenant metadata table: `{tenant_id → tier, partition_list}`. This table is small (5,000 entries), cached on every application server, and updated infrequently (only when tenants cross tier thresholds).

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Three-tier composite partitioning with hash-based co-location for small tenants, dedicated partitions for medium tenants, and multi-partition hash sharding for large tenants.

1. **Tenant classification:**
   - Small: < 100,000 tasks. Approximately 4,900 tenants. Total data: ~200 million tasks.
   - Medium: 100,000–5,000,000 tasks. Approximately 90 tenants. Total data: ~800 million tasks.
   - Large: > 5,000,000 tasks. Approximately 10 tenants. Total data: ~1 billion tasks.

2. **Partition allocation (32 partitions total):**
   - **Small tier (shared pool):** 8 partitions. 4,900 small tenants are hash-partitioned by `tenant_id` across these 8 partitions. Each partition holds ~600 tenants and ~25 million tasks. Queries for a small tenant always hit exactly one partition.
   - **Medium tier (dedicated):** 14 partitions. Each of the ~90 medium tenants gets a dedicated partition (some partitions host 6–7 medium tenants each, grouped by hash to balance load). Each partition holds ~57 million tasks.
   - **Large tier (multi-partition):** 10 partitions. Each of the 10 large tenants is hash-partitioned by `task_id` across a set of dedicated partitions. With 10 partitions for 10 tenants, each large tenant gets 1 partition initially; as tenants grow, they can expand to 2–4 partitions by splitting.

3. **Routing logic:**
   - The application maintains a tenant metadata cache: `{tenant_id → (tier, partition_ids)}`.
   - For a small tenant query: hash `tenant_id` → single partition in the shared pool.
   - For a medium tenant query: look up the dedicated partition ID.
   - For a large tenant query by `task_id`: hash `task_id` → one of the tenant's dedicated partitions. For queries that scan all tasks (e.g., reporting), scatter-gather across the tenant's partition set.

4. **Tier migration (small → medium example):**
   - Trigger: A tenant's task count crosses 100,000.
   - Step 1: Allocate a dedicated partition for the tenant (or assign it to an existing medium-tier partition with capacity).
   - Step 2: Begin background migration — copy all of the tenant's tasks from the shared pool partition to the new dedicated partition. During migration, new writes go to both locations (double-write).
   - Step 3: Once migration is complete, update the tenant metadata cache to point to the new partition. Reads now go to the dedicated partition only.
   - Step 4: Delete the tenant's data from the shared pool partition.
   - Downtime: zero. The double-write period ensures no data is lost, and the metadata switch is atomic.

5. **Cross-tenant query isolation:**
   - **Physical separation:** Small-tier and large-tier partitions run on different hardware. A large tenant's full-table scan cannot consume I/O on small-tier machines.
   - **Per-tenant resource quotas:** Each tenant is assigned a maximum concurrent query count (e.g., small: 5, medium: 20, large: 50) and a per-query timeout (e.g., small: 5s, medium: 30s, large: 120s). Queries exceeding the quota are queued or rejected.
   - **Connection pool separation:** Each tier has its own database connection pool. A surge in large-tenant queries cannot exhaust connections available to small tenants.

6. **Scaling the partition count:**
   - To add partitions, assign new partitions to the tier that needs capacity. For example, if the small tier is overloaded, add 2 partitions to the shared pool and rebalance tenants using consistent hashing (only ~20% of small tenants need to move).
   - For a large tenant that outgrows its single partition, split it into 2 partitions by hash-partitioning its `task_id` space. Only that tenant's data moves; other tenants are unaffected.

</details>

---

## Problem 5 — Handling Cross-Partition Queries and Distributed Joins

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a query execution strategy for a partitioned database system that must support queries spanning multiple partitions, including cross-partition joins, aggregations, and transactions. Your design should minimize the performance overhead of cross-partition operations while preserving correctness, and it should define clear guidelines for when cross-partition queries are acceptable versus when the data model should be restructured to avoid them.

### Scenario

**Context:** A financial services platform stores account data partitioned by `account_id` across 20 database partitions using hash-based partitioning. The system handles 200,000 single-account queries per second efficiently — each query hits exactly one partition. However, several critical business operations require cross-partition data access. (1) A compliance report must join the `accounts` table with the `transactions` table to list all transactions above $10,000 across all accounts — this requires a full scatter-gather across all 20 partitions. (2) A money transfer between two accounts (debit one, credit another) often involves two different partitions and requires a distributed transaction to maintain consistency. (3) A dashboard query computes the total balance across all accounts, requiring an aggregation across all 20 partitions. These cross-partition operations currently take 5–10× longer than single-partition queries and occasionally cause timeouts. The team needs a strategy to make cross-partition operations efficient and reliable without abandoning the partitioning scheme.

**Requirements:** Design solutions for three cross-partition operation types: scatter-gather queries (the compliance report), distributed transactions (the money transfer), and global aggregations (the total balance dashboard). For each, describe the execution plan, the consistency guarantees, and the performance characteristics. Propose data model changes that could eliminate or reduce the need for the most expensive cross-partition operations. Define a decision framework for when to accept cross-partition query overhead versus when to denormalize or restructure the data model.

**Expected Approach:** For scatter-gather queries, use a coordinator node that fans out the query to all partitions in parallel, collects partial results, and merges them. For distributed transactions, use a two-phase commit protocol with a transaction coordinator, and consider whether eventual consistency (saga pattern) is acceptable for money transfers. For global aggregations, maintain a materialized aggregate table that is updated asynchronously. For data model restructuring, co-locate related data on the same partition (e.g., partition transactions by `account_id` so that account + transaction joins are local) and use denormalized summary tables for cross-partition reporting.

<details>
<summary>Hints</summary>

1. Scatter-gather queries scale linearly with the number of partitions — querying 20 partitions in parallel takes roughly the same wall-clock time as querying 1 partition (assuming sufficient network bandwidth and coordinator capacity), but consumes 20× the total resources. The key optimization is pushing filters and projections down to each partition so that only matching rows are returned to the coordinator.
2. Two-phase commit (2PC) guarantees atomicity across partitions but introduces a blocking risk: if the coordinator crashes after sending "prepare" but before sending "commit," all participating partitions hold locks until the coordinator recovers. The saga pattern avoids this by using compensating transactions (e.g., if the credit fails, issue a compensating debit) but provides only eventual consistency.
3. Materialized aggregate tables trade write-time overhead for read-time efficiency. Every time an account balance changes, the system updates a global `total_balance` counter. This makes the dashboard query a single-row read instead of a 20-partition scatter-gather, but adds complexity to the write path.
4. The decision framework should consider query frequency, latency requirements, and consistency needs. A compliance report that runs once daily can tolerate scatter-gather. A money transfer that happens 1,000 times per second needs an efficient distributed transaction or saga. A dashboard that refreshes every 5 seconds benefits from a materialized aggregate.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Three targeted strategies for three cross-partition operation types, plus a data model restructuring plan and a decision framework.

1. **Scatter-gather for the compliance report:**
   - A coordinator node receives the compliance query: "SELECT account_id, transaction_id, amount FROM transactions WHERE amount > 10000."
   - The coordinator fans out the query to all 20 partitions in parallel. Each partition executes the query locally, applying the `amount > 10000` filter, and returns only matching rows.
   - The coordinator collects partial results from all 20 partitions and merges them into a single result set (simple concatenation, since no cross-partition join is needed — the `transactions` table is partitioned by `account_id`, and each transaction belongs to one account).
   - **Performance:** Wall-clock time ≈ slowest partition response time + coordinator merge time. If each partition responds in 50 ms and returns 500 rows, the coordinator merges 10,000 rows in < 10 ms. Total: ~60 ms. This is acceptable for a daily compliance report.
   - **Optimization:** If the compliance report also needs account metadata (name, type), co-locate the `accounts` and `transactions` tables on the same partition key (`account_id`). This makes the join local to each partition — the coordinator receives pre-joined results and only needs to concatenate.

2. **Distributed transactions for money transfers:**
   - **Option A — Two-phase commit (2PC):**
     - A transaction coordinator initiates the transfer: debit $500 from account A (partition 7), credit $500 to account B (partition 13).
     - Phase 1 (Prepare): Coordinator sends "prepare" to partitions 7 and 13. Each partition acquires a row lock on the respective account, validates the operation (e.g., sufficient balance), and responds "ready."
     - Phase 2 (Commit): Coordinator sends "commit" to both partitions. Each partition applies the change and releases the lock.
     - Latency: ~10–20 ms (two network round-trips). Suitable for the 1,000 transfers/second workload.
     - Risk: If the coordinator crashes between prepare and commit, partitions hold locks until recovery. Mitigate with a persistent transaction log on the coordinator and a timeout-based abort on partitions (e.g., abort if no commit received within 5 seconds).
   - **Option B — Saga pattern (eventual consistency):**
     - Step 1: Debit $500 from account A (local transaction on partition 7). Record a "pending transfer" event.
     - Step 2: Credit $500 to account B (local transaction on partition 13). Record a "transfer complete" event.
     - If Step 2 fails: Execute a compensating transaction — credit $500 back to account A (reversal on partition 7).
     - Latency: ~5–10 ms per step (single-partition transactions). Total: ~10–20 ms.
     - Trade-off: Between Step 1 and Step 2, the system is in an inconsistent state (money has been debited but not yet credited). This window is typically < 100 ms and is acceptable if the application can tolerate brief inconsistency.
   - **Recommendation:** Use 2PC for money transfers where atomicity is critical. Use sagas for less critical cross-partition operations (e.g., updating a user's profile across partitions).

3. **Global aggregation for the total balance dashboard:**
   - Maintain a materialized `global_balances` table on a dedicated aggregation node. Schema: `{metric_name, value, last_updated}`.
   - Every time an account balance changes (deposit, withdrawal, transfer), the partition publishes a balance-change event to a message queue.
   - An aggregation worker consumes these events and updates the `global_balances` table: `total_balance += delta`.
   - The dashboard reads `total_balance` from the aggregation node — a single-row read taking < 1 ms.
   - **Staleness:** The materialized value lags behind real-time by the event processing delay (typically 1–5 seconds). For a dashboard that refreshes every 5 seconds, this is acceptable.
   - **Consistency check:** Periodically (e.g., hourly), run a full scatter-gather aggregation across all 20 partitions and compare with the materialized value. If they diverge, recalculate the materialized value from scratch.

4. **Data model restructuring recommendations:**
   - **Co-locate related tables:** Partition `accounts`, `transactions`, and `balances` all by `account_id`. This ensures that single-account queries (which represent 99% of traffic) never cross partitions, and account-transaction joins are always local.
   - **Denormalize for reporting:** Create a `daily_transaction_summary` table (partitioned by date) that stores pre-aggregated data: total transaction count, total amount, and count of high-value transactions per account per day. The compliance report can query this summary table instead of scanning raw transactions.
   - **Avoid cross-partition foreign keys:** Do not create foreign key relationships between tables on different partition keys. Instead, enforce referential integrity at the application layer.

5. **Decision framework — when to accept cross-partition overhead:**

   | Factor | Accept Cross-Partition | Restructure Data Model |
   |--------|----------------------|----------------------|
   | Query frequency | Low (daily, weekly) | High (per-second) |
   | Latency requirement | Relaxed (seconds OK) | Strict (< 50 ms) |
   | Consistency need | Eventual OK | Strong consistency required |
   | Data volume | Small result sets | Large scans or joins |
   | Development cost | Low (query works as-is) | High (schema changes, migration) |

   Apply this framework: the daily compliance report (low frequency, relaxed latency) is fine as a scatter-gather. The money transfer (high frequency, strong consistency) justifies 2PC or co-location. The dashboard (high frequency, relaxed consistency) benefits from a materialized aggregate.

</details>
