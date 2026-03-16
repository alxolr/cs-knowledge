# CAP Theorem

**Track:** Design Concepts
**Difficulty Tier:** Intermediate
**Prerequisites:** None

## Concept Overview

The CAP theorem, formulated by Eric Brewer and later proved by Seth Gilbert and Nancy Lynch, states that a distributed data store can simultaneously provide at most two of the following three guarantees: Consistency (every read receives the most recent write or an error), Availability (every request receives a non-error response, without guaranteeing it contains the most recent write), and Partition Tolerance (the system continues to operate despite an arbitrary number of messages being dropped or delayed by the network between nodes). Since network partitions are inevitable in any distributed system, the practical choice reduces to favoring either consistency or availability when a partition occurs.

A CP (Consistent and Partition-tolerant) system prioritizes consistency over availability. When a network partition happens, nodes that cannot confirm they have the latest data will refuse to serve reads or writes, returning errors instead. Examples include distributed coordination services like ZooKeeper and etcd, and strongly consistent databases like Google Spanner. These systems are appropriate when stale or conflicting data is unacceptable — for instance, in distributed locking, leader election, or financial ledgers.

An AP (Available and Partition-tolerant) system prioritizes availability over consistency. During a partition, every reachable node continues to accept reads and writes, even though different nodes may temporarily hold divergent versions of the same data. Once the partition heals, the system reconciles conflicts using strategies such as last-write-wins, vector clocks, or application-level merge functions. Examples include Amazon DynamoDB (in its eventually consistent mode), Apache Cassandra, and DNS. AP systems are well-suited for use cases where temporary staleness is tolerable, such as shopping carts, social media timelines, and content caches.

Understanding CAP is essential but not sufficient for designing real distributed systems. The theorem describes behavior only during a partition — it says nothing about the trade-offs a system makes when the network is healthy. The PACELC extension addresses this gap by adding a second dimension: when the network is partitioned, choose between Availability and Consistency (the PAC part); else, when the system is running normally, choose between Latency and Consistency (the ELC part). This richer model explains why systems like Cassandra (PA/EL — available during partitions, low latency otherwise) and Spanner (PC/EC — consistent always, higher latency) behave differently even when no partition is occurring.

---

## Problem 1 — Classifying Distributed Systems Under the CAP Theorem

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Given a set of distributed systems with described behaviors during network partitions, classify each system as CP or AP and justify your reasoning. For each system, identify which CAP guarantee is sacrificed and explain the practical consequences of that trade-off for the system's users.

### Scenario

**Context:** A cloud infrastructure company operates five internal distributed services. During a recent data center network split (a partition between the east and west availability zones lasting 12 minutes), each service behaved differently. The platform team wants to formally classify each service's CAP behavior to inform future architecture decisions and SLA definitions.

The five services are:
1. **Configuration Store:** During the partition, nodes in the minority partition stopped accepting reads and writes, returning "service unavailable" errors. Nodes in the majority partition continued operating normally after re-electing a leader. After the partition healed, all nodes converged to the same state with no conflicts.
2. **User Session Cache:** During the partition, all nodes continued serving reads and writes. Users logged in on the east side could not see session updates made on the west side. After the partition healed, conflicting sessions were resolved using last-write-wins based on timestamps.
3. **Inventory Counter:** During the partition, write requests were queued locally on each side. Both sides continued to accept decrement operations on the same inventory items. After the partition healed, some items showed negative inventory counts because both sides had independently sold the last units.
4. **Distributed Lock Service:** During the partition, the service refused to grant any new locks and existing lock holders could not renew. Applications waiting for locks received timeout errors. After the partition healed, the lock service resumed normal operation with no conflicting locks.
5. **DNS Cache:** During the partition, all nodes continued resolving queries using their local cache, even though updates published on one side were not visible on the other. After the partition healed, caches gradually converged through TTL expiration and refresh.

**Requirements:** For each of the five services, state whether it is CP or AP. Identify which guarantee (C or A) is sacrificed. Explain one practical consequence of the trade-off for end users or dependent services.

**Expected Approach:** Analyze each service's partition behavior. Services that reject requests during a partition sacrifice availability (CP). Services that continue serving potentially stale or divergent data sacrifice consistency (AP). The inventory counter illustrates a dangerous AP scenario where eventual consistency leads to real-world data integrity issues.

<details>
<summary>Hints</summary>

