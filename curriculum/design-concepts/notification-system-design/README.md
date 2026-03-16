# Notification System Design

**Track:** Design Concepts
**Difficulty Tier:** Advanced
**Prerequisites:** [Message Queues](../message-queues/README.md), [Microservices Architecture](../microservices-architecture/README.md), [Chat System Design](../chat-system-design/README.md)

## Concept Overview

A notification system is the backbone of user engagement for any modern application. Whether it is a social media platform alerting a user about a new follower, a banking app confirming a transaction, or a ride-sharing service announcing that a driver has arrived, notifications bridge the gap between backend events and user awareness. At scale, a notification platform must ingest events from dozens of internal services, determine who should be notified, select the appropriate delivery channel (push notification, SMS, email, or in-app), render the message from a template, and dispatch it — all within seconds of the triggering event.

Designing a notification system at scale introduces challenges that span nearly every area of distributed systems. The ingestion layer must handle bursty traffic — a flash sale announcement might generate millions of notifications in seconds. The routing layer must respect user preferences (opt-outs, quiet hours, channel priorities) and deduplicate events so users are not bombarded with repeated alerts. The delivery layer must integrate with external providers (APNs, FCM, Twilio, SendGrid) that each have their own rate limits, retry semantics, and failure modes. Observability is critical: the team must be able to trace any notification from the triggering event through template rendering, channel selection, and final delivery status.

Beyond the happy path, a production notification system must handle failures gracefully. Push notification tokens expire when users reinstall an app. Email providers throttle senders who exceed rate limits. SMS delivery to certain regions is unreliable. The system must retry transient failures, circuit-break on persistent ones, and provide fallback channels when the primary channel fails. Features like notification grouping (batching multiple alerts into a single digest), priority levels (a security alert must never be delayed), and cross-channel coordination (do not send both a push and an email for the same event) add further architectural complexity.

---

## Problem 1 — Core Notification Pipeline Architecture

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Design the high-level architecture for a notification system that accepts events from multiple internal services, routes them through a processing pipeline, and delivers notifications to users via push notifications, email, and SMS. Focus on the core flow from event ingestion to delivery, including how the system decouples producers (services that trigger notifications) from the delivery infrastructure.

### Scenario

**Context:** An e-commerce company has a dozen backend services (order service, payment service, shipping service, marketing service, etc.) that each need to notify users about various events — order confirmations, payment receipts, shipment tracking updates, promotional offers. Currently, each service implements its own notification logic, leading to inconsistent formatting, duplicated integrations with email and SMS providers, and no centralized control over user preferences. The company wants to build a unified notification platform that all services publish events to, and the platform handles the rest. The system serves 5 million active users and sends approximately 20 million notifications per day.

**Requirements:** Design the event ingestion interface that producer services use to submit notification requests. Define the processing pipeline stages from event receipt to delivery. Choose an appropriate message queue architecture to decouple ingestion from processing. Describe how the system selects the delivery channel (push, email, SMS) for each notification. Explain how the system integrates with external delivery providers (APNs for iOS push, FCM for Android push, an email service, and an SMS gateway).

**Expected Approach:** Use a message queue (e.g., Kafka or RabbitMQ) as the central decoupling layer. Producer services publish structured notification events to the queue. A pipeline of consumer workers processes each event through stages: validation, user preference lookup, template rendering, channel selection, and dispatch to the appropriate delivery provider. Each delivery provider integration is encapsulated behind an adapter interface.

<details>
<summary>Hints</summary>

1. A message queue between producers and the notification pipeline ensures that a spike in events (e.g., a flash sale triggering millions of order confirmations) does not overwhelm the processing workers. The queue absorbs the burst and workers consume at a sustainable rate.
2. Structure the notification event with fields like `event_type`, `user_id`, `payload` (dynamic data for template rendering), and `priority`. This keeps the producer interface simple — producers do not need to know about channels, templates, or user preferences.
3. The channel selection logic should consult a user preferences table that stores each user's opted-in channels, quiet hours, and per-category opt-outs. A default channel priority (push > email > SMS) applies when the user has not set explicit preferences.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build an event-driven pipeline with a message queue for decoupling, a multi-stage processing flow, and adapter-based integrations with external delivery providers.

1. **Event ingestion interface:**
   - Producer services publish notification events to a Kafka topic (`notification-events`) via a lightweight SDK or HTTP API.
   - Event schema:
     ```
     {
       "event_id": "uuid",
       "event_type": "order_confirmed",
       "user_id": "user_123",
       "payload": { "order_id": "ORD-456", "total": "$79.99", "items": 3 },
       "priority": "normal",
       "timestamp": "2024-01-15T10:30:00Z"
     }
     ```
   - The SDK validates the event schema before publishing, rejecting malformed events at the source.

