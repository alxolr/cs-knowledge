# Replication Strategies

**Track:** Design Concepts
**Difficulty Tier:** Intermediate
**Prerequisites:** [Database Schema Design](../database-schema-design/README.md), [CAP Theorem](../cap-theorem/README.md)

## Concept Overview

Replication is the practice of maintaining copies of the same data on multiple machines connected via a network. By keeping identical data on several nodes, a system gains fault tolerance (the service continues operating when a node fails), reduced latency (clients can read from a geographically nearby replica), and increased read throughput (multiple replicas can serve read queries in parallel). Every distributed database, message broker, and large-scale storage system relies on some form of replication, making it one of the most fundamental building blocks of distributed systems design.

The central challenge of replication is keeping all copies consistent as the data changes. If data never changed after being written, replication would be trivial — just copy the bytes once. The difficulty arises entirely from handling writes. Different replication strategies make different trade-offs between consistency, availability, latency, and operational complexity. Single-leader replication funnels all writes through one node, providing strong ordering guarantees but creating a single point of failure for writes. Multi-leader replication allows writes at multiple nodes, improving write availability and geographic latency but introducing the thorny problem of write conflicts. Leaderless replication eliminates the leader concept entirely, using quorum-based reads and writes to achieve both availability and tunable consistency.

Orthogonal to the leader topology is the question of synchronous versus asynchronous replication. Synchronous replication guarantees that a write is durable on multiple replicas before acknowledging it to the client, providing stronger durability at the cost of higher write latency and reduced availability (a single slow replica can stall all writes). Asynchronous replication acknowledges writes immediately after the leader persists them, offering lower latency and higher availability but risking data loss if the leader fails before replicas catch up. Most production systems use a hybrid approach — one synchronous follower for durability and the rest asynchronous for performance.

Understanding replication strategies is essential for designing systems that must remain available and consistent under failure. The choice of replication topology, synchronization mode, and conflict resolution mechanism directly shapes a system's behavior during network partitions, node failures, and geographic distribution — the exact scenarios where design decisions matter most.

---

## Problem 1 — Single-Leader Replication for a Financial Ledger

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a single-leader replication architecture for a financial ledger service that must guarantee strict ordering of all transactions while providing read scalability across multiple replicas. Your design should define the write path, the replication mechanism from leader to followers, and the strategy for handling leader failure.

### Scenario

**Context:** A fintech company operates a ledger service that records all monetary transactions — deposits, withdrawals, and transfers — for 2 million customer accounts. The service handles 5,000 write transactions per second and 50,000 read queries per second. Every transaction must be applied in a globally consistent order because account balances depend on the exact sequence of debits and credits. The current single-node database handles the write load comfortably but cannot keep up with the read traffic, causing query latency to spike during peak hours. The team wants to add read replicas to offload read traffic from the primary database without compromising the strict ordering guarantee that the financial domain requires.

**Requirements:** All writes must flow through a single leader node that assigns a monotonically increasing sequence number to each transaction, guaranteeing a total order. Read replicas must receive transactions in the same order they were applied on the leader. The system must detect leader failure within 10 seconds and promote a follower to become the new leader. Clients reading from followers must be aware that they may see slightly stale data (replication lag) and must have a mechanism to request a strongly consistent read when needed.

**Expected Approach:** Use a single-leader (primary-secondary) replication topology where the leader writes to a write-ahead log (WAL) and streams log entries to followers in order. Followers apply entries sequentially to maintain the same transaction order. A failover mechanism based on heartbeat monitoring promotes the most up-to-date follower when the leader becomes unreachable. Strongly consistent reads are served by routing them to the leader; eventually consistent reads go to any follower.

<details>
<summary>Hints</summary>