1. The key question for each service is: "During the partition, did it stop serving requests (sacrificing A) or did it serve potentially inconsistent data (sacrificing C)?"
2. CP systems typically use leader election or quorum mechanisms — the minority partition cannot form a quorum and stops operating.
3. AP systems typically use conflict resolution strategies (last-write-wins, CRDTs, application-level merge) that only run after the partition heals.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Classify each service by examining its behavior during the partition — whether it prioritized consistency (rejecting requests) or availability (serving potentially stale data).

1. **Configuration Store — CP:**
   - Sacrifices: Availability. Minority-partition nodes returned errors.
   - Justification: The service uses leader election (majority quorum). Nodes that cannot reach the leader refuse to serve data, ensuring every successful read returns the latest committed value.
   - Consequence: Applications depending on the configuration store in the minority partition experienced downtime for 12 minutes. Dependent services needed retry logic or fallback to cached configuration.

2. **User Session Cache — AP:**
   - Sacrifices: Consistency. Both sides served sessions independently, leading to divergent state.
   - Justification: All nodes continued accepting reads and writes. Conflicts were resolved after the partition healed using last-write-wins.
   - Consequence: A user who logged in on the east side and was routed to the west side during the partition might have seen a stale session or been asked to re-authenticate. Tolerable for session data.

3. **Inventory Counter — AP:**
   - Sacrifices: Consistency. Both sides independently decremented inventory.
   - Justification: Writes were accepted on both sides without coordination, leading to conflicting state (negative inventory).
   - Consequence: Overselling occurred — items were sold that did not exist. This is a case where AP behavior is dangerous and the system should likely be CP (or use a CRDT counter that only allows increments, with a separate reservation mechanism for decrements).

4. **Distributed Lock Service — CP:**
   - Sacrifices: Availability. No new locks were granted during the partition.
   - Justification: Granting locks on both sides of a partition would violate mutual exclusion — two processes could hold the same lock simultaneously. The service correctly chose to be unavailable rather than inconsistent.
   - Consequence: Applications waiting for locks were blocked for 12 minutes. Critical operations (e.g., database migrations, singleton job execution) were delayed but not corrupted.

5. **DNS Cache — AP:**
   - Sacrifices: Consistency. Nodes served stale cached records.
   - Justification: DNS is designed for availability — resolvers serve cached records even when they cannot reach authoritative servers. TTL-based expiration provides eventual consistency.
   - Consequence: DNS updates published during the partition were not visible on the other side until caches expired. For most use cases (web browsing, service discovery), short-lived staleness is acceptable.

**Key insight:** The inventory counter demonstrates that AP is not always the right choice. When data integrity has real-world consequences (money, inventory, locks), CP behavior — or a carefully designed CRDT — is usually preferable despite the availability cost.

</details>

---

## Problem 2 — Designing a CP Distributed Configuration Service

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Design a distributed configuration service that stores application settings (feature flags, connection strings, rate limits) and guarantees that every successful read returns the most recently written value, even during network partitions. The service must clearly sacrifice availability in the minority partition rather than serve stale configuration data. Your design should cover the consensus mechanism, read/write paths, and the behavior of nodes that are partitioned away from the majority.

### Scenario

**Context:** A fintech company runs 200 microservices that read configuration from a shared configuration service. A recent incident occurred when a stale feature flag was served during a network partition, causing a disabled payment processor to be re-enabled on one side of the partition. This resulted in duplicate charges to customers. The team has decided that configuration reads must always return the latest committed value — they would rather have a service return an error than serve stale data.

**Requirements:** The configuration service runs on 5 nodes distributed across 3 availability zones. It must tolerate the failure of up to 2 nodes (or a partition isolating up to 2 nodes) while continuing to serve consistent reads and writes from the majority partition. Nodes in the minority partition must reject all reads and writes with a clear error. Writes must be durable — once acknowledged, the value must not be lost even if a node crashes immediately after. The design should describe how a partitioned node rejoins and catches up after the partition heals.

**Expected Approach:** Use a consensus protocol (such as Raft or Paxos) that requires a majority quorum (3 out of 5 nodes) for both reads and writes. The leader accepts writes, replicates them to a majority, and then acknowledges. Reads are served by the leader (or via a read quorum) to ensure freshness. Nodes in the minority partition cannot form a quorum and refuse all operations. When the partition heals, minority nodes replay the log from the leader to catch up.

<details>
<summary>Hints</summary>

