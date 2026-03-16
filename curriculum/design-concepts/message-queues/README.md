# Message Queues

**Track:** Design Concepts
**Difficulty Tier:** Intermediate
**Prerequisites:** None

## Concept Overview

Message queues are middleware components that enable asynchronous communication between services by temporarily storing messages in a buffer until the receiving service is ready to process them. Instead of one service calling another directly and waiting for a response, the sender places a message on a queue and continues its work. The receiver pulls messages from the queue at its own pace. This decoupling improves system resilience — if the receiver is temporarily down, messages accumulate in the queue rather than causing failures in the sender — and allows producers and consumers to scale independently.

Two fundamental messaging patterns dominate distributed systems. In the point-to-point model, each message is delivered to exactly one consumer. A pool of workers competes to pull messages from the queue, and once a worker acknowledges a message, it is removed. This pattern is ideal for task distribution where each unit of work should be processed once. In the publish-subscribe (pub/sub) model, a message published to a topic is delivered to all subscribers. Each subscriber receives its own copy of every message, enabling event broadcasting where multiple services need to react to the same event independently.

Reliable messaging introduces several challenges. Delivery guarantees range from at-most-once (messages may be lost but are never duplicated), through at-least-once (messages are never lost but may be delivered more than once), to exactly-once (each message is processed precisely once, the hardest guarantee to achieve). Dead letter queues (DLQs) capture messages that fail processing repeatedly, preventing poison messages from blocking the main queue. Message ordering — ensuring consumers process messages in the sequence they were produced — is straightforward on a single queue but becomes complex when partitioning for throughput. Backpressure mechanisms allow a consumer to signal that it is overwhelmed, causing the producer or broker to slow down rather than flooding the system with unprocessable messages.

---

## Problem 1 — Point-to-Point Queue for Image Processing Tasks

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a task distribution system for an image processing pipeline where uploaded images must be resized into multiple formats. Each image should be processed exactly once, even when multiple worker instances are running. Your design should explain how a point-to-point queue distributes work across workers and how the system handles a worker crashing mid-processing.

### Scenario

**Context:** A photo-sharing platform allows users to upload images. Each upload triggers the creation of four resized variants (thumbnail, small, medium, large). The resizing is CPU-intensive and takes 2–5 seconds per variant. The platform runs six worker instances that pull resizing jobs from a shared queue. During peak hours, users upload 500 images per minute, producing 2,000 resize jobs per minute.

**Requirements:** Each resize job must be processed by exactly one worker — no duplicates and no missed jobs. If a worker crashes while processing a job, that job must be retried by another worker. The system must handle variable upload rates without losing jobs. Workers should be independently scalable — adding more workers should increase throughput linearly without configuration changes.

**Expected Approach:** Use a point-to-point message queue where the upload service enqueues one message per resize job. Workers compete to dequeue messages. Use visibility timeouts and acknowledgment to ensure crash recovery. Explain how competing consumers naturally distribute load.

<details>
<summary>Hints</summary>

1. In a point-to-point queue, each message is delivered to exactly one consumer. When multiple workers poll the same queue, the broker ensures only one worker receives each message.
2. Use a visibility timeout: when a worker pulls a message, it becomes invisible to other workers for a set period (e.g., 30 seconds). If the worker finishes and acknowledges the message, it is permanently deleted. If the worker crashes and the timeout expires, the message reappears on the queue for another worker to pick up.
3. Adding more workers does not require any queue configuration changes — new workers simply start polling the same queue and begin receiving messages immediately.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use a point-to-point message queue with visibility timeouts and explicit acknowledgment to distribute resize jobs across a pool of competing workers.

1. **Message production:**
   - When a user uploads an image, the upload service publishes four messages to the resize queue — one per variant (thumbnail, small, medium, large).
   - Each message contains the image ID, the target variant, and the storage location of the original image.

2. **Competing consumers:**
   - Six worker instances long-poll the same queue. The broker delivers each message to exactly one worker.
   - Workers process messages independently and in parallel. At 2,000 jobs per minute and 6 workers each handling ~4 seconds per job, each worker processes ~15 jobs per minute, well within capacity.