1. The write-ahead log (WAL) is the natural replication unit — the leader already writes every transaction to the WAL for crash recovery. Streaming WAL entries to followers ensures they see the exact same sequence of operations in the exact same order.
2. For failover, each follower tracks its replication position (the last WAL sequence number it has applied). The follower with the highest position has the least data loss and is the best promotion candidate.
3. For strongly consistent reads, the simplest approach is to route them to the leader. A more sophisticated approach is "read-your-writes" consistency: after a client performs a write, it remembers the WAL position of that write and, when reading from a follower, waits until the follower has caught up to at least that position.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Single-leader replication with WAL streaming, heartbeat-based failover, and a read consistency protocol.

1. **Write path:**
   - All write transactions are sent to the leader node.
   - The leader assigns a monotonically increasing log sequence number (LSN) to each transaction.
   - The leader writes the transaction to its local WAL and applies it to the database.
   - The leader acknowledges the write to the client after local persistence (asynchronous replication to followers).

2. **Replication to followers:**
   - The leader continuously streams WAL entries to each follower over a persistent TCP connection.
   - Each follower receives entries in LSN order and applies them sequentially to its local database.
   - Each follower tracks its current applied LSN and periodically reports it to the leader (used for monitoring replication lag).

3. **Read path:**
   - Eventually consistent reads: routed to any follower via a load balancer. The client accepts that data may be up to a few seconds stale (replication lag).
   - Strongly consistent reads: routed to the leader, which always has the latest data.
   - Read-your-writes consistency: after a write, the client receives the write's LSN. On subsequent reads, the client includes this LSN in the request. The follower either serves the read immediately (if its applied LSN ≥ the requested LSN) or waits briefly for replication to catch up.

4. **Read scalability:**
   - With 3 followers, each handles approximately 50,000 / 3 ≈ 16,700 reads/second.
   - The leader is freed from read traffic and dedicates its resources to the 5,000 writes/second.

5. **Leader failure detection and failover:**
   - Each follower sends a heartbeat to the leader every 2 seconds. If a follower receives no response for 3 consecutive heartbeats (6 seconds), it suspects leader failure.
   - A consensus mechanism among followers (or an external coordinator like ZooKeeper) confirms the failure and selects the follower with the highest applied LSN as the new leader.
   - The new leader begins accepting writes and streaming WAL entries to the remaining followers.
   - Clients are redirected to the new leader via DNS update or a proxy layer.
   - **Data loss risk:** If the old leader had accepted writes that were not yet replicated to any follower, those writes are lost. This is the inherent trade-off of asynchronous replication — the window of potential data loss equals the replication lag at the moment of failure.

</details>

---

## Problem 2 — Multi-Leader Replication for a Collaborative Document Editor

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Design a multi-leader replication architecture for a collaborative document editing platform where users in different geographic regions can edit documents simultaneously with low latency. Your design should define how writes are accepted at multiple leaders, how changes are propagated between leaders, and how the system detects write conflicts that arise when two users edit the same section of a document concurrently.

### Scenario

**Context:** A global collaboration platform allows teams to edit shared documents in real time. The platform operates data centers in three regions: US-East, EU-West, and AP-Southeast. With single-leader replication, all writes flow to the US-East leader, causing 200 ms round-trip latency for European users and 350 ms for Asian users — unacceptable for a real-time editing experience. The team wants each region to have its own leader that accepts writes locally, reducing write latency to under 20 ms for all users. Documents are edited by teams that are often distributed across regions, so the same document may receive concurrent edits from multiple leaders.

**Requirements:** Each regional data center must have a leader that accepts writes for any document. Changes made at one leader must be asynchronously replicated to all other leaders within 2 seconds under normal network conditions. The system must detect when two leaders have concurrently modified the same document section (a write conflict) and flag these conflicts for resolution rather than silently dropping one edit. The replication topology must tolerate the temporary failure of one data center without blocking writes at the remaining two.

**Expected Approach:** Deploy a multi-leader (active-active) topology with three leaders, one per region. Each leader accepts writes locally and asynchronously replicates changes to the other two leaders using a circular or all-to-all topology. Each write is tagged with a globally unique timestamp and the originating leader ID. Conflict detection compares incoming replicated writes against local writes — if two writes modify the same document section with overlapping timestamps and different leader origins, a conflict is flagged.