2. **Processing pipeline stages:**
   ```
   Kafka topic → [Validation] → [Preference Lookup] → [Template Rendering] → [Channel Selection] → [Dispatch Queue(s)] → [Delivery Workers]
   ```
   - **Validation:** Verify the event has all required fields, the `event_type` is recognized, and the `user_id` exists.
   - **Preference lookup:** Query the user preferences service to determine: (a) whether the user has opted out of this notification category, (b) which channels the user has enabled, (c) whether the user is in a quiet-hours window.
   - **Template rendering:** Look up the template for the `event_type` and render it with the `payload` data. Each channel may have a different template (e.g., a rich HTML email vs. a short SMS text vs. a push notification title + body).
   - **Channel selection:** Based on user preferences and channel availability (e.g., does the user have a registered push token?), select one or more channels. Default priority: push > email > SMS.
   - **Dispatch:** Route the rendered notification to channel-specific queues (`push-queue`, `email-queue`, `sms-queue`).

3. **Delivery workers and provider adapters:**
   - Each channel has a pool of delivery workers consuming from its queue.
   - Workers use an adapter interface to call the external provider:
     ```
     interface DeliveryAdapter:
         send(recipient, message) → DeliveryResult
     ```
   - Adapters: `APNsAdapter` (iOS push), `FCMAdapter` (Android push), `SendGridAdapter` (email), `TwilioAdapter` (SMS).
   - Each adapter handles provider-specific authentication, payload formatting, and response parsing.
   - On success, the worker logs the delivery status. On transient failure (timeout, 5xx), the message is retried with exponential backoff. On permanent failure (invalid token, unsubscribed), the worker updates the user's channel registration.

4. **Data flow example (order confirmation):**
   ```
   Order Service publishes: { event_type: "order_confirmed", user_id: "user_123", payload: {...} }
   → Kafka → Validation worker: event is valid
   → Preference lookup: user_123 has push enabled, email enabled, SMS disabled; not in quiet hours
   → Template rendering: push title = "Order Confirmed!", push body = "Your order ORD-456 ($79.99) is confirmed."
                          email subject = "Order Confirmation — ORD-456", email body = rendered HTML
   → Channel selection: push (primary), email (secondary)
   → Dispatch: push message → push-queue, email message → email-queue
   → Push worker: sends via FCM (user is on Android) → delivered
   → Email worker: sends via SendGrid → delivered
   ```

5. **Why Kafka for ingestion:**
   - Kafka handles high-throughput, bursty writes (millions of events during a flash sale) with durable, partitioned storage.
   - Consumer groups allow horizontal scaling of processing workers.
   - Message retention enables replay if a processing bug is discovered (reprocess events from a past offset).

</details>

---

## Problem 2 — User Preference Management and Channel Routing

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design the user preference and channel routing subsystem for a notification platform. The system must allow users to control which notification categories they receive, which channels (push, email, SMS, in-app) are used for each category, and when notifications are delivered (quiet hours). The routing engine must evaluate these preferences in real time for every notification and handle conflicts such as a user disabling push but enabling email, or a high-priority security alert that must bypass quiet hours.

### Scenario

**Context:** The e-commerce notification platform from Problem 1 is live, but users are complaining about notification overload. Some users receive both a push notification and an email for every order update. Others want promotional notifications only via email, not push. Several users in different time zones are woken up by late-night push notifications. The product team wants to give users granular control: per-category channel preferences, global quiet hours, and the ability to unsubscribe from specific categories entirely. At the same time, certain notifications (password reset, fraud alert) must always be delivered regardless of user preferences — they are classified as "mandatory." The system serves 5 million users, each with potentially unique preference configurations, and the routing engine must evaluate preferences for 20 million notifications per day without adding significant latency.

**Requirements:** Design the data model for user notification preferences, including per-category channel selections, global quiet hours (with time zone support), and category-level opt-outs. Define the routing algorithm that evaluates preferences for each notification and determines which channels to use. Specify how mandatory notifications bypass preference checks. Handle the case where a user's preferred channel is unavailable (e.g., no push token registered) by falling back to the next preferred channel. Ensure the preference lookup is fast enough to not bottleneck the pipeline (target: under 10 ms per lookup).

**Expected Approach:** Store preferences in a structured format per user with category-level overrides. Cache frequently accessed preferences in Redis for low-latency lookups. Implement a routing algorithm that checks: (1) is the category opted out? (2) is it quiet hours? (3) which channels are enabled for this category? (4) is the channel available? Mandatory notifications skip steps 1 and 2.

<details>
<summary>Hints</summary>

1. Model preferences hierarchically: global defaults → category-level overrides → mandatory flags. A user might have global defaults (push + email for everything) but override a specific category (promotions → email only). Mandatory notifications ignore both levels.
2. Quiet hours should be stored as a start time, end time, and time zone (e.g., 22:00–07:00 America/New_York). The routing engine converts the current UTC time to the user's time zone before checking.
3. Cache the full preference object for each user in Redis with a TTL of 5 minutes. At 5 million users × ~500 bytes per preference object, the cache requires ~2.5 GB — easily fits in a single Redis instance.
4. Channel availability is determined by checking whether the user has a valid push token (for push), a verified email address (for email), or a verified phone number (for SMS). This data can be stored alongside preferences or in a separate user profile.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use a hierarchical preference model with Redis caching and a multi-step routing algorithm that respects opt-outs, quiet hours, channel availability, and mandatory overrides.