1. A majority quorum of 5 nodes is 3. Any partition that isolates 2 or fewer nodes leaves a majority of 3 that can continue operating. A partition that isolates 3 nodes means neither side has a majority — the entire system becomes unavailable (this is the cost of CP).
2. The leader handles all writes. A write is committed only after it is replicated to at least 3 nodes (the leader plus 2 followers). This ensures durability — even if the leader crashes, at least 2 other nodes have the write.
3. For linearizable reads, the leader must confirm it is still the leader before serving a read (it could have been deposed by a new election on the other side of a partition). One approach is a "read index" where the leader sends a heartbeat to a majority before responding to the read.
4. When a partitioned node rejoins, it discovers it is behind by comparing its log index with the leader's. It replays missing log entries from the leader until it is caught up, then resumes participating in quorum votes.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build a CP configuration service using Raft consensus across 5 nodes, requiring a 3-node majority quorum for all reads and writes.

1. **Cluster topology:**
   - 5 nodes: nodes 1 and 2 in zone A, nodes 3 and 4 in zone B, node 5 in zone C.
   - Majority quorum = 3 nodes. The system tolerates any 2 node failures or any partition that isolates at most 2 nodes.

2. **Leader election:**
   - Nodes elect a leader using Raft's term-based election. A candidate must receive votes from a majority (3 nodes) to become leader.
   - If a partition isolates 2 nodes, the 3-node majority side elects (or retains) a leader. The 2 isolated nodes cannot elect a leader because they cannot form a majority.

3. **Write path:**
   - A client sends a write (e.g., `SET feature.payments.enabled = false`) to the leader.
   - The leader appends the entry to its local log and replicates it to followers.
   - Once 3 nodes (including the leader) have persisted the entry, the leader commits it and responds to the client with success.
   - If the leader cannot reach a majority (e.g., it is on the minority side of a partition), the write times out and the client receives an error.

4. **Read path (linearizable):**
   - A client sends a read to the leader.
   - The leader confirms it is still the leader by sending a heartbeat to a majority of nodes and receiving acknowledgments.
   - Once confirmed, the leader reads the value from its committed state and responds to the client.
   - If the leader cannot confirm its leadership (partitioned from the majority), the read returns an error.
   - Alternative: clients can read from any node using a "read quorum" (query 3 nodes and take the value with the highest log index), but this adds latency.

5. **Minority partition behavior:**
   - Nodes 1 and 2 are isolated from nodes 3, 4, and 5.
   - Nodes 1 and 2 cannot form a quorum (need 3). They reject all reads and writes with an error: "No quorum available — service unavailable."
   - Clients connected to nodes 1 or 2 must retry against nodes in the majority partition or wait for the partition to heal.

6. **Partition recovery and catch-up:**
   - When the partition heals, nodes 1 and 2 receive heartbeats from the leader (on the majority side).
   - They discover their logs are behind (e.g., the leader is at log index 1042, node 1 is at 1035).
   - The leader sends the missing log entries (1036–1042) to nodes 1 and 2.
   - Once caught up, nodes 1 and 2 resume participating in quorum votes and can serve as followers for read distribution.

7. **Durability guarantee:**
   - A committed write exists on at least 3 nodes. Even if the leader crashes immediately after acknowledging, 2 other nodes have the entry. The next leader election will choose a node with the most up-to-date log, preserving the write.

**Why CP is correct here:** Serving stale configuration (e.g., a revoked API key still appearing valid, or a disabled feature flag appearing enabled) can cause financial damage, security breaches, or data corruption. The 12-minute unavailability during a partition is far less costly than the consequences of inconsistent configuration. Dependent services should implement local caching with a clear "last known good" fallback and alerting when the configuration service is unreachable.

</details>

---

## Problem 3 — Designing an AP Shopping Cart Service

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a distributed shopping cart service that remains fully available during network partitions, allowing customers to add and remove items from their carts on any reachable node even when nodes cannot communicate with each other. The service must reconcile divergent cart states after the partition heals without losing any items the customer intentionally added. Your design should cover the data model, conflict resolution strategy, and the trade-offs of choosing availability over consistency.

### Scenario

**Context:** An e-commerce platform operates across three geographic regions (US-East, US-West, Europe). Each region has its own cluster of cart service nodes. During normal operation, cart updates are replicated asynchronously across regions. During a recent transatlantic network partition lasting 8 minutes, customers in Europe could not see items they had added to their cart from a US-based mobile app moments before the partition. Worse, when the partition healed, some customers found items missing from their carts because a naive "last-write-wins" reconciliation strategy had discarded the European additions in favor of the older US-side state.