<details>
<summary>Hints</summary>

1. In a multi-leader setup, each leader independently accepts writes and then propagates them to the other leaders. The replication is asynchronous — each leader does not wait for the others to acknowledge before confirming the write to the client. This is what gives each region low write latency.
2. Conflict detection requires tracking the causal history of each write. A simple approach is to attach a vector clock or a Lamport timestamp plus leader ID to each write. When a leader receives a replicated write, it checks whether the write's causal predecessor matches the local state — if not, the writes are concurrent and a conflict exists.
3. An all-to-all replication topology (each leader sends to every other leader) is simpler and more fault-tolerant than a circular topology. In a circle, if one leader fails, the chain is broken. In all-to-all, each leader communicates directly with every other leader, so one failure does not block replication between the remaining two.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Three-leader active-active replication with all-to-all topology, vector clock–based conflict detection, and asynchronous propagation.

1. **Replication topology:**
   - Three leaders: L-US (US-East), L-EU (EU-West), L-AP (AP-Southeast).
   - All-to-all topology: each leader maintains a replication stream to every other leader. L-US sends changes to both L-EU and L-AP; L-EU sends to L-US and L-AP; L-AP sends to L-US and L-EU.
   - Each replication stream is an asynchronous, ordered log of write operations.

2. **Write path (local):**
   - A user in EU-West edits a document. The write is sent to L-EU.
   - L-EU assigns the write a unique identifier: `(leader_id=EU, sequence_number=N, vector_clock={US:5, EU:12, AP:8})`.
   - L-EU applies the write to its local database and acknowledges the client (latency < 20 ms).
   - L-EU enqueues the write for replication to L-US and L-AP.

3. **Replication propagation:**
   - Each leader continuously streams its write log to the other two leaders.
   - Under normal conditions, a write reaches the other leaders within 1–2 seconds (network latency between regions plus batching delay).
   - Each receiving leader applies the replicated write to its local database, advancing its vector clock for the originating leader's component.

4. **Conflict detection:**
   - When L-US receives a replicated write from L-EU for document D, section S, it checks whether L-US has any local writes to the same (D, S) that are concurrent — i.e., writes that L-EU had not seen when it made its write (determined by comparing vector clocks).
   - Two writes W1 (from L-US) and W2 (from L-EU) are concurrent if neither's vector clock dominates the other's: W1's clock does not fully precede W2's, and W2's clock does not fully precede W1's.
   - If concurrent writes to the same document section are detected, the system flags a conflict and stores both versions.

5. **Conflict storage and surfacing:**
   - Conflicting writes are stored as sibling versions in the document's history.
   - The application layer presents the conflict to the user (e.g., "Two edits were made to paragraph 3 — choose which to keep or merge them").
   - Until resolved, the system can display one version as the default (e.g., the one with the higher timestamp) while preserving the other.

6. **Failure tolerance:**
   - If L-AP fails, L-US and L-EU continue accepting writes and replicating to each other. Writes destined for L-AP are queued.
   - When L-AP recovers, it catches up by consuming the queued writes from L-US and L-EU.
   - No writes are blocked during the outage — the remaining two leaders operate independently.

</details>

---

## Problem 3 — Leaderless Replication for a Shopping Cart Service

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a leaderless replication architecture for a high-availability shopping cart service that must remain writable even during node failures and network partitions. Your design should define the quorum parameters, explain how reads and writes are distributed across replicas, and describe the anti-entropy mechanisms that repair inconsistencies between replicas.

### Scenario

**Context:** An e-commerce platform serves 10 million active users during peak sales events. The shopping cart service must be available for both reads and writes at all times — if a user cannot add items to their cart during a flash sale, the company loses revenue. The service stores cart data across 5 replica nodes. The team previously used single-leader replication, but during a recent flash sale the leader node became overloaded and failed, making all carts read-only for 45 seconds until failover completed. During those 45 seconds, the platform lost an estimated $200,000 in abandoned carts. The team wants a replication strategy that has no single point of failure for writes — every node should be able to accept writes independently, and the system should tolerate up to 2 node failures without any loss of availability.