3. **Visibility timeout and acknowledgment:**
   - When a worker receives a message, the broker marks it invisible for 30 seconds (the visibility timeout).
   - If the worker completes the resize and sends an acknowledgment (ACK), the message is permanently deleted from the queue.
   - If the worker crashes or takes longer than 30 seconds without acknowledging, the message becomes visible again and another worker picks it up.
   - Set the visibility timeout to 2–3× the expected processing time to avoid premature re-delivery while still recovering from crashes promptly.

4. **Scaling:**
   - Adding three more workers during peak hours requires no queue changes — the new workers poll the same queue and immediately begin receiving messages.
   - The queue acts as a buffer: if uploads spike faster than workers can process, messages accumulate in the queue and are drained as workers catch up.

5. **Failure handling:**
   - Worker crash: the visibility timeout expires and the message is redelivered to another worker.
   - Repeated failures: after a configurable number of delivery attempts (e.g., 3), the message is moved to a dead letter queue for manual inspection.

**Why point-to-point fits:** Each resize job is a discrete unit of work that should be done exactly once. The competing consumer pattern naturally distributes load across workers, and the visibility timeout mechanism provides crash recovery without complex coordination.

</details>

---

## Problem 2 — Pub/Sub for E-Commerce Order Event Broadcasting

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Design an event broadcasting system for an e-commerce platform where a single order event must be consumed by multiple independent services — inventory, shipping, analytics, and email notification. Each service processes the event differently and at its own pace. Your design should explain how pub/sub decouples the order service from its consumers and how each subscriber maintains independent processing progress.

### Scenario

**Context:** When a customer places an order, four downstream services must react: the inventory service reserves stock, the shipping service schedules a pickup, the analytics service records the sale for reporting, and the email service sends an order confirmation. Currently, the order service makes synchronous HTTP calls to each downstream service in sequence. If the analytics service is slow or the email service is down, the entire order placement is delayed or fails. The team wants to decouple the order service from its consumers so that a slow or failing downstream service does not affect order placement.

**Requirements:** The order service must publish a single event per order, and all four downstream services must receive their own copy of that event. Each service must process events independently — a failure in the email service must not affect inventory reservation. Services must be able to process events at different speeds without losing messages. Adding a new consumer (e.g., a fraud detection service) must not require changes to the order service. The design should address how each subscriber tracks its own processing position and what happens when a subscriber falls behind.

**Expected Approach:** Use a pub/sub topic where the order service publishes order events. Each downstream service subscribes to the topic and receives every message independently. Each subscription maintains its own offset or acknowledgment state, so a slow subscriber does not block others. Discuss fan-out delivery, independent consumer groups, and message retention for slow subscribers.

<details>
<summary>Hints</summary>

1. In pub/sub, a message published to a topic is delivered to every active subscription. Unlike point-to-point, where one consumer gets each message, pub/sub fans out each message to all subscribers.
2. Each subscription acts as an independent queue with its own acknowledgment tracking. The inventory service can be caught up to the latest message while the analytics service is still processing messages from 5 minutes ago — neither blocks the other.
3. Message retention on the topic (e.g., 7 days) ensures that a subscriber that goes down for maintenance can resume from where it left off without losing messages, as long as it catches up within the retention window.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use a pub/sub topic for order events with independent subscriptions per downstream service, each tracking its own processing position.

1. **Topic and publishing:**
   - Create an `order-events` topic. The order service publishes one message per order containing the order ID, customer ID, item list, total amount, and timestamp.
   - The order service's only responsibility is to publish the event and return success to the customer. It has no knowledge of which services consume the event.

2. **Independent subscriptions:**
   - Each downstream service creates its own subscription to the `order-events` topic:
     - `inventory-subscription` — the inventory service pulls messages and reserves stock.
     - `shipping-subscription` — the shipping service pulls messages and schedules pickups.
     - `analytics-subscription` — the analytics service pulls messages and records sales data.
     - `email-subscription` — the email service pulls messages and sends confirmations.
   - Each subscription maintains its own offset (position in the message stream). The inventory service can acknowledge message 1,000 while the analytics service is still on message 950.