**Requirements:** The cart service must accept reads and writes on any reachable node during a partition (AP behavior). After a partition heals, the reconciled cart must contain the union of all items added on any side of the partition — no customer-added item may be silently dropped. Items explicitly removed by the customer on one side should be removed from the reconciled cart only if they were not re-added on the other side. The design should handle the case where the same item is added on one side and removed on the other side of the partition. Describe the data structure used to track cart state, the merge algorithm, and the replication protocol.

**Expected Approach:** Model the cart as a state-based CRDT (Conflict-free Replicated Data Type) — specifically, an Observed-Remove Set (OR-Set) — where each add operation is tagged with a unique identifier. The merge function takes the union of all add-tagged entries and applies removes only to the specific add-tags they observed. This ensures that a concurrent add and remove of the same item results in the item being present (add wins), which is the safest default for a shopping cart. Replication uses asynchronous gossip between regions, and the merge function is applied whenever two replicas synchronize.

<details>
<summary>Hints</summary>

1. A naive set with "add" and "remove" operations cannot be merged safely after a partition because you cannot distinguish between "item was never added" and "item was added then removed." Tagging each add with a unique ID solves this.
2. The OR-Set CRDT works as follows: each add(item) generates a unique tag (e.g., a UUID). The set stores pairs of (item, tag). A remove(item) removes all (item, tag) pairs currently visible on that replica. If another replica concurrently adds the same item with a new tag, the remove does not affect the new tag — so the item survives the merge.
3. The merge of two OR-Set replicas is: take the union of all (item, tag) pairs from both replicas, then subtract any (item, tag) pairs that were explicitly removed on either replica. The result contains an item if at least one un-removed tag exists for it.
4. For the shopping cart, "add wins" semantics (concurrent add and remove results in the item being present) is the safe default. It is better to show an extra item in the cart (the customer can remove it again) than to silently lose an item the customer wanted.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Model the shopping cart as an OR-Set CRDT with add-wins semantics, replicated asynchronously across regions via gossip.