**Requirements:** The system must use a leaderless (Dynamo-style) replication model with 5 replicas. Writes must be sent to multiple replicas in parallel and considered successful when a write quorum is reached. Reads must query multiple replicas and use a read quorum to return the most recent value. The quorum parameters must be chosen so that the system tolerates 2 node failures while guaranteeing that every read sees the most recent write. The design must include a mechanism for repairing replicas that have stale data — both during reads (read repair) and in the background (anti-entropy).

**Expected Approach:** With N=5 replicas, choose W=3 (write quorum) and R=3 (read quorum) so that W + R > N (3 + 3 = 6 > 5), guaranteeing overlap between the write set and the read set. This means every read contacts at least one node that has the latest write. The system tolerates 2 failures because 3 of 5 nodes are sufficient for both reads and writes. Read repair updates stale replicas discovered during reads. A background anti-entropy process (using Merkle trees) periodically compares replicas and synchronizes differences.

<details>
<summary>Hints</summary>

1. The quorum condition W + R > N ensures that the set of nodes that acknowledged a write and the set of nodes queried during a read always overlap by at least one node. That overlapping node has the latest value, so the read can return it.
2. With N=5, W=3, R=3: writes succeed as long as 3 of 5 nodes are reachable (tolerates 2 failures). Reads also succeed as long as 3 of 5 nodes are reachable. The system remains fully available for both reads and writes with up to 2 node failures.
3. Read repair works opportunistically: when a read query contacts 3 replicas and discovers that one of them has a stale version, the coordinator sends the latest version to the stale replica immediately. This repairs inconsistencies on the read path without any background process.
4. Anti-entropy using Merkle trees works by each replica maintaining a hash tree over its data. Two replicas compare their root hashes — if they differ, they recursively compare subtrees to identify the specific keys that are out of sync, then exchange only the differing values. This is efficient even for large datasets.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Leaderless replication with N=5, W=3, R=3 quorums, read repair, and Merkle tree–based anti-entropy.

1. **Quorum parameters:**
   - N = 5 (total replicas for each cart).
   - W = 3 (write quorum — a write succeeds when 3 of 5 replicas acknowledge it).
   - R = 3 (read quorum — a read queries 3 of 5 replicas and returns the most recent value).
   - Overlap guarantee: W + R = 6 > 5 = N, so at least one node in every read set has the latest write.
   - Fault tolerance: the system tolerates up to 2 node failures for both reads and writes (3 of 5 nodes are sufficient).

2. **Write path:**
   - The client sends an "add item to cart" request to a coordinator node (any node can be the coordinator).
   - The coordinator forwards the write to all 5 replicas in parallel.
   - Each replica that receives the write persists it locally and responds with an acknowledgment.
   - The coordinator waits for 3 acknowledgments (W=3), then responds to the client with success.
   - If fewer than 3 replicas respond within a timeout (e.g., 500 ms), the write fails and the client retries.
   - Each write is tagged with a vector clock or a timestamp to establish a causal ordering.

3. **Read path:**
   - The client sends a "get cart" request to a coordinator node.
   - The coordinator queries 3 of the 5 replicas in parallel.
   - Each replica returns its local version of the cart along with the version's timestamp or vector clock.
   - The coordinator compares the 3 responses and returns the one with the highest version to the client.
   - If the 3 responses disagree (some replicas have stale data), the coordinator triggers read repair.

4. **Read repair:**
   - During the read above, if replica R2 returned version V5 while replicas R1 and R3 returned the newer version V7, the coordinator sends V7 to R2 in the background.
   - R2 updates its local copy to V7, repairing the inconsistency.
   - Read repair is opportunistic — it only fixes replicas that are contacted during reads. Replicas that are rarely read may remain stale until anti-entropy runs.