3. **Fan-out delivery:**
   - When the order service publishes message N, the broker delivers a copy of message N to each of the four subscriptions. Each service receives and processes the same event independently.
   - If the email service is down, messages accumulate in the `email-subscription` backlog. The other three services continue processing normally.

4. **Handling slow subscribers:**
   - The topic retains messages for a configurable period (e.g., 7 days). A subscriber that falls behind can catch up by processing its backlog.
   - If a subscriber is consistently slower than the publish rate, it needs to scale horizontally (add more consumer instances within the same subscription) or the retention period must be extended.
   - Monitor each subscription's lag (the difference between the latest published offset and the subscription's acknowledged offset). Alert if lag exceeds a threshold.

5. **Adding new consumers:**
   - A new fraud detection service creates a `fraud-subscription` on the same topic. It immediately begins receiving new messages. If it needs historical events, it can start reading from the beginning of the retention window.
   - The order service requires zero changes — it continues publishing to the same topic.

6. **Failure isolation:**
   - A crash in the email service only affects the `email-subscription`. Messages accumulate until the service recovers and drains its backlog.
   - The order service never blocks or retries because of a downstream failure — it publishes and moves on.

**Why pub/sub fits:** A single event must reach multiple independent consumers, each with different processing logic and speed. Pub/sub provides fan-out delivery with independent progress tracking, completely decoupling the producer from its consumers. Adding or removing consumers requires no changes to the publisher.

</details>

---

## Problem 3 — Dead Letter Queues and Retry Strategies for Payment Processing

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a retry and dead letter queue strategy for a payment processing pipeline where some messages fail due to transient errors (network timeouts, temporary service unavailability) and others fail due to permanent errors (invalid account numbers, insufficient funds). Your design should distinguish between retryable and non-retryable failures, implement an exponential backoff retry policy, and route permanently failed messages to a dead letter queue for manual review.

### Scenario

**Context:** A fintech platform processes payment transactions via a message queue. A payment service publishes transaction messages, and a processing worker validates the transaction, calls an external bank API, and records the result. Approximately 5% of transactions fail on the first attempt: 3% due to transient errors (bank API timeouts, network blips) that succeed on retry, and 2% due to permanent errors (invalid routing numbers, closed accounts) that will never succeed regardless of retries. Currently, all failed messages are retried indefinitely, causing poison messages (permanent failures) to cycle through the queue forever, consuming worker capacity and delaying legitimate transactions.

**Requirements:** Implement a retry policy with exponential backoff for transient failures (retry after 1s, 2s, 4s, 8s, up to a maximum of 5 retries). Messages that exceed the maximum retry count must be moved to a dead letter queue. Permanent failures must be identified immediately (without exhausting retries) and routed directly to the dead letter queue. The dead letter queue must preserve the original message, the failure reason, and the number of attempts for manual investigation. The design should include a mechanism for operators to inspect DLQ messages and either fix and replay them or discard them.

**Expected Approach:** Classify errors into transient and permanent categories at the worker level. For transient errors, requeue the message with an incremented retry counter and a delay based on exponential backoff. For permanent errors, send the message directly to the DLQ. After 5 failed retries of a transient error, move the message to the DLQ. Provide an operator tool to inspect, replay, or purge DLQ messages.

<details>
<summary>Hints</summary>

1. Add metadata to each message: a `retryCount` field (initially 0) and an `errorType` field (set by the worker on failure). When a worker catches a transient error, it increments `retryCount` and requeues the message with a delay of `2^retryCount` seconds. When it catches a permanent error, it sends the message directly to the DLQ.
2. The dead letter queue is a separate queue that stores failed messages along with diagnostic metadata (original message body, error reason, timestamp of each attempt, total attempt count). Operators can query this queue to understand failure patterns.
3. A replay mechanism reads a message from the DLQ, optionally modifies it (e.g., corrects an invalid account number), and republishes it to the main processing queue. A purge mechanism permanently deletes messages that are confirmed unrecoverable.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Classify failures at the worker level, apply exponential backoff retries for transient errors, route permanent failures directly to the DLQ, and provide operator tooling for DLQ management.