1. **Preference data model:**
   ```
   Table: user_notification_preferences
   +---------------------+--------------+------------------------------------------------+
   | Column              | Type         | Notes                                          |
   +---------------------+--------------+------------------------------------------------+
   | user_id             | VARCHAR(64)  | Primary key                                    |
   | global_channels     | JSON         | Default channels: ["push", "email"]            |
   | category_overrides  | JSON         | Per-category: {"promotions": {"channels": ["email"], "opted_out": false}} |
   | quiet_hours_start   | TIME         | e.g., 22:00                                    |
   | quiet_hours_end     | TIME         | e.g., 07:00                                    |
   | timezone            | VARCHAR(64)  | e.g., America/New_York                         |
   | updated_at          | TIMESTAMP    | For cache invalidation                         |
   +---------------------+--------------+------------------------------------------------+

   Table: user_channel_registrations
   +---------------------+--------------+------------------------------------------------+
   | Column              | Type         | Notes                                          |
   +---------------------+--------------+------------------------------------------------+
   | user_id             | VARCHAR(64)  | FK to users                                    |
   | channel             | ENUM         | 'push_ios', 'push_android', 'email', 'sms'    |
   | token_or_address    | VARCHAR(512) | Push token, email address, or phone number     |
   | verified            | BOOLEAN      | Whether the address/token is confirmed valid   |
   | updated_at          | TIMESTAMP    | Last update time                               |
   +---------------------+--------------+------------------------------------------------+
   Primary key: (user_id, channel)
   ```

2. **Routing algorithm:**
   ```
   function routeNotification(event, userPrefs, channelRegistrations):
       category = getCategoryForEventType(event.event_type)
       isMandatory = category in MANDATORY_CATEGORIES  // e.g., security_alert, password_reset

       // Step 1: Check category opt-out (skip for mandatory)
       if not isMandatory:
           if userPrefs.category_overrides[category].opted_out:
               return SUPPRESSED("user opted out of category")

       // Step 2: Check quiet hours (skip for mandatory)
       if not isMandatory:
           userLocalTime = convertToTimezone(now(), userPrefs.timezone)
           if isInQuietHours(userLocalTime, userPrefs.quiet_hours_start, userPrefs.quiet_hours_end):
               return DEFERRED(deliverAt = nextQuietHoursEnd(userPrefs))

       // Step 3: Determine channels
       channels = userPrefs.category_overrides[category].channels
                  or userPrefs.global_channels
                  or DEFAULT_CHANNELS  // ["push", "email"]

       // Step 4: Filter by channel availability
       availableChannels = []
       for ch in channels:
           reg = channelRegistrations.get(ch)
           if reg and reg.verified:
               availableChannels.append(ch)

       if not availableChannels:
           return SUPPRESSED("no available channels")

       // Step 5: For mandatory, force at least one channel
       if isMandatory and not availableChannels:
           availableChannels = getAllVerifiedChannels(channelRegistrations)

       return DELIVER(channels = availableChannels)
   ```

3. **Quiet hours with deferred delivery:**
   - When a non-mandatory notification arrives during quiet hours, the routing engine does not discard it. Instead, it schedules delivery for the end of the quiet-hours window.
   - A deferred-delivery queue (e.g., a Kafka topic with a scheduled consumer or a database table with a `deliver_at` timestamp) holds these notifications. A scheduler worker polls for notifications whose `deliver_at` has passed and re-injects them into the pipeline.

4. **Caching strategy:**
   - On first access, load the user's preferences and channel registrations from the database and cache them in Redis as a single JSON blob with a 5-minute TTL.
   - When a user updates their preferences via the settings UI, invalidate the cache entry immediately (write-through).
   - Cache key: `notif_prefs:{user_id}`.
   - At 20 million notifications/day ≈ 231 notifications/second. With a 5-minute TTL and 5 million users, the cache hit rate will be high for active users, keeping database load minimal.

5. **Mandatory notification categories:**
   - Defined in a configuration file or database table: `security_alert`, `password_reset`, `account_locked`, `fraud_detected`, `legal_notice`.
   - These bypass opt-out checks and quiet hours. They are delivered on all available channels to maximize the chance of reaching the user.

6. **Channel fallback logic:**
   - If the user prefers push but has no valid push token (e.g., uninstalled the app), fall back to email. If no email is verified, fall back to SMS.
   - Fallback order is configurable per category. For transactional notifications (order updates), the fallback is push → email → SMS. For promotional notifications, the fallback is push → email (no SMS to avoid cost).

</details>

---

## Problem 3 — Reliable Delivery with Retries and Fallback Channels

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design the delivery reliability layer for a notification system that guarantees every notification reaches the user through at least one channel, even when external providers fail. The system must handle transient failures (provider timeouts, rate limiting), permanent failures (invalid push tokens, bounced emails), and prolonged outages of an entire provider. It must implement retry strategies with backoff, circuit breakers to avoid hammering failing providers, and cross-channel fallback when the primary channel is exhausted.