5. **Anti-entropy (background repair):**
   - Each replica maintains a Merkle tree over its cart data, keyed by `cart_id`.
   - Every 5 minutes, each pair of replicas compares their Merkle tree root hashes.
   - If the roots differ, they recursively compare subtrees to identify the specific `cart_id` values that are out of sync.
   - The replica with the older version fetches the newer version from its peer.
   - Merkle tree comparison is efficient: for 10 million carts, the tree has ~24 levels, so identifying all differences requires at most 24 round trips of hash comparisons, regardless of how many carts are out of sync.

6. **Handling concurrent writes (cart conflicts):**
   - Two users sharing a cart (e.g., a family account) may add items concurrently to different replicas.
   - If the writes are concurrent (neither causally depends on the other, as determined by vector clocks), the system stores both versions as siblings.
   - On the next read, the application merges the siblings by taking the union of items in both cart versions — this is a safe merge strategy for shopping carts because adding an item is a commutative operation.

</details>

---

## Problem 4 — Synchronous vs. Asynchronous Replication for a Healthcare Records System

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design a replication strategy for a healthcare records system that must balance strict durability requirements (no patient data can be lost) with performance requirements (doctors must be able to update records without perceptible delay). Your design should define which replicas receive synchronous writes, which receive asynchronous writes, and how the system adapts when a synchronous replica becomes slow or unreachable.

### Scenario

**Context:** A hospital network operates an electronic health records (EHR) system across two data centers — a primary site at the main hospital and a disaster recovery (DR) site 50 km away. The system stores 5 million patient records and handles 2,000 write operations per second (medication orders, lab results, clinical notes). Regulatory requirements mandate that no committed patient record update can be lost, even if the primary data center is completely destroyed (e.g., by a natural disaster). This means every committed write must be durably stored at the DR site before the system acknowledges it to the doctor. However, the network round trip between the two sites is 4 ms, and the current fully synchronous replication adds this latency to every write, causing the P99 write latency to spike to 50 ms when the DR site experiences minor network congestion. Doctors have complained that the system feels sluggish during peak hours. The team needs a replication strategy that guarantees zero data loss for committed writes while minimizing the latency impact on the write path.

**Requirements:** The system must guarantee that every acknowledged write is durable on at least two nodes — one at the primary site and one at the DR site — before the acknowledgment is sent to the client (zero RPO — recovery point objective). The P99 write latency must remain below 30 ms under normal conditions. The system must handle the scenario where the DR site becomes temporarily unreachable (network partition) without blocking writes at the primary site indefinitely — but it must clearly communicate to operators that the zero-data-loss guarantee is temporarily suspended. The design must include a mechanism for the DR site to catch up after a partition heals, and it must address the "split-brain" risk if both sites believe they are the primary.

**Expected Approach:** Use semi-synchronous replication: one replica at the DR site receives synchronous writes (the write is not acknowledged until the DR replica confirms), while additional replicas at both sites receive asynchronous writes. If the synchronous DR replica becomes slow or unreachable, the system promotes an asynchronous replica at the primary site to synchronous status temporarily, maintaining the "two durable copies" guarantee locally. The system raises an alert that cross-site durability is degraded. When the DR site recovers, it catches up from the replication log and resumes synchronous replication. Split-brain prevention uses a fencing mechanism with a shared quorum service.

<details>
<summary>Hints</summary>

