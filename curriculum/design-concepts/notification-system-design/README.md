# Notification System Design

**Track:** Design Concepts
**Difficulty Tier:** Advanced
**Prerequisites:** [Message Queues](../message-queues/README.md), [Microservices Architecture](../microservices-architecture/README.md), [Chat System Design](../chat-system-design/README.md)

## Concept Overview

A notification system is the backbone of user engagement for any modern application. Its job is deceptively simple — deliver the right message to the right user through the right channel at the right time — but achieving this reliably at scale is a significant engineering challenge. Notification systems must support multiple delivery channels (push notifications, email, SMS, and in-app), each with its own protocol, latency profile, rate limits, and failure modes. A single user action, such as a friend request or a payment confirmation, can trigger notifications across several of these channels simultaneously, and the system must coordinate delivery without sending duplicates or overwhelming the user.

Beyond simple delivery, a production notification system must handle routing and prioritization. A security alert about a compromised account is far more urgent than a weekly digest email, and the system must reflect this difference in how quickly and aggressively it delivers each type. Routing logic determines which channels to use based on notification type, user preferences, and real-time context (e.g., suppress push notifications during the user's configured quiet hours). Prioritization ensures that high-urgency notifications are processed ahead of bulk marketing messages, even when the system is under heavy load.

At scale, notification systems face unique reliability challenges. A flash sale announcement sent to 50 million users must be delivered within minutes, not hours, requiring massive horizontal throughput. At the same time, transactional notifications like password reset codes must arrive within seconds and must never be lost. Achieving exactly-once delivery semantics — where every notification is delivered precisely once despite retries, network failures, and provider outages — requires careful idempotency design, dead-letter handling, and provider failover. The problems in this module explore these dimensions: multi-channel delivery, intelligent routing, user preference management, aggregation and batching, and reliability at scale.

---

## Problem 1 — Multi-Channel Notification Delivery Architecture

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Design the high-level architecture for a notification system that delivers notifications through four channels: push notifications (mobile), email, SMS, and in-app. The design should cover how a notification request enters the system, how it is routed to the appropriate channel(s), and how each channel's delivery adapter handles the specifics of its protocol. Focus on the core delivery flow for a single notification sent to a single user.

### Scenario

**Context:** A fintech startup is building a unified notification service to replace four separate, independently built notification scripts scattered across their codebase. Currently, the payments team sends emails directly via an SMTP library, the fraud team sends SMS through a Twilio wrapper, the mobile team manages push notifications through Firebase, and in-app notifications are written directly to a database table. There is no central place to see what notifications a user has received, no consistent retry logic, and adding a new channel (e.g., WhatsApp) requires changes in every team's code. The company wants a single notification service that all teams call through one API, which then routes and delivers through the appropriate channel(s).

**Requirements:** Design a single entry-point API that accepts a notification request containing the recipient, notification type, content, and desired channel(s). Define the internal architecture that routes the request to one or more channel-specific delivery adapters. Each adapter must encapsulate the protocol details of its channel (APNS/FCM for push, SMTP for email, Twilio/SMS gateway for SMS, database write for in-app). The system must support sending the same notification through multiple channels simultaneously (e.g., both push and email). Explain how adding a new channel (e.g., WhatsApp) requires only a new adapter without modifying existing code.

**Expected Approach:** Introduce a Notification Service that receives requests via an API, resolves the target channels, and dispatches to channel-specific adapters behind a common interface. Use a message queue between the service and the adapters to decouple ingestion from delivery and enable independent scaling of each channel.

<details>
<summary>Hints</summary>

1. Define a `ChannelAdapter` interface with a single method like `send(recipient, content) -> DeliveryResult`. Each channel (push, email, SMS, in-app) implements this interface with its own protocol logic. The Notification Service never interacts with channel-specific APIs directly — it only calls the adapter interface.
2. Use a dedicated message queue (e.g., Kafka or RabbitMQ) with a separate topic or queue per channel. When a notification request arrives, the service publishes one message per target channel. Channel-specific worker pools consume from their respective queues and invoke the appropriate adapter.
3. For multi-channel delivery, the service simply publishes to multiple queues. Each channel processes independently, so a slow email provider does not block push delivery.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build a centralized Notification Service with a channel-agnostic API, per-channel message queues, and pluggable delivery adapters behind a common interface.

1. **API entry point:**
   ```
   POST /notifications
   {
     "recipient_id": "user_123",
     "type": "payment_received",
     "channels": ["push", "email"],
     "content": {
       "title": "Payment Received",
       "body": "You received $50.00 from Alice.",
       "data": { "payment_id": "pay_abc", "amount": 5000 }
     },
     "priority": "high"
   }
   ```
   - The API validates the request, resolves the recipient's contact details (device tokens, email address, phone number) from a User Profile Service, and persists a notification record to the database with status `pending`.

2. **Channel routing and dispatch:**
   - For each channel in the request's `channels` array, the service publishes a delivery task to the corresponding queue:
     - `push` → `queue:push_notifications`
     - `email` → `queue:email_notifications`
     - `SMS` → `queue:sms_notifications`
     - `in_app` → `queue:inapp_notifications`
   - Each queue has its own consumer pool that scales independently. The push consumer pool might have 20 workers, while the SMS pool has 5 (since SMS is rate-limited by the provider).

3. **Channel adapters (common interface):**
   ```
   interface ChannelAdapter:
       send(recipient: RecipientInfo, content: NotificationContent) -> DeliveryResult

   class PushAdapter implements ChannelAdapter:
       send(recipient, content):
           token = recipient.device_token
           payload = formatForAPNS(content)  // or FCM
           response = apnsClient.send(token, payload)
           return DeliveryResult(success=response.ok, provider_id=response.id)

   class EmailAdapter implements ChannelAdapter:
       send(recipient, content):
           to = recipient.email
           html = renderTemplate(content.type, content.data)
           response = smtpClient.send(to, content.title, html)
           return DeliveryResult(success=response.ok, provider_id=response.message_id)

   class SmsAdapter implements ChannelAdapter:
       send(recipient, content):
           phone = recipient.phone_number
           text = formatPlainText(content)
           response = twilioClient.send(phone, text)
           return DeliveryResult(success=response.ok, provider_id=response.sid)

   class InAppAdapter implements ChannelAdapter:
       send(recipient, content):
           db.insert("in_app_notifications", recipient.user_id, content, read=false)
           return DeliveryResult(success=true)
   ```

4. **Adding a new channel (e.g., WhatsApp):**
   - Create a new `WhatsAppAdapter` implementing `ChannelAdapter`.
   - Create a new queue `queue:whatsapp_notifications` with its own consumer pool.
   - Register the adapter in the channel registry. No changes to the API, the Notification Service, or existing adapters.

5. **Notification record tracking:**
   ```
   Table: notifications
   +-------------------+--------------+----------------------------------------------+
   | Column            | Type         | Notes                                        |
   +-------------------+--------------+----------------------------------------------+
   | notification_id   | UUID         | Primary key                                  |
   | recipient_id      | VARCHAR(64)  | Target user                                  |
   | type              | VARCHAR(64)  | e.g., "payment_received"                     |
   | channels          | JSON         | ["push", "email"]                            |
   | content           | JSON         | Title, body, data payload                    |
   | priority          | ENUM         | low, medium, high, critical                  |
   | created_at        | TIMESTAMP    | When the request was received                |
   +-------------------+--------------+----------------------------------------------+

   Table: delivery_attempts
   +-------------------+--------------+----------------------------------------------+
   | Column            | Type         | Notes                                        |
   +-------------------+--------------+----------------------------------------------+
   | attempt_id        | UUID         | Primary key                                  |
   | notification_id   | UUID         | FK to notifications                          |
   | channel           | VARCHAR(32)  | push, email, sms, in_app                     |
   | status            | ENUM         | pending, sent, delivered, failed              |
   | provider_id       | VARCHAR(128) | ID returned by the external provider         |
   | attempted_at      | TIMESTAMP    | When delivery was attempted                  |
   | error_message     | TEXT         | Nullable; populated on failure               |
   +-------------------+--------------+----------------------------------------------+
   ```

**Why this architecture fits:** The adapter pattern decouples the notification logic from channel-specific protocols, making the system open for extension (new channels) without modification. Per-channel queues allow independent scaling and isolation — a backlog in email delivery does not affect push notification latency. The centralized notification record provides a single audit trail for all channels.

</details>

---

## Problem 2 — Notification Routing and Priority Queuing

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a notification routing and prioritization engine that determines which channels to use for each notification type and ensures high-priority notifications are processed before low-priority ones, even under heavy system load. The engine must support configurable routing rules (e.g., "security alerts go to push + SMS + email; marketing updates go to email only") and a multi-tier priority queue that prevents bulk campaigns from starving time-sensitive transactional notifications.

### Scenario

**Context:** The notification service from Problem 1 is now handling 5 million notifications per day across a large e-commerce platform. The marketing team recently launched a flash sale campaign that sent 2 million promotional emails in one batch. During the campaign, password reset emails (triggered by individual users) were delayed by 15 minutes because they were stuck behind the marketing batch in the same email queue. Meanwhile, fraud alerts that should have been sent via both push and SMS were only sent via email because the calling service hardcoded the channel. The platform needs a routing engine that automatically selects channels based on notification type and a priority system that guarantees transactional notifications are never delayed by bulk sends.

**Requirements:** Design a routing rules engine that maps notification types to channels and priority levels. Rules should be configurable without code changes (e.g., stored in a database or configuration file). Define at least three priority tiers (critical, standard, bulk) with clear processing guarantees for each. Design the queue architecture that enforces priority ordering — critical notifications must be processed within seconds, standard within minutes, and bulk within hours. Explain how the system prevents priority inversion (a flood of bulk notifications blocking critical ones). Handle the case where a notification type is not found in the routing rules (fallback behavior).

**Expected Approach:** Implement a routing rules table that maps each notification type to its channels and priority tier. Use separate queues per priority tier (not per channel) so that high-priority messages are always consumed first. Workers poll the critical queue before the standard queue, and the standard queue before the bulk queue. Rate-limit bulk queue consumption to reserve capacity for higher-priority tiers.

<details>
<summary>Hints</summary>

1. A routing rules table with columns `(notification_type, channels, priority_tier)` allows the system to look up the correct channels and priority for any notification type. For example: `("password_reset", ["email", "push"], "critical")`, `("flash_sale", ["email"], "bulk")`.
2. Use three separate queue groups — `queue:critical:*`, `queue:standard:*`, `queue:bulk:*` — each with per-channel sub-queues. Workers use a weighted polling strategy: check the critical queue first, then standard, then bulk. This ensures critical messages are always processed first.
3. Rate-limit the bulk queue consumers to a fixed throughput (e.g., 1,000 messages/second) regardless of queue depth. This reserves the remaining system capacity for critical and standard notifications that arrive unpredictably.
4. For unknown notification types, define a fallback rule (e.g., default to in-app channel with standard priority) and log a warning so the team can add a proper rule.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build a configurable routing rules engine backed by a database table, combined with a tiered priority queue architecture that uses weighted consumption and rate limiting to enforce processing guarantees.

1. **Routing rules table:**
   ```
   Table: notification_routing_rules
   +---------------------+--------------+----------------------------------------------+
   | Column              | Type         | Notes                                        |
   +---------------------+--------------+----------------------------------------------+
   | notification_type   | VARCHAR(64)  | Primary key; e.g., "password_reset"          |
   | channels            | JSON         | ["push", "email", "sms"]                     |
   | priority_tier       | ENUM         | critical, standard, bulk                     |
   | description         | TEXT         | Human-readable explanation                   |
   | updated_at          | TIMESTAMP    | Last modification time                       |
   +---------------------+--------------+----------------------------------------------+

   Example rows:
   | notification_type     | channels              | priority_tier |
   |-----------------------|-----------------------|---------------|
   | password_reset        | ["email", "push"]     | critical      |
   | fraud_alert           | ["push", "sms", "email"] | critical   |
   | order_shipped         | ["push", "email"]     | standard      |
   | friend_request        | ["push", "in_app"]    | standard      |
   | flash_sale            | ["email"]             | bulk          |
   | weekly_digest         | ["email"]             | bulk          |
   ```
   - Rules are cached in memory with a TTL (e.g., 5 minutes) so that updates take effect without redeployment. A cache invalidation event is published when rules change.

2. **Routing resolution flow:**
   ```
   function resolveRouting(notificationType):
       rule = cache.get(notificationType)
       if rule is null:
           rule = db.query("SELECT * FROM notification_routing_rules WHERE notification_type = ?", notificationType)
           if rule is null:
               log.warn("No routing rule for type: " + notificationType)
               rule = FALLBACK_RULE  // channels: ["in_app"], priority: "standard"
           cache.set(notificationType, rule, ttl=300)
       return rule
   ```

3. **Tiered priority queue architecture:**
   ```
   Critical tier:
       queue:critical:push
       queue:critical:email
       queue:critical:sms
       queue:critical:in_app

   Standard tier:
       queue:standard:push
       queue:standard:email
       queue:standard:sms
       queue:standard:in_app

   Bulk tier:
       queue:bulk:push
       queue:bulk:email
       queue:bulk:sms
       queue:bulk:in_app
   ```
   - When a notification arrives, the routing engine resolves its priority tier and channels, then publishes to the appropriate tier+channel queue (e.g., a critical push notification goes to `queue:critical:push`).

4. **Weighted consumption strategy:**
   - Each channel has a pool of workers. Workers use a strict priority polling order:
     ```
     function workerLoop(channel):
         while true:
             msg = dequeue("queue:critical:" + channel)  // check critical first
             if msg is null:
                 msg = dequeue("queue:standard:" + channel)  // then standard
             if msg is null:
                 msg = dequeue("queue:bulk:" + channel, rateLimit=true)  // then bulk, rate-limited
             if msg is not null:
                 adapter = getAdapter(channel)
                 result = adapter.send(msg.recipient, msg.content)
                 updateDeliveryStatus(msg.notification_id, channel, result)
             else:
                 sleep(100ms)  // all queues empty
     ```
   - Critical messages are always dequeued first. A flood of bulk messages cannot block critical or standard messages because workers always check higher-priority queues first.

5. **Rate limiting for bulk tier:**
   - Bulk queue consumers are rate-limited to a configurable throughput (e.g., 1,000 emails/second for the email channel). This ensures that even a 2-million-message campaign is spread over ~33 minutes, leaving ample capacity for critical and standard notifications.
   - The rate limit is enforced via a token bucket or sliding window counter shared across bulk workers.

6. **Processing guarantees by tier:**
   | Tier     | Target Latency | Retry Policy                | Use Cases                          |
   |----------|---------------|-----------------------------|------------------------------------|
   | Critical | < 30 seconds  | 3 retries, 5s backoff       | Password reset, fraud alert, OTP   |
   | Standard | < 5 minutes   | 3 retries, 30s backoff      | Order updates, friend requests     |
   | Bulk     | < 4 hours     | 2 retries, 5min backoff     | Marketing campaigns, digests       |

7. **Preventing priority inversion:**
   - Separate queues per tier ensure physical isolation — bulk messages never share a queue with critical messages.
   - Worker polling order (critical → standard → bulk) ensures that even if all workers are busy with bulk messages, they will preempt bulk processing as soon as a critical message arrives (after the current message completes).
   - Auto-scaling: if the critical queue depth exceeds a threshold, spin up additional workers dedicated exclusively to the critical tier.

**Why tiered priority queues fit:** A single shared queue with priority fields still risks head-of-line blocking when bulk messages flood the queue. Physically separate queues with strict polling order guarantee that critical notifications are never delayed by lower-priority traffic, regardless of volume.

</details>

---

## Problem 3 — User Notification Preference Management

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Design a user notification preference system that allows users to control which notifications they receive, through which channels, and when. The system must support granular per-category opt-in/opt-out, per-channel preferences, quiet hours (do-not-disturb schedules), and frequency caps — all while respecting regulatory requirements that mandate certain transactional notifications cannot be suppressed. The preference evaluation must be fast enough to be checked inline during notification processing without adding significant latency.

### Scenario

**Context:** The e-commerce platform's notification system sends 20 notification types across four channels. Users are complaining about notification fatigue — they receive too many push notifications for promotions they do not care about, and some users in different time zones are woken up by non-urgent alerts at 3 AM. The product team wants to build a notification preferences page where users can: (1) toggle individual notification categories on/off per channel (e.g., "Send me order updates via push but not email"), (2) set quiet hours during which only critical notifications are delivered, and (3) set a maximum number of push notifications per day. At the same time, the legal team requires that certain notifications (password resets, account security alerts, legal notices) must always be delivered regardless of user preferences. The system must evaluate all these rules in under 10 ms per notification to avoid adding latency to the delivery pipeline.

**Requirements:** Design the data model for storing user notification preferences, including per-category per-channel toggles, quiet hours with timezone support, and frequency caps. Define the preference evaluation algorithm that determines whether a given notification should be delivered to a given user through a given channel. Specify which notification types are "mandatory" and bypass preference checks entirely. Handle the case where a user has not configured any preferences (default behavior). Design the preference evaluation to be cacheable and fast (sub-10 ms). Explain how preference changes propagate to the evaluation layer without requiring a service restart.

**Expected Approach:** Store preferences in a normalized database with a denormalized cache layer (Redis hash per user). The evaluation algorithm checks in order: (1) is the notification mandatory? If yes, deliver. (2) Has the user opted out of this category on this channel? (3) Is the current time within the user's quiet hours for this channel? (4) Has the user exceeded their frequency cap for this channel today? Cache the full preference set per user in Redis for sub-millisecond lookups, with cache invalidation on preference updates.