### Scenario

**Context:** The notification platform delivers 20 million notifications per day across push (APNs and FCM), email (SendGrid), and SMS (Twilio). On a typical day, approximately 2% of push deliveries fail due to expired tokens, 0.5% of emails bounce, and SMS delivery to certain international regions has a 5% failure rate. During a recent incident, the FCM API experienced a 30-minute outage, causing 150,000 push notifications to fail. The team's current retry logic simply retries three times with a fixed 5-second delay, which overwhelmed FCM when it recovered (thundering herd) and did not fall back to email for users who missed the push. The team needs a robust delivery layer that handles all failure modes, avoids thundering herds, and ensures critical notifications (e.g., two-factor authentication codes) are delivered within a strict time window even if the primary channel is down.

**Requirements:** Design the retry strategy for each delivery channel, including backoff schedules and maximum retry counts. Implement a circuit breaker pattern that detects when a provider is experiencing an outage and stops sending requests to it temporarily. Define the cross-channel fallback mechanism — when retries on the primary channel are exhausted or the circuit breaker is open, the system should attempt delivery on the next available channel. Handle permanent failures (invalid tokens, hard bounces) by updating the user's channel registration so future notifications do not attempt the failed channel. Design a priority-aware delivery path where high-priority notifications (2FA codes, fraud alerts) skip the normal queue and use aggressive retry and immediate fallback. Ensure the system avoids duplicate delivery when a notification is retried or falls back to another channel.

**Expected Approach:** Use exponential backoff with jitter for retries. Implement a circuit breaker per provider that opens after a threshold of consecutive failures and closes after a cooldown period with a half-open probe. On primary channel exhaustion, re-enqueue the notification to the fallback channel's queue. Track delivery state per notification in a database to prevent duplicates and enable end-to-end tracing.

<details>
<summary>Hints</summary>

1. Exponential backoff with jitter: delay = min(base × 2^attempt + random_jitter, max_delay). For push notifications, use base = 1 second, max_delay = 60 seconds, max_attempts = 5. The jitter (random 0–1 second) prevents thundering herd when many retries fire simultaneously after a provider recovers.
2. A circuit breaker has three states: Closed (normal operation), Open (all requests fail-fast without calling the provider), and Half-Open (allow a single probe request to test if the provider has recovered). Transition to Open after 10 consecutive failures or a 50% failure rate in a 1-minute window. Transition to Half-Open after a 30-second cooldown. If the probe succeeds, close the circuit; if it fails, reopen.
3. Track each notification's delivery state in a `notification_deliveries` table with columns: `notification_id`, `channel`, `attempt`, `status` (pending, delivered, failed_transient, failed_permanent, fallback_triggered). This prevents duplicate delivery — before sending, check if the notification has already been delivered on any channel.
4. For high-priority notifications (2FA codes), use a separate fast-path queue with higher consumer concurrency, shorter retry intervals (500 ms base), and immediate fallback after 2 failed attempts (instead of 5).

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement per-provider circuit breakers, exponential backoff with jitter for retries, a stateful delivery tracker for deduplication, and a priority-aware dual-path architecture with cross-channel fallback.

1. **Retry strategy per channel:**
   ```
   Push (APNs/FCM):
       base_delay = 1s, max_delay = 60s, max_attempts = 5
       delay(n) = min(1 × 2^n + random(0, 1), 60) seconds
       Permanent failures: invalid token → remove token from user_channel_registrations

   Email (SendGrid):
       base_delay = 5s, max_delay = 300s, max_attempts = 4
       delay(n) = min(5 × 2^n + random(0, 5), 300) seconds
       Permanent failures: hard bounce → mark email as invalid

   SMS (Twilio):
       base_delay = 2s, max_delay = 120s, max_attempts = 3
       delay(n) = min(2 × 2^n + random(0, 2), 120) seconds
       Permanent failures: invalid number → mark phone as invalid
   ```

2. **Circuit breaker implementation:**
   ```
   class CircuitBreaker:
       state = CLOSED
       failure_count = 0
       failure_threshold = 10
       failure_rate_threshold = 0.50
       cooldown_period = 30 seconds
       last_failure_time = null

       function call(provider, request):
           if state == OPEN:
               if now() - last_failure_time > cooldown_period:
                   state = HALF_OPEN
               else:
                   raise CircuitOpenError  // fail fast

           if state == HALF_OPEN:
               result = provider.send(request)
               if result.success:
                   state = CLOSED; failure_count = 0
               else:
                   state = OPEN; last_failure_time = now()
               return result

           // state == CLOSED
           result = provider.send(request)
           if result.failure:
               failure_count += 1
               if failure_count >= failure_threshold:
                   state = OPEN; last_failure_time = now()
           else:
               failure_count = 0
           return result
   ```
   - One circuit breaker instance per provider (APNs, FCM, SendGrid, Twilio).
   - When the circuit is open, the delivery worker immediately triggers fallback instead of waiting for retries to exhaust.