1. Fully synchronous replication to the DR site guarantees zero data loss but ties write latency to the inter-site network. Semi-synchronous replication keeps one synchronous replica for durability while allowing the rest to be asynchronous, limiting the latency impact to a single synchronous round trip.
2. When the synchronous DR replica becomes unreachable, the system faces a choice: block all writes (preserving the zero-data-loss guarantee but sacrificing availability) or switch to a local synchronous replica (preserving availability but temporarily losing cross-site durability). The right choice depends on the regulatory context — in healthcare, a brief period of local-only durability with an operator alert is usually preferable to blocking all writes and potentially delaying patient care.
3. Split-brain occurs when a network partition causes both sites to believe they are the primary and both accept writes independently. Preventing this requires a tie-breaking mechanism — typically a quorum-based lease or a third-party witness node that grants the "primary" role to exactly one site at a time.
4. After the DR site recovers from a partition, it must replay all writes that occurred during the partition. The replication log at the primary site serves as the source of truth — the DR site requests all log entries after its last applied position and applies them sequentially.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Semi-synchronous replication with adaptive synchronous replica selection, operator alerting for degraded durability, and quorum-based split-brain prevention.

1. **Replica topology:**
   - Primary site: Leader node (L1) + one asynchronous follower (F1-local).
   - DR site: One synchronous follower (F2-DR) + one asynchronous follower (F3-DR).
   - Total: 4 replicas across 2 sites.

2. **Normal operation (write path):**
   - A doctor submits a record update to L1 (the leader at the primary site).
   - L1 writes the update to its local WAL.
   - L1 sends the update to F2-DR (synchronous) and to F1-local and F3-DR (asynchronous) in parallel.
   - L1 waits for F2-DR to acknowledge the write (confirming it is durable at the DR site).
   - Once both L1's local WAL and F2-DR's acknowledgment are received, L1 acknowledges the write to the doctor.
   - Write latency: local WAL write (~1 ms) + synchronous round trip to DR (~4 ms) + F2-DR disk write (~2 ms) = ~7 ms typical. P99 stays below 30 ms because only one synchronous replica is involved.

3. **Degraded mode (DR site unreachable):**
   - If F2-DR does not acknowledge within a timeout (e.g., 500 ms), L1 declares F2-DR unreachable.
   - L1 promotes F1-local (the asynchronous follower at the primary site) to synchronous status.
   - Writes now require acknowledgment from both L1 and F1-local before being acknowledged to the client. This maintains the "two durable copies" guarantee, but both copies are at the primary site.
   - The system raises a critical alert: "Cross-site durability degraded — writes are not replicated to the DR site. Zero-RPO guarantee is suspended for cross-site disaster scenarios."
   - Writes continue without blocking — doctors are not impacted.

4. **DR site recovery:**
   - When the network partition heals, F2-DR reconnects to L1.
   - F2-DR reports its last applied WAL position (e.g., LSN 1,000,000).
   - L1 streams all WAL entries from LSN 1,000,001 onward to F2-DR.
   - F2-DR applies the entries sequentially until it catches up to L1's current position.
   - Once F2-DR is within an acceptable lag threshold (e.g., < 100 ms behind), L1 promotes F2-DR back to synchronous status and demotes F1-local back to asynchronous.
   - The alert is cleared: "Cross-site durability restored."

5. **Split-brain prevention:**
   - A quorum service (e.g., a 3-node ZooKeeper or etcd cluster, with 2 nodes at the primary site and 1 at the DR site) manages a "primary lease."
   - Only the site holding the lease can accept writes. The lease has a TTL (e.g., 10 seconds) and must be renewed periodically.
   - If the primary site loses connectivity to the quorum service (cannot renew the lease), it stops accepting writes after the lease expires — even if it is otherwise healthy. This prevents both sites from accepting writes simultaneously.
   - If the DR site needs to take over (e.g., the primary site is destroyed), it acquires the lease from the quorum service (possible because 1 of 3 quorum nodes is at the DR site, and the 2 primary-site nodes are unreachable — but this requires manual intervention or a pre-configured policy, since 1 of 3 nodes cannot form a majority alone).
   - For true automatic DR failover, the quorum service should have nodes in a third location (e.g., a cloud region), giving the DR site the ability to form a majority (2 of 3) even when the primary site is down.