1. **Error classification:**
   - The processing worker wraps the bank API call in error handling that distinguishes transient errors (HTTP 503, connection timeout, socket reset) from permanent errors (HTTP 400 with "invalid account," HTTP 422 with "insufficient funds").
   - Transient errors trigger the retry path. Permanent errors trigger the DLQ path immediately.

2. **Exponential backoff retry:**
   - Each message carries a `retryCount` metadata field, initialized to 0.
   - On a transient failure, the worker increments `retryCount` and requeues the message with a delay of `min(2^retryCount, 16)` seconds:
     - Attempt 1 fails → requeue with 1s delay (retryCount = 1).
     - Attempt 2 fails → requeue with 2s delay (retryCount = 2).
     - Attempt 3 fails → requeue with 4s delay (retryCount = 3).
     - Attempt 4 fails → requeue with 8s delay (retryCount = 4).
     - Attempt 5 fails → requeue with 16s delay (retryCount = 5).
   - If `retryCount` exceeds 5, the message is moved to the DLQ instead of being requeued.

3. **Dead letter queue structure:**
   - The DLQ is a separate queue (e.g., `payments-dlq`) that stores enriched messages containing:
     - The original message body (transaction ID, amount, account details).
     - The `errorType` (transient or permanent) and the specific error message from the last failure.
     - The `retryCount` (number of attempts made).
     - Timestamps of the first attempt and the final failure.
   - DLQ messages have a long retention period (e.g., 30 days) to give operators time to investigate.

4. **Permanent failure fast path:**
   - When the worker detects a permanent error (e.g., invalid routing number), it skips the retry loop entirely and sends the message to the DLQ with `retryCount = 1` and `errorType = permanent`.
   - This prevents poison messages from consuming 5 retry cycles and wasting worker capacity.

5. **Operator tooling:**
   - **Inspect:** A dashboard or CLI tool lists DLQ messages with their failure reasons, grouped by error type. Operators can filter by date, error category, or transaction ID.
   - **Replay:** An operator selects a message, optionally edits it (e.g., corrects a typo in an account number), and republishes it to the main processing queue with `retryCount` reset to 0.
   - **Purge:** Messages confirmed as unrecoverable (e.g., test transactions, duplicate submissions) are permanently deleted from the DLQ.
   - **Bulk replay:** After a transient outage is resolved, operators can replay all DLQ messages with `errorType = transient` in a single batch.

6. **Monitoring:**
   - Track the DLQ depth (number of messages). A rising DLQ depth indicates a systemic issue (e.g., the bank API is down, or a code bug is misclassifying errors).
   - Alert if the DLQ receives more than N messages per hour, triggering investigation before the backlog grows.

**Why this approach fits:** Blindly retrying all failures wastes resources on permanent errors and delays recovery for transient ones. Classifying errors at the source, applying bounded retries with backoff for transient failures, and fast-tracking permanent failures to the DLQ keeps the main queue healthy and gives operators the information they need to resolve issues.

</details>

---

## Problem 4 — Message Ordering and Idempotency for a Financial Ledger

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design a messaging system for a financial ledger service where account balance updates must be processed in the exact order they were produced, and each update must be applied exactly once — even if the message is delivered more than once. The system must maintain ordering guarantees while still allowing horizontal scaling of consumers across multiple account partitions. Your design should address how ordering is preserved, how idempotency is enforced, and what trade-offs arise between strict ordering and throughput.

### Scenario

**Context:** A banking platform publishes account transaction events (deposits, withdrawals, transfers) to a message queue. A ledger service consumes these events and updates account balances. For correctness, transactions on the same account must be applied in order: if account A receives a deposit of $100 followed by a withdrawal of $50, the ledger must process the deposit before the withdrawal. However, transactions on different accounts are independent and can be processed in parallel. The platform handles 100,000 transactions per second across 10 million accounts. A single consumer cannot keep up with this volume, so the team needs to partition the workload across multiple consumers while preserving per-account ordering. Additionally, the at-least-once delivery guarantee of the message broker means some messages may be delivered twice, and applying a transaction twice would corrupt the account balance.