3. **Cross-channel fallback mechanism:**
   ```
   function deliverWithFallback(notification, channelPriority):
       for channel in channelPriority:
           if deliveryTracker.isDelivered(notification.id):
               return  // already delivered on a previous channel

           if circuitBreaker[channel].isOpen():
               continue  // skip this channel, try next

           result = attemptDeliveryWithRetries(notification, channel)
           if result == DELIVERED:
               deliveryTracker.markDelivered(notification.id, channel)
               return
           if result == PERMANENT_FAILURE:
               updateChannelRegistration(notification.user_id, channel, invalid=true)
               continue  // try next channel
           if result == RETRIES_EXHAUSTED:
               continue  // try next channel

       // All channels exhausted
       deliveryTracker.markFailed(notification.id, "all_channels_exhausted")
       alertOpsTeam(notification)
   ```

4. **Delivery state tracking (deduplication):**
   ```
   Table: notification_deliveries
   +---------------------+--------------+------------------------------------------------+
   | Column              | Type         | Notes                                          |
   +---------------------+--------------+------------------------------------------------+
   | notification_id     | UUID         | FK to notifications                            |
   | channel             | ENUM         | 'push_ios', 'push_android', 'email', 'sms'    |
   | attempt             | INT          | Retry attempt number (1, 2, 3, ...)            |
   | status              | ENUM         | 'pending', 'delivered', 'failed_transient',    |
   |                     |              | 'failed_permanent', 'fallback_triggered'       |
   | provider_response   | TEXT         | Raw response from the provider                 |
   | created_at          | TIMESTAMP    | When this attempt was made                     |
   +---------------------+--------------+------------------------------------------------+
   Index: (notification_id, channel) for deduplication checks
   ```
   - Before each delivery attempt, the worker checks: `SELECT status FROM notification_deliveries WHERE notification_id = ? AND status = 'delivered'`. If a row exists, skip delivery (another channel already succeeded).

5. **Priority-aware dual-path architecture:**
   - **Normal path:** Notifications flow through the standard Kafka topic → processing pipeline → channel-specific queues. Retry delays are standard (seconds to minutes).
   - **Fast path:** High-priority notifications (2FA codes, fraud alerts, security warnings) are routed to a dedicated high-priority Kafka topic with separate consumer groups that have higher concurrency. Retry parameters are aggressive: base_delay = 500 ms, max_attempts = 2. Fallback triggers after just 2 failures instead of 5. If push fails twice, immediately attempt SMS (for 2FA codes) or email.
   - Priority is determined by the `event_type` → category mapping. Categories like `two_factor_auth` and `fraud_alert` are flagged as high-priority in the configuration.

6. **Thundering herd prevention:**
   - The jitter component in the backoff formula ensures that retries from different notifications do not all fire at the same instant when a provider recovers.
   - The circuit breaker's half-open state allows only one probe request through, preventing a flood of retries from hitting the recovering provider simultaneously.
   - When the circuit closes (provider recovered), the delivery workers resume consuming from the queue at their normal rate, naturally spreading the load.

</details>

---

## Problem 4 — Notification Deduplication and Intelligent Grouping

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design a deduplication and grouping subsystem for a notification platform that prevents users from receiving redundant notifications and intelligently batches related notifications into digests. The system must detect duplicate events from upstream services, suppress redundant notifications when multiple events map to the same user-visible alert, and group rapid-fire notifications of the same type into a single summary notification rather than bombarding the user with individual alerts.

### Scenario

**Context:** A social media platform's notification system sends alerts for likes, comments, follows, and mentions. During peak hours, a popular post can receive 500 likes in 10 minutes. Without grouping, the post author receives 500 individual push notifications ("User A liked your post", "User B liked your post", …), which is an unacceptable user experience. Additionally, the upstream "likes" service occasionally publishes duplicate events due to at-least-once delivery semantics in its message queue — the same like event is published twice, resulting in two identical notifications. The team also discovered that when a user edits a comment, the comment service publishes both an "edited" and an "updated" event, causing two notifications for the same action. The platform needs: (1) event-level deduplication to suppress exact duplicate events, (2) semantic deduplication to recognize that two different events represent the same user action, and (3) intelligent grouping to batch rapid-fire notifications into digests like "Alice and 47 others liked your post."

**Requirements:** Design the event-level deduplication mechanism using idempotency keys. Design the semantic deduplication layer that identifies when different events represent the same logical notification. Design the grouping engine that accumulates notifications of the same type within a configurable time window and produces a single digest notification. Define the data structures for tracking active grouping windows. Handle edge cases: what happens when a grouped notification is partially delivered and new events arrive? How does the system handle a mix of groupable and non-groupable notification types? Ensure the grouping delay does not exceed a configurable maximum (e.g., 5 minutes) so users are not waiting too long for their digest.