6. **Durability guarantees summary:**
   | Mode | Acknowledged write durable at | Zero RPO (cross-site)? |
   |------|-------------------------------|------------------------|
   | Normal | Primary + DR site | Yes |
   | Degraded | Primary (2 local copies) | No (local disaster risk) |
   | DR failover | DR site (new primary) | Yes (after catch-up) |

</details>

---

## Problem 5 — Conflict Resolution in a Multi-Leader Inventory System

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a conflict resolution strategy for a multi-leader inventory management system where warehouses in different regions can independently accept stock updates and sales orders. Your design must handle the case where two warehouses concurrently sell the last unit of the same product, define the conflict resolution policy, and ensure that the system converges to a consistent state across all leaders without violating business invariants (e.g., selling more units than physically exist).

### Scenario

**Context:** A global retail company operates 3 regional warehouses — Americas, Europe, and Asia-Pacific — each with its own database leader. Each warehouse independently accepts sales orders and stock replenishment updates for the products it carries. Many popular products are stocked at all three warehouses. The system uses multi-leader replication so that each warehouse can process orders with low latency even if the network between regions is temporarily unavailable. A critical problem has emerged: during a recent product launch, all three warehouses had 10 units of a limited-edition item. Within a 2-second window, the Americas warehouse sold 8 units, the Europe warehouse sold 6 units, and the Asia-Pacific warehouse sold 4 units — a total of 18 units sold against only 10 in stock. Because each warehouse made its sales decisions based on its local view of inventory (which had not yet received the other warehouses' sales), the system oversold by 8 units. The company had to cancel 8 orders, damaging customer trust. The team needs a conflict resolution strategy that prevents overselling while preserving the low-latency, high-availability benefits of multi-leader replication.

**Requirements:** The system must detect inventory conflicts — situations where concurrent sales across leaders would reduce stock below zero. The conflict resolution policy must be deterministic and converge to the same result at all leaders regardless of the order in which replicated writes arrive (convergent conflict resolution). The system must support multiple resolution strategies: last-writer-wins (LWW) for non-critical fields like product descriptions, and a custom domain-specific strategy for inventory quantities that prevents negative stock. The design must handle the case where a conflict is detected after orders have already been confirmed to customers, defining a compensation mechanism (e.g., backorder or cancellation) for orders that cannot be fulfilled.

**Expected Approach:** Use a combination of conflict resolution techniques. For inventory quantities, use a CRDT-inspired approach: model each warehouse's sales as a grow-only counter (each warehouse tracks only its own sales), and compute the available stock as `total_replenished - sum(all_warehouse_sales)`. This is conflict-free because each warehouse only increments its own counter, and the global stock is computed by merging all counters. To prevent overselling, each warehouse reserves a portion of the total stock (e.g., Americas gets 4 units, Europe gets 3, Asia-Pacific gets 3) and can only sell up to its reservation without cross-region coordination. Selling beyond the reservation requires a synchronous check with the other leaders. For non-inventory fields, use LWW with a hybrid logical clock for ordering.

<details>
<summary>Hints</summary>

1. The root cause of the overselling problem is that each warehouse made a local decision (decrement stock) based on stale data (it had not yet seen the other warehouses' sales). Any solution must either (a) prevent concurrent decrements from exceeding the total stock, or (b) detect and compensate after the fact.
2. A CRDT (Conflict-free Replicated Data Type) approach avoids conflicts entirely by structuring the data so that concurrent updates are inherently mergeable. For inventory, a PN-Counter (positive-negative counter) tracks additions and subtractions per replica. The global value is the sum of all additions minus the sum of all subtractions. Each replica only modifies its own counters, so there are no conflicting updates — but this alone does not prevent the global value from going negative.
3. Stock reservation (also called "inventory partitioning" or "soft locking") assigns each warehouse a quota of the total stock. A warehouse can sell up to its quota without coordination. If it needs more, it must request a quota transfer from another warehouse — a synchronous operation that prevents overselling but adds latency for sales beyond the local quota.
4. For the compensation mechanism, consider that some oversold orders may have already been confirmed. The system should prioritize orders by timestamp (earliest confirmed orders are fulfilled first) and either backorder or cancel the remaining orders, notifying affected customers immediately.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** CRDT-based inventory counters for conflict-free merging, stock reservation per warehouse to prevent overselling, synchronous quota transfer for sales beyond the local reservation, and a compensation mechanism for edge cases.

1. **Data model — CRDT inventory counters:**
   - For each product, maintain a per-warehouse counter structure:
     ```
     product_inventory = {
       total_replenished: 100,        // global, updated via synchronous coordination
       sales: {
         americas: 0,                 // only Americas warehouse increments this
         europe: 0,                   // only Europe warehouse increments this
         asia_pacific: 0              // only Asia-Pacific warehouse increments this
       },
       reservations: {
         americas: 40,                // Americas can sell up to 40 without coordination
         europe: 30,                  // Europe can sell up to 30
         asia_pacific: 30             // Asia-Pacific can sell up to 30
       }
     }
     ```
   - Available stock at any warehouse = `reservation[warehouse] - sales[warehouse]`.
   - Global available stock = `total_replenished - sum(all sales)`.
   - Each warehouse only increments its own `sales` counter, so there are no write conflicts on the sales counters — they merge trivially by taking the maximum value seen for each warehouse's counter.

2. **Local sale (within reservation):**
   - Americas warehouse receives an order for 1 unit.
   - Check: `reservation[americas] - sales[americas] > 0`? If yes, increment `sales[americas]` by 1 and confirm the order. No cross-region coordination needed.
   - The sale is replicated asynchronously to Europe and Asia-Pacific, which update their view of `sales[americas]`.

3. **Sale beyond reservation (quota transfer):**
   - Americas warehouse has sold all 40 reserved units (`sales[americas] = 40`). A new order arrives.
   - Americas sends a synchronous quota transfer request to the other warehouses: "Transfer 10 units of reservation to Americas."
   - Europe checks its available reservation: `reservation[europe] - sales[europe] = 30 - 20 = 10`. It transfers 10 units: `reservation[europe] -= 10`, `reservation[americas] += 10`.
   - Americas now has 10 more units to sell locally. The transfer is replicated to all leaders.
   - If no warehouse has spare reservation, the order is either queued (backorder) or rejected.

4. **Convergent conflict resolution for non-inventory fields:**
   - Product descriptions, prices, and metadata use last-writer-wins (LWW) resolution.
   - Each write is tagged with a hybrid logical clock (HLC) timestamp that combines a physical clock with a logical counter to ensure uniqueness and monotonicity.
   - When two leaders concurrently update the same field, the write with the higher HLC timestamp wins. All leaders apply the same rule, so they converge to the same value regardless of the order in which they receive the replicated writes.

5. **Compensation mechanism for edge-case overselling:**
   - Despite reservations, a narrow race condition can occur: two warehouses simultaneously request quota transfers from the same source, and the source grants both before learning of the other request. This can cause a brief period of over-allocation.
   - A background reconciliation process runs every 30 seconds, computing `global_available = total_replenished - sum(all sales)`. If `global_available < 0`, the system has oversold.
   - Oversold orders are identified by timestamp (most recent confirmed orders are the ones that exceeded stock).
   - The system automatically creates backorders for the oversold quantity and notifies affected customers: "Your order is on backorder — expected fulfillment in 3–5 days."
   - If backorder is not possible (discontinued product), the system cancels the order and issues a refund.

6. **Why this approach prevents the original problem:**
   - In the original scenario, all three warehouses had unrestricted access to the full 10 units. With reservations of 4, 3, and 3, the maximum possible sales without coordination is 4 + 3 + 3 = 10 — exactly the total stock. Overselling is impossible within reservations.
   - Sales beyond reservations require synchronous coordination, which serializes the decision and prevents concurrent over-allocation.
   - The CRDT counter structure ensures that even if replication is delayed, the eventual merged state is correct — no sales are lost or double-counted.

</details>