**Requirements:** Guarantee that all transactions for a given account are processed in the order they were published. Allow transactions for different accounts to be processed in parallel by different consumers. Ensure that duplicate message deliveries do not result in duplicate balance updates (idempotent processing). The design must scale to 100,000 transactions per second. Explain the partitioning strategy, the ordering guarantee mechanism, and the idempotency enforcement approach. Address what happens when a consumer crashes and its partition is reassigned.

**Expected Approach:** Partition the topic by account ID so that all transactions for the same account land on the same partition, which is consumed by a single consumer. This guarantees per-account ordering. For idempotency, assign each transaction a unique ID and have the ledger service track processed IDs — if a duplicate arrives, it is skipped. Discuss the trade-off between partition count (more partitions = more parallelism but more overhead) and consumer count.

<details>
<summary>Hints</summary>

1. Use the account ID as the partition key. A hash of the account ID determines which partition the message lands on. Since all messages for account A go to the same partition, and each partition is consumed by exactly one consumer at a time, per-account ordering is guaranteed.
2. For idempotency, each transaction message carries a globally unique `transactionId`. The ledger service maintains a set of recently processed transaction IDs (in a database or in-memory cache with a TTL). Before applying a transaction, the service checks whether the `transactionId` has already been processed. If yes, it acknowledges the message without applying it again.
3. When a consumer crashes, its partitions are reassigned to other consumers in the group (rebalancing). The new consumer resumes from the last committed offset for each reassigned partition. Some messages may be redelivered (between the last committed offset and the crash point), but the idempotency check prevents double-processing.
4. The number of partitions sets the upper bound on consumer parallelism. With 50 partitions, at most 50 consumers can process in parallel. Choose a partition count that balances parallelism needs against the overhead of managing many partitions.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Partition the transaction topic by account ID for per-account ordering, enforce idempotency via unique transaction IDs, and scale consumers up to the partition count.

1. **Partitioning by account ID:**
   - The transaction topic is divided into N partitions (e.g., 200 partitions for 100K TPS).
   - The producer assigns each message to a partition using `hash(accountId) % N`. All transactions for account A always land on the same partition.
   - Each partition is an ordered, append-only log. Messages within a partition are strictly ordered by their offset.

2. **Consumer group and ordering guarantee:**
   - A consumer group of M consumers (M ≤ N) subscribes to the topic. The broker assigns each partition to exactly one consumer in the group.
   - Since each partition is consumed by one consumer, and all messages for a given account are on the same partition, per-account ordering is guaranteed.
   - Transactions for different accounts on different partitions are processed in parallel by different consumers.

3. **Idempotent processing:**
   - Each transaction message includes a unique `transactionId` (e.g., a UUID generated by the producing service).
   - The ledger service maintains an idempotency store — a database table or cache mapping `transactionId → processed (boolean)`.
   - Processing flow:
     1. Consumer receives message with `transactionId = txn-12345`.
     2. Check idempotency store: has `txn-12345` been processed?
     3. If yes → acknowledge the message and skip (duplicate delivery).
     4. If no → apply the balance update, write `txn-12345` to the idempotency store, and acknowledge the message.
   - The idempotency check and the balance update should be performed atomically (within the same database transaction) to prevent a crash between the update and the idempotency write from causing inconsistency.

4. **Consumer crash and rebalancing:**
   - Consumers periodically commit their current offset (the position of the last successfully processed message) to the broker.
   - If a consumer crashes, the broker reassigns its partitions to surviving consumers. The new consumer resumes from the last committed offset.
   - Messages between the last committed offset and the crash point are redelivered. The idempotency check ensures these redelivered messages are not applied twice.

5. **Scaling and partition count trade-offs:**
   - 200 partitions allow up to 200 parallel consumers, each handling ~500 TPS — well within a single consumer's capacity.
   - More partitions increase parallelism but add broker overhead (more metadata, more rebalancing complexity). Fewer partitions limit maximum consumer count.
   - The partition count should be chosen based on peak throughput requirements with headroom for growth (e.g., 2× current peak).