**Expected Approach:** Use a deduplication cache (Redis) with event IDs as keys and a short TTL for exact deduplication. For semantic deduplication, define a "notification key" that maps different events to the same logical notification (e.g., both "comment_edited" and "comment_updated" for the same comment ID produce the same notification key). For grouping, use a time-windowed accumulator per (user, notification_type) that collects events for a configurable window, then flushes a single digest notification.

<details>
<summary>Hints</summary>

1. For event-level deduplication, each event should carry an `event_id` (UUID). Before processing, check if `event_id` exists in a Redis set with a TTL of 10 minutes. If it exists, skip the event. If not, add it and proceed. This handles at-least-once delivery duplicates from upstream services.
2. For semantic deduplication, define a `notification_key` derived from the event's semantic meaning: e.g., `like:{post_id}:{liker_user_id}` or `comment_update:{comment_id}`. If two events produce the same `notification_key`, only the first is processed. Store notification keys in Redis with a TTL matching the grouping window.
3. For grouping, maintain a per-user accumulator in Redis: key = `group:{user_id}:{notification_type}:{target_id}`, value = a list of contributing events. A timer (or scheduled task) flushes the accumulator after the window expires (e.g., 5 minutes) and produces a single digest notification.
4. The grouping window should have both a time limit (max 5 minutes) and a count threshold (e.g., flush immediately if 100 events accumulate before the window expires). This ensures that extremely popular posts do not wait the full window before the user is notified.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement a three-layer deduplication and grouping pipeline: exact deduplication via event ID cache, semantic deduplication via notification keys, and time-windowed accumulation with count-based early flush for digest generation.

1. **Layer 1 — Exact event deduplication:**
   - Every event published by upstream services includes a unique `event_id`.
   - The first stage of the notification pipeline checks Redis: `SETNX dedup:{event_id} 1 EX 600` (10-minute TTL).
   - If the key already exists (SETNX returns 0), the event is a duplicate — discard it.
   - If the key is new (SETNX returns 1), proceed to the next stage.
   - This catches at-least-once delivery duplicates from Kafka or other message queues.

2. **Layer 2 — Semantic deduplication:**
   - Define a `notification_key` function per event type:
     ```
     function notificationKey(event):
         switch event.event_type:
             case "comment_edited":
             case "comment_updated":
                 return "comment_change:{event.payload.comment_id}"
             case "like":
                 return "like:{event.payload.post_id}:{event.payload.liker_id}"
             case "follow":
                 return "follow:{event.payload.follower_id}:{event.payload.followee_id}"
             default:
                 return "{event.event_type}:{event.event_id}"  // non-groupable, unique
     ```
   - Check Redis: `SETNX semantic:{notification_key} 1 EX 300` (5-minute TTL matching the grouping window).
   - If the key exists, this is a semantically duplicate notification — discard it.
   - This prevents "comment_edited" and "comment_updated" for the same comment from generating two notifications.

3. **Layer 3 — Grouping engine:**
   - Groupable notification types are configured: `like`, `comment`, `follow`, `repost`.
   - For each groupable event, the engine adds it to an accumulator:
     ```
     Redis key: group:{recipient_user_id}:{notification_type}:{target_id}
     Redis type: Sorted Set (score = event timestamp, member = event JSON)
     ```
   - Example: When user_456's post receives likes, each like event is added to `group:user_456:like:post_789`.
   - **Flush triggers:**
     - **Time-based:** A scheduled worker scans for accumulators older than the window duration (e.g., 5 minutes since the first event in the group). It flushes the accumulator and produces a digest notification.
     - **Count-based:** After adding an event, if the accumulator size exceeds a threshold (e.g., 50 events), flush immediately. This ensures viral posts do not wait the full window.
     - **First-event fast notification:** When the first event arrives in an empty accumulator, deliver it immediately as a normal notification ("Alice liked your post"). Subsequent events within the window are accumulated. At flush, a digest replaces the initial notification ("Alice and 47 others liked your post").

4. **Digest notification rendering:**
   ```
   function renderDigest(accumulatorKey):
       events = redis.zrange(accumulatorKey, 0, -1)
       count = len(events)
       firstActor = events[0].payload.actor_name
       if count == 1:
           return "{firstActor} liked your post"
       elif count == 2:
           secondActor = events[1].payload.actor_name
           return "{firstActor} and {secondActor} liked your post"
       else:
           return "{firstActor} and {count - 1} others liked your post"
   ```

5. **Edge cases:**
   - **Partial delivery + new events:** If a digest for "48 likes" was already delivered and 10 more likes arrive, a new grouping window starts. The next digest says "User X and 9 others liked your post" (only the new likes, not a cumulative count). The in-app notification center shows the cumulative count by aggregating all digest entries.
   - **Non-groupable notifications:** Events like `order_confirmed` or `password_reset` bypass the grouping engine entirely and are delivered immediately. The grouping engine checks a configuration map to determine if an event type is groupable.
   - **Mixed priority in a group:** If a groupable event is also high-priority (rare), it is delivered immediately and also added to the accumulator. The digest at flush time excludes already-delivered events to avoid duplication.