1. **Data model — OR-Set:**
   - Each cart is identified by `cart_id` (e.g., the customer's user ID).
   - The cart state is a set of entries: `{ (item_id, unique_tag) }`.
   - Each replica also maintains a "tombstone set" of removed entries: `{ (item_id, unique_tag) }`.
   - An item is considered "in the cart" if at least one `(item_id, tag)` pair exists in the entry set that is not in the tombstone set.

2. **Add operation:**
   - `add(cart_id, item_id)`:
     - Generate a unique tag (e.g., UUID v4): `tag = "a3f7c2..."`.
     - Insert `(item_id, tag)` into the local entry set.
     - The operation is immediately visible on this replica.
     - The new entry is queued for asynchronous replication to other regions.

3. **Remove operation:**
   - `remove(cart_id, item_id)`:
     - Find all `(item_id, tag)` pairs currently in the local entry set.
     - Move them to the tombstone set: for each pair, add `(item_id, tag)` to tombstones and remove from the entry set.
     - This removes only the tags that this replica has observed. Tags added concurrently on other replicas (with different UUIDs) are unaffected.

4. **Merge algorithm (executed on replication sync):**
   - Given two replicas R1 and R2:
     - Merged entry set = (R1.entries ∪ R2.entries) − (R1.tombstones ∪ R2.tombstones).
     - Merged tombstone set = R1.tombstones ∪ R2.tombstones.
   - An item appears in the merged cart if at least one of its tags survived (was not tombstoned on either side).

5. **Conflict scenario — concurrent add and remove:**
   - Before the partition: cart contains `(widget, tag-1)`.
   - During the partition:
     - US-East: customer removes widget → `(widget, tag-1)` moves to tombstones.
     - Europe: customer adds widget again → `(widget, tag-2)` is created (new UUID).
   - After the partition heals, merge:
     - Entry set union: `{ (widget, tag-2) }` (tag-1 was removed on US-East).
     - Tombstone union: `{ (widget, tag-1) }`.
     - Result: `(widget, tag-2)` is present and not tombstoned → widget is in the cart.
   - The customer's European add survives the US-East remove. This is "add wins" — the safe default for a shopping cart.

6. **Replication protocol:**
   - Each region gossips cart updates to other regions every 500 ms (or on-demand when a sync is triggered).
   - The gossip payload contains new entries and new tombstones since the last sync.
   - On receiving a gossip message, the replica applies the merge algorithm.
   - During a partition, gossip messages are queued. When the partition heals, queued messages are delivered and merged.

7. **Trade-offs of AP for shopping carts:**
   - **Pro:** Customers can always modify their cart, even during partitions. No "service unavailable" errors.
   - **Pro:** No items are silently lost — add-wins semantics preserves customer intent.
   - **Con:** During a partition, a customer may see a stale cart (missing items added on the other side). This is temporary and resolves after the partition heals.
   - **Con:** The OR-Set tombstone set grows over time. Periodic garbage collection (pruning tombstones older than a threshold, e.g., 24 hours) is needed to bound storage.

**Why AP with OR-Set is correct here:** A shopping cart is a low-stakes, high-availability use case. A customer seeing a temporarily stale cart is far less harmful than being unable to add items during a sale event. The OR-Set CRDT guarantees that no intentional additions are lost, which aligns with the business priority of never losing a potential sale.

</details>

---

## Problem 4 — Tunable Consistency with Quorum Reads and Writes

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design a distributed key-value store that offers tunable consistency by allowing clients to specify the number of replicas that must acknowledge a read or write operation. The system should support a spectrum of consistency levels — from strong consistency to eventual consistency — by adjusting the read quorum (R), write quorum (W), and replication factor (N). Your design should explain how different combinations of R, W, and N produce different consistency guarantees, how the system behaves during node failures under each configuration, and the latency and availability trade-offs of each setting.

### Scenario

**Context:** A multi-tenant platform hosts three categories of data with different consistency requirements. Tenant A stores financial transaction records and requires strong consistency — every read must return the latest write. Tenant B stores user activity logs and can tolerate eventual consistency in exchange for low latency and high availability. Tenant C stores user profile data and wants a middle ground — reads should usually return the latest write, but occasional staleness (within a few seconds) is acceptable if it means lower latency. The platform uses a distributed key-value store with a replication factor of N=5 (each key is stored on 5 nodes). The team wants to serve all three tenants from the same cluster by tuning R and W per request rather than running separate clusters.

**Requirements:** Define the quorum parameters (R, W, N) for each tenant's consistency level. Prove mathematically why R + W > N guarantees strong consistency (the read set and write set must overlap). Show what happens when R + W ≤ N (the read may miss the latest write). For each tenant, calculate the maximum number of node failures the system can tolerate while still serving reads and writes. Analyze the latency characteristics: strong consistency requires waiting for the slowest node in the quorum, while eventual consistency returns as soon as the fastest node responds. Design a conflict resolution strategy for the eventual consistency case where a read returns multiple versions of the same key.

**Expected Approach:** For N=5: Tenant A uses R=3, W=3 (strong consistency, tolerates 2 failures). Tenant B uses R=1, W=1 (eventual consistency, tolerates 4 failures for reads, 4 for writes). Tenant C uses R=2, W=3 (strong consistency with lower read latency than R=3, tolerates 2 write failures and 3 read failures). The mathematical proof: if W=3 nodes store the latest write and R=3 nodes are read, with only 5 nodes total, at least 1 node must be in both sets (by the pigeonhole principle), guaranteeing the read sees the latest write.

<details>
<summary>Hints</summary>

1. The quorum intersection formula: R + W > N guarantees that the set of nodes that acknowledged the write and the set of nodes that respond to the read share at least one node. That shared node has the latest value, so the read can identify it (using version numbers or timestamps).
2. With R=1, W=1, N=5: a write goes to 1 node, a read goes to 1 node. There are 5 nodes total, so the probability that the read hits the node with the latest write is only 1/5 = 20%. The other 80% of the time, the read returns a stale value. This is eventual consistency — the stale replicas will eventually receive the update via background anti-entropy.
3. Latency is determined by the slowest node in the quorum. With R=3, the read latency is the time until the 3rd-fastest node responds (the p60 latency of the 5 nodes). With R=1, the read latency is the time until the fastest node responds (the p20 latency). This is why lower quorums are faster.
4. When R + W ≤ N and a read returns multiple versions, the system needs a conflict resolution strategy. Common approaches: (a) return the version with the highest timestamp (last-write-wins), (b) return all versions and let the client resolve (read repair), (c) use vector clocks to detect causal ordering and only flag true conflicts.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Configure per-tenant quorum parameters on a shared N=5 cluster, proving consistency guarantees mathematically and analyzing failure tolerance and latency for each configuration.

1. **Quorum parameters by tenant:**

   | Tenant | Use Case | R | W | R+W | > N? | Consistency Level |
   |--------|----------|---|---|-----|------|-------------------|
   | A | Financial transactions | 3 | 3 | 6 | Yes (6 > 5) | Strong |
   | B | Activity logs | 1 | 1 | 2 | No (2 ≤ 5) | Eventual |
   | C | User profiles | 2 | 3 | 5 | No (5 = 5) | Eventual* |

   *Note: R + W > N is required for strong consistency. R=2, W=3 gives R+W=5 which equals N, not exceeds it. For strong consistency, Tenant C should use R=3, W=3 or R=2, W=4. Adjusting to R=3, W=3 for strong consistency, or accepting R=2, W=3 as "almost strong" where a narrow race condition exists.

   Corrected table for strong consistency:

   | Tenant | R | W | R+W | Consistency |
   |--------|---|---|-----|-------------|
   | A | 3 | 3 | 6 > 5 | Strong |
   | B | 1 | 1 | 2 ≤ 5 | Eventual |
   | C | 2 | 4 | 6 > 5 | Strong (lower read latency) |

2. **Proof that R + W > N guarantees strong consistency:**
   - N = 5 nodes total. A write is acknowledged by W nodes. A read queries R nodes.
   - By the pigeonhole principle: if W + R > N, then the write set (W nodes) and the read set (R nodes) must share at least W + R − N nodes.
   - For Tenant A: overlap ≥ 3 + 3 − 5 = 1 node. At least one node in the read set has the latest write.
   - The read coordinator collects responses from R nodes, compares version numbers, and returns the value with the highest version. Since at least one response contains the latest write, the read is guaranteed to return it.

3. **What happens when R + W ≤ N (Tenant B):**
   - W=1: the write is stored on 1 node. R=1: the read queries 1 node.
   - The read node may or may not be the node that received the write. If it is not, the read returns a stale value.
   - Background anti-entropy (read repair, merkle tree sync, or gossip) eventually propagates the write to all 5 nodes. Until then, reads may return stale data.
   - For activity logs, this is acceptable — a log entry appearing with a few seconds of delay does not affect business logic.

4. **Failure tolerance by tenant:**

   | Tenant | R | W | Max node failures for reads | Max node failures for writes |
   |--------|---|---|----------------------------|------------------------------|
   | A | 3 | 3 | 2 (need 3 of 5 alive) | 2 (need 3 of 5 alive) |
   | B | 1 | 1 | 4 (need 1 of 5 alive) | 4 (need 1 of 5 alive) |
   | C | 2 | 4 | 3 (need 2 of 5 alive) | 1 (need 4 of 5 alive) |

   - Tenant B has the highest availability (tolerates 4 failures) but the weakest consistency.
   - Tenant C trades write availability (only 1 failure tolerated) for lower read latency (only 2 nodes needed).

5. **Latency analysis:**
   - Assume 5 nodes have response times: 1ms, 2ms, 4ms, 8ms, 15ms (sorted).
   - R=1: read latency = 1ms (fastest node). R=2: latency = 2ms. R=3: latency = 4ms.
   - W=1: write latency = 1ms. W=3: latency = 4ms. W=4: latency = 8ms.
   - Tenant A (R=3, W=3): read = 4ms, write = 4ms. Total round-trip = 8ms.
   - Tenant B (R=1, W=1): read = 1ms, write = 1ms. Total round-trip = 2ms. 4× faster than Tenant A.
   - Tenant C (R=2, W=4): read = 2ms, write = 8ms. Reads are fast, writes are slow.

6. **Conflict resolution for Tenant B (eventual consistency):**
   - When a read returns a value, it may be stale. The system attaches a vector clock or version number to each value.
   - If a read-repair mechanism is used: the coordinator reads from all 5 nodes in the background, identifies the latest version, and pushes it to stale nodes.
   - If the client reads a stale value, the next read (after anti-entropy) will return the correct value.
   - For true concurrent writes (two clients write different values to the same key simultaneously with W=1), the system uses last-write-wins (timestamp-based) or returns both versions to the client for application-level resolution.

**Key insight:** Tunable consistency lets a single cluster serve workloads with radically different requirements. The trade-off is always the same: higher quorums give stronger consistency but higher latency and lower fault tolerance. Lower quorums give better performance and availability but weaker consistency. The R + W > N rule is the bright line between "guaranteed to read the latest write" and "might read a stale value."

</details>

---

## Problem 5 — Applying the PACELC Framework to Evaluate Distributed Database Trade-offs

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

The CAP theorem only describes system behavior during a network partition. The PACELC extension adds a second dimension: when the system is running normally (no partition), it must still choose between lower latency and stronger consistency. Given a set of distributed databases with described behaviors in both partitioned and non-partitioned states, classify each using the PACELC framework and recommend which database is best suited for each of three application workloads with different consistency and latency requirements.

### Scenario

**Context:** A technology company is evaluating four distributed databases for three different application workloads. The company needs to understand not just how each database behaves during partitions (CAP), but also how it behaves during normal operation — specifically, whether it prioritizes low latency or strong consistency when the network is healthy. The PACELC framework (Partition: Availability vs. Consistency; Else: Latency vs. Consistency) provides the vocabulary for this analysis.

The four databases under evaluation:

1. **Database Alpha:** During a partition, nodes in the minority partition stop accepting writes and return errors for reads (CP behavior). During normal operation, all reads are routed to the leader node, which serves the latest committed value. Reads have higher latency (5–10ms) because they require a round-trip to the leader, even if a local replica is closer. Writes require majority acknowledgment before responding.

2. **Database Beta:** During a partition, all nodes continue accepting reads and writes (AP behavior). During normal operation, reads are served from the nearest replica with sub-millisecond latency, but the replica may be slightly behind the leader. Writes are acknowledged after being stored locally and replicated asynchronously. A read immediately after a write to a different node may not see the new value.

3. **Database Gamma:** During a partition, nodes in the minority partition stop accepting writes (CP behavior). During normal operation, the system offers two read modes: a "strong" mode that reads from the leader (10ms latency) and a "timeline" mode that reads from the nearest replica (1ms latency) with possible staleness. Writes always go through the leader with synchronous replication to a majority.

4. **Database Delta:** During a partition, all nodes continue accepting reads and writes (AP behavior). During normal operation, reads and writes are served locally with sub-millisecond latency. The system uses CRDTs for conflict-free merging. All replicas converge eventually, but there is no option for strong consistency even when the network is healthy.

The three application workloads:

- **Workload X — Banking Ledger:** Every read must return the latest committed balance. Latency of 10–15ms per operation is acceptable. Incorrect balances are unacceptable under any circumstances.
- **Workload Y — Social Media Timeline:** Users expect sub-millisecond read latency. Seeing a post a few seconds late is acceptable. The system must never show an error page, even during outages.
- **Workload Z — Collaborative Document Editor:** Users need low-latency reads for a responsive editing experience, but the system must guarantee that when a user saves a document, subsequent reads by any user return the saved version. Occasional higher latency during network issues is acceptable.

**Requirements:** Classify each database using the full PACELC notation (e.g., PA/EL, PC/EC, PA/EC, PC/EL). Justify each classification based on the described behavior. Map each workload to the best-fit database and explain why. Identify any workload where no single database is a perfect fit and describe what compromise the team would need to accept.

**Expected Approach:** Classify the databases as: Alpha = PC/EC, Beta = PA/EL, Gamma = PC/EC with optional EL mode, Delta = PA/EL. Map Workload X to Alpha (needs PC/EC), Workload Y to Beta or Delta (needs PA/EL), Workload Z to Gamma (needs PC for writes but benefits from EL for reads). Discuss that Workload Z is the hardest to fit because it wants both low latency and strong consistency — Gamma's dual-mode reads are the closest match.

<details>
<summary>Hints</summary>

1. The PACELC notation has two parts separated by a slash. The first part describes partition behavior: PA (available during partitions) or PC (consistent during partitions). The second part describes normal-operation behavior: EL (favors latency) or EC (favors consistency). So PA/EL means "available during partitions, low latency during normal operation."
2. Database Gamma is interesting because it offers a choice during normal operation (strong reads vs. timeline reads). This means it can operate as PC/EC or PC/EL depending on the read mode selected. This flexibility is valuable for workloads that need strong consistency for some operations and low latency for others.
3. For Workload Z (collaborative editor), the key insight is that writes need strong consistency (when a user saves, everyone must see the save) but reads can tolerate slight staleness for most keystrokes (the local user sees their own edits immediately via local state, and remote edits can arrive with a small delay). This maps to a system that is PC for writes and can switch between EC and EL for reads.
4. No database can be both PA and PC, or both EL and EC simultaneously for the same operation. PACELC forces a choice on each axis. The art of system design is matching the database's trade-off profile to the application's requirements.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Classify each database using PACELC notation by analyzing partition behavior and normal-operation behavior independently, then match each workload to the best-fit database.

1. **PACELC classification:**

   | Database | Partition Behavior | Normal Behavior | PACELC | Rationale |
   |----------|-------------------|-----------------|--------|-----------|
   | Alpha | PC — minority rejects reads/writes | EC — reads routed to leader, higher latency for consistency | PC/EC | Always prioritizes consistency over both availability and latency. |
   | Beta | PA — all nodes accept reads/writes | EL — reads from nearest replica, sub-ms latency, possible staleness | PA/EL | Always prioritizes availability and latency over consistency. |
   | Gamma | PC — minority rejects writes | EC or EL — configurable per read (strong mode = EC, timeline mode = EL) | PC/EC(EL) | Consistent during partitions; normal operation is tunable per query. |
   | Delta | PA — all nodes accept reads/writes, CRDTs for merging | EL — local reads/writes, sub-ms latency, no strong consistency option | PA/EL | Always prioritizes availability and latency; CRDTs ensure convergence but not recency. |

2. **Workload-to-database mapping:**

   **Workload X — Banking Ledger → Database Alpha (PC/EC):**
   - Requirement: every read returns the latest balance; errors preferred over stale data.
   - Alpha's PC behavior ensures no stale reads during partitions (minority nodes return errors).
   - Alpha's EC behavior ensures reads during normal operation always go to the leader, returning the latest committed value.
   - The 5–10ms read latency is within the acceptable 10–15ms budget.
   - Trade-off accepted: during a partition, the minority side is unavailable. For a banking ledger, this is the correct choice — serving a stale balance could lead to overdrafts or double-spending.

   **Workload Y — Social Media Timeline → Database Beta (PA/EL):**
   - Requirement: sub-millisecond reads, never show error pages, staleness is acceptable.
   - Beta's PA behavior ensures the timeline is always available, even during partitions.
   - Beta's EL behavior serves reads from the nearest replica with sub-ms latency.
   - Staleness (a post appearing a few seconds late) is explicitly acceptable.
   - Trade-off accepted: during a partition, different users may see different versions of the timeline. After the partition heals, replicas converge. This is the standard behavior users expect from social media platforms.
   - Database Delta would also work, but Beta is preferred if the team wants the option to upgrade to stronger consistency later (Beta could potentially be reconfigured; Delta's CRDT-only model has no strong consistency path).

   **Workload Z — Collaborative Document Editor → Database Gamma (PC/EC with EL option):**
   - Requirement: low-latency reads for responsive editing, but saves must be strongly consistent.
   - Gamma's PC behavior ensures that a saved document is never lost or overwritten by a stale version during a partition.
   - Gamma's dual-mode reads allow the application to use "timeline" mode (EL) for real-time keystroke syncing (low latency, slight staleness acceptable) and "strong" mode (EC) for document save confirmations (guaranteed to read the latest version).
   - Trade-off accepted: during a partition, the minority side cannot save documents (CP behavior). The application can buffer edits locally and retry when the partition heals. Real-time keystroke syncing may pause on the minority side, but no data is lost.

3. **Workload with no perfect fit:**
   - Workload Z is the hardest to fit. It wants both low latency (for keystrokes) and strong consistency (for saves) — which are opposing forces on the ELC axis.
   - Gamma's dual-mode reads are the closest match, but the application must explicitly choose the mode per operation, adding complexity to the client code.
   - A true "best of both worlds" would require a database that can dynamically switch between EC and EL per query, which Gamma supports but most databases do not.
   - If Gamma were unavailable, the team would need to split the workload: use a PA/EL database for real-time syncing and a PC/EC database for document persistence, with an application-level coordination layer.

4. **Summary table:**

   | Workload | Best Fit | PACELC Match | Key Trade-off |
   |----------|----------|--------------|---------------|
   | X — Banking | Alpha | PC/EC | Unavailability during partitions |
   | Y — Timeline | Beta | PA/EL | Stale reads during partitions and normal operation |
   | Z — Editor | Gamma | PC/EC(EL) | Complexity of dual-mode reads; unavailability for writes during partitions |

**Key insight:** The CAP theorem alone would classify Alpha and Gamma identically (both are CP) and Beta and Delta identically (both are AP). PACELC reveals the crucial difference in normal-operation behavior: Alpha always pays the latency cost for consistency, while Gamma lets you choose per query. Similarly, Beta and Delta are both PA/EL, but Delta's CRDT-only model offers no path to strong consistency, while Beta's asynchronous replication could potentially be tightened. PACELC is the more useful framework for real-world database selection because partitions are rare — the ELC trade-off dominates day-to-day system behavior.

</details>