6. **Ordering vs. throughput trade-off:**
   - Strict global ordering (all transactions in one partition) limits throughput to a single consumer. Per-account ordering via partitioning allows massive parallelism while preserving the ordering that matters for correctness.
   - If two accounts are involved in a transfer (A sends to B), the deposit on B and the withdrawal on A are on different partitions and may be processed out of order relative to each other. This is acceptable because each account's balance is independently consistent. Cross-account consistency is handled at the application level (e.g., saga pattern).

**Why partitioning with idempotency fits:** Partitioning by account ID provides the exact ordering granularity needed — per-account, not global — while enabling horizontal scaling. Idempotent processing via unique transaction IDs handles the inevitable duplicate deliveries from at-least-once semantics without corrupting balances.

</details>

---

## Problem 5 — Backpressure and Flow Control for a Real-Time Data Ingestion Pipeline

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a backpressure and flow control mechanism for a real-time data ingestion pipeline where IoT sensors produce data at variable rates and the downstream processing service has a fixed throughput capacity. During traffic spikes, the producer rate can exceed the consumer rate by 10×, and without flow control the message broker's storage fills up, causing data loss or broker instability. Your design should implement multiple layers of backpressure — from consumer to broker to producer — and define degradation strategies when the system is overwhelmed.

### Scenario

**Context:** A smart factory operates 50,000 IoT sensors that publish telemetry data (temperature, vibration, pressure) to a central message broker. Under normal conditions, sensors collectively produce 20,000 messages per second, and the processing pipeline (anomaly detection, aggregation, storage) handles 25,000 messages per second — comfortable headroom. During a factory-wide calibration event, all sensors increase their reporting frequency, spiking production to 200,000 messages per second for 10–15 minutes. The processing pipeline cannot scale instantly to match this rate. The broker has 50 GB of disk storage for message buffering, which fills up in approximately 4 minutes at the spike rate if consumers cannot keep pace. Once the broker's storage is full, it begins rejecting new messages, and sensor data is permanently lost.

**Requirements:** Design a multi-layered backpressure system that prevents broker storage exhaustion and minimizes data loss during traffic spikes. The system must include: (1) consumer-side flow control that signals when consumers are falling behind, (2) broker-level buffering with configurable limits and overflow policies, (3) producer-side rate limiting that throttles sensor output when the pipeline is saturated. Define a graceful degradation strategy — what data is dropped first when the system cannot keep up (e.g., reduce sampling frequency, drop low-priority sensors). The design should recover automatically when the spike subsides, draining the backlog and restoring normal operation.

**Expected Approach:** Implement backpressure as a feedback loop: consumers report their lag to the broker, the broker monitors queue depth and applies overflow policies (reject, drop oldest, drop by priority), and producers monitor broker health metrics or receive explicit slow-down signals to reduce their publish rate. Use a tiered degradation strategy: first buffer, then reduce sampling, then drop low-priority data. Design for automatic recovery when consumer throughput exceeds producer rate again.

<details>
<summary>Hints</summary>