6. **Data cleanup:**
   - Accumulator keys in Redis have a TTL of 2× the window duration (e.g., 10 minutes for a 5-minute window) as a safety net. The flush worker deletes the key after producing the digest.
   - Deduplication keys (`dedup:*` and `semantic:*`) expire automatically via their TTLs.

</details>

---

## Problem 5 — Scalable Notification Delivery with Rate Limiting and Observability

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design the scaling strategy, rate limiting layer, and observability infrastructure for a notification platform that must handle 100 million notifications per day with strict SLAs: 99.9% of normal-priority notifications delivered within 5 minutes of the triggering event, and 99.99% of high-priority notifications delivered within 30 seconds. The system must enforce per-provider rate limits to avoid being throttled or banned, implement per-user rate limits to prevent notification spam, and provide end-to-end tracing so that any notification can be traced from the triggering event through every pipeline stage to final delivery status.

### Scenario

**Context:** The notification platform has grown from 20 million to 100 million notifications per day as the company expanded internationally. The system now integrates with six delivery providers: APNs, FCM, two regional email providers, and two SMS gateways. Each provider has different rate limits — FCM allows 1,000 requests per second per project, one of the SMS gateways limits to 100 messages per second, and the email providers have hourly sending quotas. During a recent marketing campaign, the system attempted to send 5 million promotional emails in 30 minutes, exceeding the email provider's hourly quota and causing all email delivery (including transactional emails like password resets) to be blocked for an hour. Separately, the operations team has no visibility into notification latency — when a user reports "I never received my 2FA code," the team cannot determine whether the notification was generated, which pipeline stage it reached, or why delivery failed. The team needs: (1) a scaling architecture that handles 100M notifications/day with headroom for 3× spikes, (2) per-provider rate limiting that prevents quota exhaustion and prioritizes transactional over promotional traffic, and (3) end-to-end observability with distributed tracing.

**Requirements:** Design the horizontal scaling strategy for each pipeline stage (ingestion, processing, delivery). Implement per-provider rate limiting that enforces the provider's rate limits and prioritizes high-priority notifications when the rate limit is approached. Implement per-user rate limiting that caps the number of notifications a user receives per hour to prevent spam. Design the end-to-end tracing system that assigns a trace ID to each notification and records timestamps, outcomes, and metadata at every pipeline stage. Define the key metrics and dashboards the operations team needs (delivery latency percentiles, failure rates by provider, queue depths, circuit breaker states). Handle the scenario where a provider's rate limit is reached — how are excess notifications queued, and how does the system ensure high-priority notifications are not starved by a backlog of promotional notifications.

**Expected Approach:** Partition pipeline stages by notification priority (separate queues for high and normal priority). Use token bucket rate limiters per provider, with separate buckets for transactional and promotional traffic. Implement distributed tracing by propagating a trace ID through all pipeline stages and logging structured events to a centralized tracing system. Scale horizontally by adding consumer instances per Kafka partition.

<details>
<summary>Hints</summary>

1. At 100 million notifications/day, the average throughput is ~1,157 notifications/second. With 3× spike headroom, the system must handle ~3,500/second. Kafka with 50–100 partitions per topic and a proportional number of consumer instances can handle this throughput.
2. Use a token bucket rate limiter per provider: the bucket refills at the provider's allowed rate (e.g., 1,000 tokens/second for FCM). When a delivery worker picks up a notification, it must acquire a token before calling the provider. If no token is available, the notification is re-queued with a short delay. Maintain separate buckets for transactional and promotional traffic, with transactional traffic getting 80% of the token allocation.
3. For per-user rate limiting, use a sliding window counter in Redis: key = `user_rate:{user_id}:{hour}`, increment on each notification sent, and check against a configurable limit (e.g., 50 notifications per hour). Mandatory notifications bypass this limit.
4. For distributed tracing, assign a `trace_id` (UUID) when the notification event is first ingested. Each pipeline stage appends a span to the trace: `{stage, timestamp, duration, outcome, metadata}`. Store traces in a time-series database or a tracing backend (e.g., Jaeger or a custom solution). The operations team can search by `trace_id`, `user_id`, or `event_type` to debug delivery issues.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement priority-partitioned Kafka topics, token-bucket rate limiters with transactional/promotional split, sliding-window per-user rate limits, and a structured distributed tracing pipeline with operational dashboards.

1. **Horizontal scaling architecture:**
   ```
   Ingestion Layer:
       - Kafka topic: notification-events-high (20 partitions)
       - Kafka topic: notification-events-normal (80 partitions)
       - Producer services tag events with priority; the ingestion API routes to the appropriate topic.

   Processing Layer:
       - High-priority consumer group: 20 instances (1 per partition), processing validation → preferences → rendering → routing.
       - Normal-priority consumer group: 40 instances (each handles 2 partitions), same pipeline stages.
       - Auto-scaling based on consumer lag: if lag exceeds 10,000 messages, scale up consumers.

   Delivery Layer:
       - Per-channel queues: push-high, push-normal, email-high, email-normal, sms-high, sms-normal.
       - Delivery workers per queue, scaled independently based on queue depth and provider rate limits.
   ```
   - Separating high and normal priority at every stage ensures that a flood of promotional notifications cannot delay 2FA codes or fraud alerts.

2. **Per-provider token bucket rate limiter:**
   ```
   class ProviderRateLimiter:
       transactional_bucket: TokenBucket  // 80% of provider's rate limit
       promotional_bucket: TokenBucket    // 20% of provider's rate limit

       function acquire(priority):
           if priority == HIGH:
               // High-priority always uses transactional bucket
               return transactional_bucket.tryAcquire(timeout=100ms)
           else:
               // Normal-priority tries promotional bucket first, then transactional overflow
               if promotional_bucket.tryAcquire(timeout=0):
                   return true
               return transactional_bucket.tryAcquire(timeout=50ms)

   // Example configuration for FCM (1,000 req/s limit):
   fcm_limiter = ProviderRateLimiter(
       transactional_bucket = TokenBucket(rate=800/s, burst=1000),
       promotional_bucket = TokenBucket(rate=200/s, burst=500)
   )
   ```
   - When a delivery worker cannot acquire a token, the notification is re-queued with a 1-second delay. This naturally throttles delivery to the provider's rate without dropping notifications.
   - The 80/20 split ensures that even during a promotional blast, transactional notifications have guaranteed capacity.

3. **Per-user rate limiting:**
   ```
   function checkUserRateLimit(userId, isMandatory):
       if isMandatory:
           return ALLOWED  // mandatory notifications bypass user rate limits

       key = "user_rate:{userId}:{currentHour}"
       count = redis.incr(key)
       if count == 1:
           redis.expire(key, 3600)  // TTL = 1 hour
       if count > USER_RATE_LIMIT:  // e.g., 50 per hour
           return RATE_LIMITED
       return ALLOWED
   ```
   - Rate-limited notifications are suppressed (not queued for later). The rationale: if a user is receiving 50+ notifications per hour, additional ones add noise, not value.
   - The limit is configurable per notification category. Transactional notifications (order updates) have a higher limit (200/hour) than promotional notifications (20/hour).

4. **End-to-end distributed tracing:**
   - **Trace ID assignment:** When the ingestion API receives an event, it generates a `trace_id` (UUID) and attaches it to the event metadata. If the upstream service already provides a correlation ID, it is stored as `parent_trace_id`.
   - **Span recording at each stage:**
     ```
     Stage: ingestion
       trace_id, timestamp, event_type, user_id, priority, outcome: "accepted"

     Stage: preference_lookup
       trace_id, timestamp, duration_ms, channels_selected: ["push", "email"], quiet_hours: false

     Stage: template_rendering
       trace_id, timestamp, duration_ms, template_id, channels_rendered: ["push", "email"]

     Stage: delivery_push
       trace_id, timestamp, duration_ms, provider: "FCM", attempt: 1, outcome: "delivered", provider_message_id: "..."

     Stage: delivery_email
       trace_id, timestamp, duration_ms, provider: "SendGrid", attempt: 1, outcome: "delivered", provider_message_id: "..."
     ```
   - Spans are published to a Kafka topic (`notification-traces`) and consumed by a trace aggregation service that stores them in a time-series database (e.g., Elasticsearch) indexed by `trace_id`, `user_id`, `event_type`, and `timestamp`.

5. **Operational dashboards and key metrics:**
   - **Delivery latency:** P50, P95, P99 latency from event ingestion to delivery confirmation, broken down by priority and channel. SLA: P99 < 5 min (normal), P99 < 30 sec (high).
   - **Failure rates:** Percentage of failed deliveries per provider per hour. Alert threshold: > 5% failure rate triggers a page.
   - **Queue depths:** Current message count in each Kafka topic and channel-specific queue. Alert: lag > 50,000 messages on high-priority topics.
   - **Circuit breaker states:** Current state (closed/open/half-open) of each provider's circuit breaker. Open state triggers an alert.
   - **Rate limiter utilization:** Percentage of token bucket capacity consumed per provider. Alert: > 90% sustained utilization for 5 minutes.
   - **User rate limit hits:** Count of notifications suppressed by per-user rate limiting per hour. High counts may indicate a misconfigured upstream service generating excessive events.

6. **Handling provider rate limit exhaustion:**
   - When the token bucket for a provider is empty, delivery workers re-queue notifications with a short delay (1–2 seconds).
   - High-priority notifications use the transactional bucket, which has 80% of the capacity, ensuring they are rarely delayed.
   - If the promotional bucket is exhausted and the transactional bucket has spare capacity, normal-priority notifications can overflow into the transactional bucket (see `acquire` function above). This maximizes throughput without starving transactional traffic.
   - During sustained overload (e.g., a marketing blast exceeding the hourly email quota), the system queues excess promotional emails and drains them gradually over the next hours. An alert notifies the marketing team that their campaign is being throttled.

</details>