1. Consumer lag (the gap between the latest published offset and the consumer's current offset) is the primary signal that the pipeline is falling behind. Monitor this metric continuously and trigger backpressure actions when lag exceeds defined thresholds.
2. The broker can implement multiple overflow policies: block producers (apply TCP-level backpressure so producers slow down), drop the oldest messages (acceptable if recent data is more valuable), or drop messages by priority (low-priority sensors are dropped first).
3. Producer-side rate limiting can be implemented by having sensors query a central rate-limit configuration service that adjusts the allowed publish rate based on current broker health. During a spike, the service reduces the allowed rate from 4 msg/s per sensor to 1 msg/s, cutting total production by 75%.
4. Tiered degradation: Level 0 (normal) → Level 1 (buffer absorbs spike) → Level 2 (reduce sampling frequency) → Level 3 (drop low-priority sensors entirely). Each level is triggered by a queue depth or consumer lag threshold.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement a three-layer backpressure system (consumer → broker → producer) with tiered degradation levels and automatic recovery.

1. **Layer 1 — Consumer-side flow control:**
   - Each consumer tracks its own processing rate and reports consumer lag (latest offset minus current offset) to a monitoring service every 5 seconds.
   - When consumer lag exceeds Threshold-1 (e.g., 60 seconds of backlog), the monitoring service triggers Level 1 backpressure.
   - Consumers use a prefetch limit (e.g., fetch at most 100 messages per poll) to avoid pulling more messages than they can process, preventing local memory exhaustion.

2. **Layer 2 — Broker-level buffering and overflow policies:**
   - The broker allocates 50 GB of disk for message buffering, with watermark thresholds:
     - Low watermark (60% / 30 GB): normal operation, no action.
     - High watermark (80% / 40 GB): trigger Level 2 backpressure — the broker begins applying TCP-level backpressure to producers (slowing their socket writes) and activates priority-based drop policy.
     - Critical watermark (95% / 47.5 GB): trigger Level 3 backpressure — the broker rejects messages from low-priority producers entirely.
   - Priority-based drop policy: each message carries a priority field (critical, standard, low). When the high watermark is breached, the broker drops `low` priority messages first, then `standard`, preserving `critical` messages as long as possible.

3. **Layer 3 — Producer-side rate limiting:**
   - Sensors query a rate-limit configuration service on startup and periodically (every 30 seconds) to get their allowed publish rate.
   - Under normal conditions: 4 messages/second per sensor (50,000 × 4 = 200K max, but sensors typically publish at lower rates).
   - Level 2 backpressure: the configuration service reduces the allowed rate to 1 message/second per sensor, cutting total production to 50,000 msg/s.
   - Level 3 backpressure: the configuration service instructs low-priority sensors to stop publishing entirely, and standard-priority sensors to reduce to 0.5 msg/s.
   - Sensors that cannot reach the configuration service fall back to a conservative default rate (1 msg/s).

4. **Tiered degradation strategy:**
   - **Level 0 (normal):** Consumer lag < 60s, broker storage < 60%. All sensors publish at full rate. No data loss.
   - **Level 1 (buffering):** Consumer lag > 60s or broker storage > 60%. The broker absorbs the spike using its disk buffer. No data is dropped yet. Monitoring alerts are raised.
   - **Level 2 (reduced sampling):** Consumer lag > 5 minutes or broker storage > 80%. Producer rate is reduced. Broker drops low-priority messages if storage continues to rise. Some data loss for low-priority sensors.
   - **Level 3 (selective shedding):** Broker storage > 95%. Low-priority sensors stop publishing. Standard-priority sensors reduce to minimum rate. Only critical sensors (safety-related) publish at full rate. Significant data loss for non-critical sensors, but the broker remains stable and critical data is preserved.

5. **Automatic recovery:**
   - When the spike subsides and consumer throughput exceeds producer rate, consumer lag begins decreasing.
   - As lag drops below each threshold, the system steps down through degradation levels: Level 3 → Level 2 → Level 1 → Level 0.
   - The configuration service gradually restores sensor publish rates over a 5-minute ramp-up period (not instantly) to avoid re-triggering the spike.
   - The broker drains its buffer as consumers catch up. Once storage drops below the low watermark, all overflow policies are deactivated.

6. **Monitoring and alerting:**
   - Dashboard showing: consumer lag per partition, broker storage utilization, current degradation level, producer publish rate vs. consumer processing rate.
   - Alerts: Level 1 triggers a warning. Level 2 triggers a page to the on-call engineer. Level 3 triggers an incident.
   - Post-spike review: analyze the duration and magnitude of the spike to determine if the processing pipeline needs permanent scaling or if the degradation strategy performed adequately.

**Why multi-layered backpressure fits:** No single mechanism is sufficient. Consumer-side flow control prevents local overload but cannot slow producers. Broker-level policies protect broker stability but cannot prevent messages from arriving. Producer-side rate limiting reduces inflow but needs a signal from downstream. The three layers form a feedback loop: consumers detect overload, the broker enforces limits, and producers adapt their rate — keeping the entire pipeline stable under extreme load while preserving the most valuable data.

</details>
