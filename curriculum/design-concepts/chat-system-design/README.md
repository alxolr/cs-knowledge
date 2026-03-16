# Chat System Design

**Track:** Design Concepts
**Difficulty Tier:** Advanced
**Prerequisites:** [Message Queues](../message-queues/README.md), [Microservices Architecture](../microservices-architecture/README.md), [URL Shortener Design](../url-shortener-design/README.md)

## Concept Overview

A chat system is one of the most demanding real-time applications to design at scale. At its core, a chat service must accept messages from senders, route them to recipients with minimal latency, persist them for later retrieval, and synchronize state across multiple devices. Services like WhatsApp, Slack, and Discord each handle billions of messages per day, requiring architectures that blend persistent connections (WebSockets), asynchronous message routing (queues and pub/sub), and efficient storage for both recent and historical conversations.

The real-time nature of chat introduces challenges that typical request-response systems do not face. Clients maintain long-lived connections to gateway servers, and the system must track which gateway each user is connected to so that messages can be pushed instantly. Group chats amplify this fan-out problem — a single message sent to a 500-member group must be delivered to 500 different connections, potentially spread across dozens of servers. Presence indicators ("online", "typing…") add another stream of ephemeral, high-frequency events that must propagate quickly without overwhelming the system.

Beyond real-time delivery, a production chat system must guarantee that no message is lost, even when users are offline or switch devices. This requires a reliable delivery pipeline with acknowledgments, retry logic, and offline message queues. Features like end-to-end encryption, media sharing (images, videos, files), read receipts, and message search each introduce their own architectural components. Designing a chat system is therefore an exercise in composing many subsystems — connection management, message routing, storage, notification, and media handling — into a cohesive, scalable whole.

---

## Problem 1 — Real-Time Messaging Architecture

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Design the high-level architecture for a one-on-one chat service that supports real-time message delivery between two users. The design should cover how clients establish persistent connections, how the server routes a message from sender to recipient, and how messages are stored. Focus on the core send-and-receive flow for users who are both online.

### Scenario

**Context:** A startup is building a messaging app that initially supports only one-on-one conversations. The MVP must deliver messages in real time — when Alice sends a message to Bob, Bob should see it within 200 ms if he is online. The team expects 100,000 concurrent users at launch. Each user maintains a single active session (no multi-device support yet). The team needs to decide on the connection protocol, the message routing mechanism, and the storage layer before writing code.

**Requirements:** Choose a protocol for persistent client-server connections and justify why it is preferred over HTTP polling. Design the message flow from sender to recipient, including how the server identifies which gateway server the recipient is connected to. Define a storage schema for messages that supports fetching conversation history in reverse chronological order. Explain what happens when the recipient is online versus offline.

**Expected Approach:** Use WebSocket connections for bidirectional, low-latency communication. Introduce a connection registry (e.g., in Redis) that maps each online user to their gateway server. When a message arrives, the chat service looks up the recipient's gateway and pushes the message. If the recipient is offline, the message is persisted and delivered when they reconnect. Store messages in a table keyed by conversation ID with a timestamp for ordering.

<details>
<summary>Hints</summary>

1. WebSockets provide full-duplex communication over a single TCP connection, eliminating the overhead of repeated HTTP handshakes. Long polling is an alternative but introduces higher latency and server resource waste.
2. A connection registry (user ID → gateway server address) stored in Redis allows any chat service instance to locate the recipient's gateway in O(1) time.
3. For message storage, a composite key of (conversation_id, timestamp) enables efficient range queries to fetch the most recent N messages in a conversation.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build a WebSocket-based architecture with a Redis connection registry and a message store optimized for conversation history retrieval.

1. **Connection layer:**
   - Clients connect to a gateway server via WebSocket. The gateway handles authentication (token-based) during the WebSocket handshake.
   - On successful connection, the gateway registers the mapping `user_id → gateway_server_address` in Redis with a TTL (e.g., 5 minutes, refreshed by heartbeats).
   - On disconnect, the gateway removes the mapping from Redis.

2. **Message send flow (both users online):**
   ```
   Alice's client → WebSocket → Gateway A → Chat Service → lookup Bob in Redis
       → Bob is on Gateway B → forward message to Gateway B → WebSocket → Bob's client
   ```
   - Alice sends a message via her WebSocket connection to Gateway A.
   - Gateway A forwards the message to the Chat Service, which persists it to the message store.
   - The Chat Service looks up Bob's gateway in Redis, finds Gateway B, and sends the message to Gateway B via an internal RPC or message queue.
   - Gateway B pushes the message to Bob's WebSocket connection.

3. **Message send flow (recipient offline):**
   - The Chat Service looks up Bob in Redis and finds no entry.
   - The message is persisted to the message store with a `delivered = false` flag.
   - When Bob reconnects, the gateway queries for undelivered messages and pushes them in order.

4. **Message storage schema:**
   ```
   Table: messages
   +------------------+--------------+----------------------------------------------+
   | Column           | Type         | Notes                                        |
   +------------------+--------------+----------------------------------------------+
   | message_id       | UUID         | Primary key                                  |
   | conversation_id  | VARCHAR(64)  | Partition key; derived from sorted user IDs  |
   | sender_id        | VARCHAR(64)  | Who sent the message                         |
   | content          | TEXT         | Message body                                 |
   | created_at       | TIMESTAMP    | Sort key for chronological ordering          |
   | delivered        | BOOLEAN      | Whether recipient has received the message   |
   +------------------+--------------+----------------------------------------------+
   Index: (conversation_id, created_at DESC) for efficient history queries
   ```
   - `conversation_id` for a 1:1 chat is derived from the two user IDs sorted alphabetically (e.g., `hash(min(alice, bob) + max(alice, bob))`), ensuring both users map to the same conversation.

5. **Why WebSockets over polling:**
   - HTTP polling wastes bandwidth and server resources by repeatedly asking "any new messages?" even when there are none.
   - Long polling reduces wasted requests but still requires re-establishing connections frequently.
   - WebSockets maintain a persistent, bidirectional channel — the server pushes messages instantly without the client asking, achieving sub-200 ms delivery latency.

</details>

---

## Problem 2 — Group Chat Data Modeling and Fan-Out

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design the data model and message distribution strategy for a chat system that supports group conversations with up to 500 members. The design must handle the fan-out problem — delivering a single message to hundreds of recipients efficiently — and support features like member management (add/remove), message history per group, and the ability for users to mute or leave groups without losing prior history.

### Scenario

**Context:** The messaging app from Problem 1 is adding group chat support. Product requirements allow groups of up to 500 members. A popular community group has 400 active members, and when someone posts a message, all 400 members should receive it in near real time. The team is concerned about the fan-out cost: delivering one message to 400 recipients means 400 push operations, potentially across dozens of gateway servers. Additionally, users want to mute notifications for noisy groups, leave groups while retaining message history, and scroll back through months of group conversation history.

**Requirements:** Design the database schema for groups, group memberships, and group messages. Describe the fan-out strategy for delivering a message to all group members — should the system fan out on write (pre-compute each member's inbox) or fan out on read (members pull from the group's message stream)? Analyze the trade-offs of each approach for groups of varying sizes. Explain how muting, leaving, and rejoining a group are modeled without duplicating or losing message data. Support paginated retrieval of group message history.

**Expected Approach:** Use a fan-out-on-write approach for small-to-medium groups (push the message to each member's connection) and consider a hybrid approach for very large groups. Model group membership as a separate table with per-member metadata (muted, last-read timestamp). Store messages once per group (not per member) and use the membership table to control visibility and notification behavior.

<details>
<summary>Hints</summary>

1. Fan-out on write means when a message is sent, the system immediately pushes it to every online member and writes a delivery record for offline members. This is simple and provides low latency but is expensive for large groups (500 pushes per message).
2. Fan-out on read means messages are stored in the group's stream, and each member pulls new messages when they open the app. This is cheaper on write but adds read latency and complexity for real-time delivery.
3. A hybrid approach uses fan-out on write for online members (push via WebSocket) and fan-out on read for offline members (they pull unread messages on reconnect using their `last_read_at` timestamp).
4. Store messages in a `group_messages` table keyed by `(group_id, created_at)`. Do not duplicate messages per member — use the membership table to determine who can see which messages.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use a hybrid fan-out strategy with a normalized data model that stores messages once per group and tracks per-member state in a membership table.

1. **Database schema:**
   ```
   Table: groups
   +------------------+--------------+----------------------------------------------+
   | Column           | Type         | Notes                                        |
   +------------------+--------------+----------------------------------------------+
   | group_id         | UUID         | Primary key                                  |
   | name             | VARCHAR(255) | Group display name                           |
   | created_by       | VARCHAR(64)  | User who created the group                   |
   | created_at       | TIMESTAMP    | Creation time                                |
   | max_members      | INT          | Limit (default 500)                          |
   +------------------+--------------+----------------------------------------------+

   Table: group_memberships
   +------------------+--------------+----------------------------------------------+
   | Column           | Type         | Notes                                        |
   +------------------+--------------+----------------------------------------------+
   | group_id         | UUID         | FK to groups                                 |
   | user_id          | VARCHAR(64)  | Member's user ID                             |
   | role             | ENUM         | 'admin', 'member'                            |
   | joined_at        | TIMESTAMP    | When the user joined                         |
   | left_at          | TIMESTAMP    | Nullable; set when user leaves               |
   | muted            | BOOLEAN      | Whether notifications are silenced           |
   | last_read_at     | TIMESTAMP    | Last message timestamp the user has seen     |
   +------------------+--------------+----------------------------------------------+
   Primary key: (group_id, user_id)

   Table: group_messages
   +------------------+--------------+----------------------------------------------+
   | Column           | Type         | Notes                                        |
   +------------------+--------------+----------------------------------------------+
   | message_id       | UUID         | Primary key                                  |
   | group_id         | UUID         | FK to groups; partition key                  |
   | sender_id        | VARCHAR(64)  | Who sent the message                         |
   | content          | TEXT         | Message body                                 |
   | created_at       | TIMESTAMP    | Sort key for ordering                        |
   +------------------+--------------+----------------------------------------------+
   Index: (group_id, created_at DESC) for paginated history
   ```

2. **Hybrid fan-out strategy:**
   - **On write (for online members):** When a message is sent to a group, the Chat Service queries the `group_memberships` table for all active members (`left_at IS NULL`). For each member, it checks the Redis connection registry. If the member is online, the message is pushed to their gateway via internal RPC. If the member is offline, no immediate action is taken — the message is already persisted in `group_messages`.
   - **On read (for offline members):** When an offline member reconnects, the client sends its `last_read_at` timestamp for each group. The server queries `group_messages WHERE group_id = ? AND created_at > last_read_at ORDER BY created_at ASC` to fetch unread messages.

3. **Fan-out cost analysis:**
   - For a 400-member group where 100 are online: the write path performs 1 database insert + 100 WebSocket pushes. The 300 offline members incur zero write-time cost; they pull messages on reconnect.
   - For a 10-member group where 8 are online: 1 insert + 8 pushes — negligible cost.
   - If all 500 members are online, the fan-out is 500 pushes per message. At 10 messages/second in the group, that is 5,000 pushes/second — manageable with parallel dispatch across gateway servers.

4. **Muting a group:**
   - Set `muted = true` in `group_memberships`. The fan-out logic still delivers the message to the user's client (so it appears in the chat), but the gateway skips sending a push notification to the user's device.

5. **Leaving a group:**
   - Set `left_at = now()` in `group_memberships`. The user retains access to messages with `created_at <= left_at` (history before they left). Messages after `left_at` are not visible to the user.
   - The fan-out logic excludes members where `left_at IS NOT NULL`.

6. **Rejoining a group:**
   - Clear `left_at` (set to NULL) and update `joined_at` to the current time. The user sees messages from `joined_at` onward. Messages between the old `left_at` and new `joined_at` are not visible (they were not a member during that period).

7. **Paginated history retrieval:**
   ```
   function getGroupHistory(groupId, userId, beforeTimestamp, limit=50):
       membership = getMembership(groupId, userId)
       minTimestamp = membership.joined_at
       maxTimestamp = membership.left_at or now()
       return query("SELECT * FROM group_messages
                     WHERE group_id = ? AND created_at < ? AND created_at >= ? AND created_at <= ?
                     ORDER BY created_at DESC LIMIT ?",
                    groupId, beforeTimestamp, minTimestamp, maxTimestamp, limit)
   ```

**Why hybrid fan-out fits:** Pure fan-out on write is wasteful for offline members in large groups. Pure fan-out on read adds latency for online members. The hybrid approach pushes instantly to online members (real-time experience) and defers delivery to offline members (efficient batch retrieval on reconnect), balancing latency and resource cost.

</details>

---

## Problem 3 — Message Delivery Guarantees and Ordering

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design a reliable message delivery pipeline for a chat system that guarantees exactly-once delivery semantics from the user's perspective, handles out-of-order message arrival, and ensures messages within a conversation are displayed in a consistent order across all participants' devices. The design must account for network failures, server crashes, duplicate sends from retrying clients, and multi-device synchronization where a user is logged in on both a phone and a laptop simultaneously.

### Scenario

**Context:** The chat application now supports multi-device sessions — a user can be logged in on their phone and laptop simultaneously, and both devices must display the same conversation state. Users have reported several issues: (1) messages occasionally appear twice when the network is flaky and the client retries a send, (2) messages sometimes appear out of order when two people send messages at nearly the same time, and (3) a message sent from the phone does not always appear on the laptop promptly. The engineering team needs a delivery pipeline that eliminates duplicates, enforces consistent ordering, and synchronizes state across devices.

**Requirements:** Design an end-to-end acknowledgment protocol between client and server that prevents message loss and duplicate delivery. Define how message ordering is determined — should the system use client timestamps, server timestamps, or logical sequence numbers? Explain the trade-offs of each. Design the multi-device synchronization mechanism so that all of a user's devices converge to the same conversation state. Handle the scenario where a client sends a message, loses connectivity before receiving the server acknowledgment, reconnects, and retries — the system must not create a duplicate message. Specify how the server detects and deduplicates retried messages.

**Expected Approach:** Use a client-generated idempotency key (UUID) attached to every message so the server can detect retries. Assign a monotonically increasing server-side sequence number per conversation for ordering. Implement a three-phase delivery protocol: (1) client sends message, (2) server persists and assigns sequence number, (3) server acknowledges to sender and pushes to recipients. For multi-device sync, each device tracks the last sequence number it has seen and pulls any gaps on reconnect.

<details>
<summary>Hints</summary>

1. An idempotency key (client-generated UUID per message) allows the server to detect retries. Before inserting a message, the server checks if a message with that idempotency key already exists. If so, it returns the existing message's details without creating a duplicate.
2. Client timestamps are unreliable for ordering because device clocks can be skewed. Server timestamps are better but can still produce ties when two messages arrive in the same millisecond. A per-conversation sequence number (auto-incrementing integer) provides a total order with no ties.
3. For multi-device sync, each device stores the highest sequence number it has seen for each conversation. On reconnect, it requests all messages with sequence numbers greater than its last-seen value.
4. The three-phase protocol ensures the sender knows the message was persisted (via ACK) before considering it "sent." If the ACK is lost, the client retries with the same idempotency key, and the server returns the existing message — no duplicate.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement an idempotency-key-based deduplication layer, per-conversation server-assigned sequence numbers for ordering, and a gap-detection sync protocol for multi-device consistency.

1. **Idempotency and deduplication:**
   - Every message sent by a client includes a `client_message_id` (UUID v4) generated on the device.
   - The server maintains a unique index on `(conversation_id, client_message_id)`.
   - On receiving a message:
     ```
     function handleIncomingMessage(conversationId, senderId, clientMessageId, content):
         existing = db.query("SELECT * FROM messages WHERE conversation_id = ? AND client_message_id = ?",
                             conversationId, clientMessageId)
         if existing:
             return ACK(existing.message_id, existing.seq_num)  // idempotent response
         seq = getNextSequenceNumber(conversationId)
         msg = db.insert(conversationId, senderId, clientMessageId, content, seq, now())
         return ACK(msg.message_id, msg.seq_num)
     ```
   - If the client retries (network failure before ACK), the server finds the existing message and returns the same ACK — no duplicate is created.

2. **Ordering with per-conversation sequence numbers:**
   - Each conversation has a monotonically increasing sequence counter stored in a dedicated row or managed via an atomic increment operation.
   - When a message is persisted, it is assigned the next sequence number for that conversation.
   - Clients display messages sorted by `seq_num`, not by timestamp. This provides a total order with no ties.
   - **Why not client timestamps:** Device clocks can be minutes or hours off. Two users sending messages "at the same time" could produce reversed timestamps.
   - **Why not server timestamps:** Better than client timestamps, but two messages arriving in the same millisecond produce a tie. Sequence numbers eliminate this ambiguity.
   - **Timestamp role:** `created_at` is still stored for display purposes ("2:34 PM") but is not used for ordering.

3. **Three-phase delivery protocol:**
   ```
   Phase 1 — Send:
       Client generates client_message_id, sends (conversation_id, client_message_id, content) via WebSocket.
       Client marks message as "sending" in local UI.

   Phase 2 — Persist & Assign:
       Server deduplicates using client_message_id.
       Server persists message and assigns seq_num.
       Server sends ACK(message_id, seq_num) back to sender.
       Client marks message as "sent" in local UI.

   Phase 3 — Deliver:
       Server pushes message to all other participants (and sender's other devices).
       Each recipient's client sends a DELIVERED acknowledgment.
       Server updates delivery status.
   ```

4. **Multi-device synchronization:**
   - Each device tracks `last_seq_num` per conversation — the highest sequence number it has received.
   - On reconnect or app foreground, the device sends a sync request:
     ```
     SYNC { conversation_id: "abc", last_seq_num: 147 }
     ```
   - The server responds with all messages where `seq_num > 147` for that conversation.
   - When the sender's phone sends a message, the server pushes it to the sender's laptop as well (Phase 3). The laptop receives it and updates its `last_seq_num`.
   - If the laptop was offline when the phone sent the message, the laptop's sync request on reconnect will fetch it.

5. **Handling network failures:**
   - **Client sends, no ACK received:** Client retries after a timeout (e.g., 3 seconds) with the same `client_message_id`. Server deduplicates and returns the existing ACK.
   - **Server crashes after persist but before ACK:** On restart, the message is already in the database. Client retries, server finds the existing message, returns ACK.
   - **Server crashes before persist:** Message is lost. Client retries, server inserts it as a new message. From the user's perspective, the message was simply delayed.
   - **Retry limit:** Client retries up to 5 times with exponential backoff. After 5 failures, the message is marked as "failed" in the UI, and the user can manually retry.

6. **Sequence number generation at scale:**
   - For low-to-moderate throughput conversations, an atomic `UPDATE conversations SET seq_counter = seq_counter + 1 WHERE id = ? RETURNING seq_counter` works well.
   - For high-throughput group chats (hundreds of messages per second), pre-allocate sequence number ranges (e.g., allocate 100 numbers at a time) to reduce database contention.

**Why this design ensures reliability:** The idempotency key eliminates duplicates regardless of how many times a client retries. The server-assigned sequence number provides a single source of truth for ordering that all devices converge on. The sync protocol ensures that offline devices catch up completely by requesting messages after their last-known sequence number.

</details>

---

## Problem 4 — Presence and Typing Indicators at Scale

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design a presence system and typing indicator feature for a chat application with 10 million concurrent users. The presence system must track whether each user is online, idle, or offline and propagate status changes to relevant contacts in near real time. The typing indicator must show "Alice is typing…" to conversation participants within 1 second of Alice starting to type, and disappear shortly after she stops. Both features generate high-frequency, ephemeral events that must not overwhelm the system or degrade message delivery performance.

### Scenario

**Context:** The chat application has grown to 10 million concurrent users. The product team wants to add presence indicators (green dot for online, yellow for idle, gray for offline) on each contact in a user's contact list, and typing indicators ("typing…") in active conversations. A naive implementation that broadcasts every presence change to every contact would generate enormous traffic — if a user has 200 contacts and goes online, that is 200 push events. With 10 million users going online/offline throughout the day, the system would be flooded. Similarly, typing events fire on every keystroke, potentially generating 5–10 events per second per active typist. The team needs a design that delivers these features without overwhelming the network or the servers.

**Requirements:** Design the presence tracking mechanism — how the server detects that a user is online, idle, or offline (considering WebSocket heartbeats, app backgrounding, and network drops). Design the presence propagation strategy — how status changes are delivered to contacts without broadcasting to every contact on every change. Specify how typing indicators are transmitted, throttled, and expired. Analyze the event volume for both features and describe how the system limits the load. Ensure that presence and typing events do not interfere with message delivery (lower priority).

**Expected Approach:** Use WebSocket heartbeats and a server-side TTL to track presence. Propagate presence changes only to contacts who are currently viewing a screen where the status is visible (subscription-based model). Throttle typing events to at most one per 2–3 seconds per conversation and auto-expire them after a timeout. Use a separate pub/sub channel for ephemeral events so they do not compete with message delivery.

<details>
<summary>Hints</summary>

1. Instead of broadcasting presence to all 200 contacts, use a subscription model: a user's client subscribes to presence updates only for contacts currently visible on screen (e.g., the 20 contacts in the active chat list). This reduces fan-out from 200 to ~20 per status change.
2. Detect "online" via an active WebSocket connection with periodic heartbeats (every 30 seconds). Detect "idle" if no user interaction (message sent, screen tap) for 5 minutes. Detect "offline" if the WebSocket disconnects or heartbeat is missed for 60 seconds.
3. Typing indicators should be throttled on the client side — send a "typing" event at most once every 3 seconds while the user is actively typing. The recipient's client displays "typing…" and auto-hides it after 4 seconds if no new typing event arrives.
4. Use a dedicated pub/sub system (e.g., Redis Pub/Sub) for presence and typing events, separate from the message delivery pipeline. These events are fire-and-forget — if one is lost, the UI simply shows a slightly stale status, which is acceptable.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement a heartbeat-based presence tracker with subscription-based propagation, and a throttled typing indicator system, both running on a dedicated ephemeral event channel separate from message delivery.

1. **Presence detection:**
   - **Online:** User has an active WebSocket connection and has sent a heartbeat within the last 60 seconds.
   - **Idle:** User has an active WebSocket connection but no user-initiated activity (message sent, UI interaction) for 5 minutes. The client sends an "idle" signal when the app is backgrounded or the screen locks.
   - **Offline:** WebSocket disconnects, or no heartbeat received for 60 seconds (covers abrupt network loss).
   - **Storage:** Redis hash `presence:{user_id}` with fields `status` (online/idle/offline), `last_active_at`, and a TTL of 90 seconds. Each heartbeat refreshes the TTL. If the TTL expires, the user is considered offline.

2. **Subscription-based presence propagation:**
   - When a client opens the app, it sends a subscription request for the presence of contacts currently visible on screen (e.g., the 15–20 contacts in the chat list).
   - The server subscribes the client to Redis Pub/Sub channels: `presence:{contact_user_id}` for each subscribed contact.
   - When a user's presence changes, the server publishes to `presence:{user_id}`. Only clients currently subscribed to that channel receive the update.
   - When the client scrolls or navigates away, it unsubscribes from contacts no longer visible and subscribes to newly visible ones.
   - **Fan-out reduction:** Instead of pushing to all 200 contacts, the system pushes only to the ~5–20 contacts who are actively viewing the user's status. At 10 million concurrent users, if each user subscribes to 20 presence channels, the system manages 200 million subscriptions — feasible with a Redis Pub/Sub cluster.

3. **Presence event volume analysis:**
   - Assume each user changes presence state ~10 times per day (login, idle, active, logout cycles).
   - 10 million users × 10 changes/day = 100 million presence events/day ≈ 1,157 events/second.
   - Each event fans out to ~10 subscribers (contacts currently viewing) = ~11,570 pushes/second. This is well within the capacity of a Redis Pub/Sub cluster.

4. **Typing indicator design:**
   - **Client-side throttling:** The client sends a `TYPING` event when the user starts typing, and at most once every 3 seconds while typing continues. When the user stops typing (no keystrokes for 3 seconds), the client sends a `STOPPED_TYPING` event.
   - **Server relay:** The server receives the `TYPING` event and forwards it to all other participants in the conversation via their WebSocket connections. No persistence — typing events are ephemeral.
   - **Recipient-side expiration:** The recipient's client displays "Alice is typing…" upon receiving a `TYPING` event. If no new `TYPING` event arrives within 4 seconds, the indicator auto-hides. This handles the case where the `STOPPED_TYPING` event is lost.

5. **Typing event volume analysis:**
   - Assume 1 million users are actively typing at any moment, each generating 1 throttled event every 3 seconds = 333,333 events/second.
   - Average conversation size is 2 (1:1 chat), so each event fans out to 1 recipient = 333,333 pushes/second.
   - For group chats, a typing event in a 50-member group fans out to 49 recipients, but only a small fraction of groups have active typists at any moment.
   - Total estimated load: ~400,000–500,000 ephemeral pushes/second, handled by the gateway servers' WebSocket connections.

6. **Separation from message delivery:**
   - Presence and typing events use a dedicated pub/sub channel and are processed by a separate thread pool on the gateway servers.
   - These events are fire-and-forget with no persistence, acknowledgment, or retry. If a presence update is lost, the client will receive the correct status on the next change or when it re-subscribes.
   - Message delivery uses a separate, reliable pipeline with persistence and acknowledgments (Problem 3). The two pipelines share the WebSocket connection but use different message types, and the gateway prioritizes message delivery over ephemeral events.

7. **Edge cases:**
   - **Abrupt disconnect (phone loses signal):** The heartbeat TTL in Redis expires after 60 seconds, triggering an offline status change. Subscribers are notified.
   - **Rapid reconnect (elevator scenario):** If the user reconnects within the 60-second TTL window, the heartbeat is refreshed and no offline event is published — avoiding a flicker of "offline" then "online."
   - **Large group typing:** In a 500-member group, if 10 people type simultaneously, the UI shows "Alice, Bob, and 8 others are typing…" The client aggregates typing events locally rather than displaying each individually.

**Why subscription-based presence fits:** Broadcasting to all contacts is O(contacts × users), which is prohibitively expensive at scale. The subscription model reduces fan-out to O(active_viewers), which is typically 10–100× smaller. Combined with client-side throttling for typing events and a dedicated ephemeral channel, the system delivers real-time presence and typing indicators without impacting message delivery performance.

</details>

---

## Problem 5 — End-to-End Encryption and Media Sharing

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design the end-to-end encryption (E2EE) scheme and media sharing pipeline for a chat application. The E2EE design must ensure that the server cannot read message content — only the sender and intended recipients can decrypt messages. The media sharing pipeline must handle image, video, and file uploads up to 100 MB, deliver them efficiently to recipients, and integrate with the E2EE scheme so that media is also encrypted. The design must address key exchange for new conversations, key management for group chats where members join and leave, and the trade-offs between security and features like server-side message search.

### Scenario

**Context:** The chat application is adding end-to-end encryption as a headline privacy feature and media sharing (photos, videos, documents) as a core functionality. For E2EE, the team wants a design similar to the Signal Protocol, where the server acts only as a relay and cannot decrypt any content. For media, users expect to send photos and videos up to 100 MB with thumbnail previews, and recipients should be able to download media on demand rather than having it auto-downloaded (to save bandwidth on mobile). The challenge is combining these two features: media must be encrypted before upload, and only conversation participants should be able to decrypt it. Additionally, when a new member joins a group, they should be able to read future messages but not decrypt historical messages sent before they joined.

**Requirements:** Design the key exchange protocol for establishing an encrypted session between two users who may never have communicated before. Explain how messages are encrypted and decrypted in a 1:1 conversation. Extend the design to group chats, including how encryption keys are managed when members join or leave. Design the media upload and delivery pipeline — how media is encrypted, uploaded to object storage, and how recipients retrieve and decrypt it. Address thumbnail generation (can the server generate thumbnails if it cannot decrypt the media?). Discuss the trade-off between E2EE and server-side features like message search, link previews, and spam detection.

**Expected Approach:** Use asymmetric key exchange (e.g., X3DH from the Signal Protocol) to establish a shared secret between two users, then use symmetric encryption (AES-256) for message content. For groups, use a shared group key (sender keys) that is rotated when membership changes. For media, encrypt the file client-side with a random symmetric key, upload the encrypted blob to object storage, and send the decryption key inside the encrypted message. Thumbnails must be generated client-side before encryption. Acknowledge that E2EE prevents server-side search and spam detection on message content.

<details>
<summary>Hints</summary>

1. In the Signal Protocol's X3DH (Extended Triple Diffie-Hellman) key exchange, each user publishes a set of public keys to the server. When Alice wants to message Bob for the first time, she downloads Bob's public keys and computes a shared secret without Bob needing to be online. This shared secret is then used to derive symmetric encryption keys.
2. For group E2EE, the "sender keys" approach is efficient: each member generates a sender key and distributes it to all other members (encrypted with each member's individual session key). When sending a group message, the sender encrypts once with their sender key, and all members can decrypt. When a member leaves, all remaining members rotate their sender keys to exclude the departed member.
3. For media, generate a random AES-256 key on the client, encrypt the file, upload the encrypted blob to object storage (e.g., S3), and include the AES key and the blob URL in the encrypted chat message. The server stores the encrypted blob but cannot decrypt it because it does not have the AES key.
4. Thumbnails cannot be generated server-side because the server cannot decrypt the media. The sender's client must generate the thumbnail, encrypt it separately, and include it in the message payload so recipients can display a preview without downloading the full file.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement X3DH-based key exchange for 1:1 sessions, sender-key-based group encryption with key rotation on membership changes, and a client-side-encrypted media pipeline with pre-generated thumbnails.

1. **Key exchange for 1:1 conversations (X3DH):**
   - Each user generates and publishes to the server:
     - An identity key pair (long-term, IK)
     - A signed pre-key pair (medium-term, SPK, rotated weekly)
     - A set of one-time pre-keys (OPK, each used once and discarded)
   - When Alice wants to message Bob for the first time:
     1. Alice downloads Bob's IK, SPK, and one OPK from the server.
     2. Alice performs X3DH: computes a shared secret from `DH(Alice_IK, Bob_SPK) || DH(Alice_EK, Bob_IK) || DH(Alice_EK, Bob_SPK) || DH(Alice_EK, Bob_OPK)` where EK is an ephemeral key Alice generates.
     3. Alice derives a symmetric root key from the shared secret using HKDF.
     4. Alice sends her first message encrypted with a message key derived from the root key, along with her IK and EK public keys.
     5. Bob receives the message, performs the same DH computations using his private keys, derives the same shared secret, and decrypts the message.
   - The server never sees the shared secret or the derived message keys — it only relays encrypted ciphertext and public keys.

2. **Message encryption in 1:1 conversations (Double Ratchet):**
   - After the initial X3DH, both parties use the Double Ratchet algorithm to derive a new message key for every message.
   - Each message is encrypted with AES-256-GCM using a unique message key. Even if one key is compromised, past and future messages remain secure (forward secrecy).
   - The ratchet state is stored on each client device. The server stores only encrypted ciphertext.

3. **Group chat encryption (Sender Keys):**
   - When a group is created, each member generates a sender key (a symmetric key + chain key for ratcheting).
   - Each member distributes their sender key to every other member, encrypted with the pairwise session key established via X3DH.
   - To send a group message, the sender encrypts once with their sender key. All members decrypt using the sender's key.
   - **Member joins:** Existing members send their current sender keys to the new member (encrypted with pairwise session keys). The new member generates and distributes their own sender key. The new member cannot decrypt messages sent before they received the sender keys (no backward access).
   - **Member leaves:** All remaining members rotate their sender keys and redistribute them, excluding the departed member. This ensures the departed member cannot decrypt future messages (forward secrecy on leave).
   - **Trade-off:** Sender keys are efficient (one encryption per message regardless of group size) but require key redistribution on every membership change. For groups with frequent membership changes, this overhead is significant.

4. **Media sharing pipeline:**
   ```
   Sender's client:
   1. Generate a random AES-256 key (media_key).
   2. Encrypt the media file with media_key using AES-256-GCM → encrypted_blob.
   3. Generate a thumbnail of the original media (e.g., 200x200 JPEG for images, first-frame for video).
   4. Encrypt the thumbnail with a separate AES-256 key (thumb_key) → encrypted_thumbnail.
   5. Upload encrypted_blob and encrypted_thumbnail to object storage (S3). Receive blob_url and thumb_url.
   6. Compose a chat message containing: blob_url, thumb_url, media_key, thumb_key, file metadata (name, size, MIME type).
   7. Encrypt the entire chat message using the conversation's encryption protocol (Double Ratchet for 1:1, Sender Key for group).
   8. Send the encrypted message via WebSocket.

   Recipient's client:
   1. Decrypt the chat message to obtain blob_url, thumb_url, media_key, thumb_key, and metadata.
   2. Download and decrypt the thumbnail for immediate preview display.
   3. When the user taps "Download," fetch encrypted_blob from blob_url and decrypt with media_key.
   ```

5. **Server's role in media:**
   - The server stores encrypted blobs in object storage. It cannot decrypt them (no access to media_key).
   - The server can enforce upload size limits (100 MB), rate limits, and virus scanning on encrypted blobs (limited effectiveness since content is encrypted).
   - Thumbnails are generated client-side because the server cannot access the original media. This adds processing time on the sender's device but preserves E2EE.

6. **Trade-offs between E2EE and server-side features:**
   - **Message search:** The server cannot index encrypted messages. Search must be performed client-side on decrypted messages stored locally. This limits search to messages available on the current device.
   - **Link previews:** The server cannot read URLs in messages to generate previews. The sender's client must generate link previews before encryption and include them in the message payload.
   - **Spam detection:** The server cannot inspect message content for spam or abuse. Alternatives include client-side reporting (users flag messages), metadata analysis (message frequency, contact patterns), and analyzing unencrypted metadata (sender, recipient, timestamp, message size).
   - **Backup and restore:** If the user loses their device, encrypted messages are unrecoverable unless the user has an encrypted backup. The system can offer encrypted cloud backups where the backup is encrypted with a user-chosen passphrase — the server stores the backup but cannot decrypt it.

7. **Key storage and device management:**
   - Private keys (identity key, session keys, sender keys) are stored in the device's secure enclave or keychain.
   - When a user adds a new device, they must verify the new device (e.g., QR code scan from an existing device) and transfer session keys securely. Alternatively, the new device establishes fresh sessions with all contacts, and historical messages are transferred via an encrypted device-to-device sync.

**Why this design preserves privacy:** The server never has access to plaintext messages, media content, or encryption keys. It acts purely as a relay for encrypted data and a directory for public keys. The X3DH and Double Ratchet protocols provide forward secrecy and post-compromise security. The sender keys scheme extends E2EE to groups efficiently while maintaining forward secrecy on membership changes. Media encryption ensures that even the object storage provider cannot access user content.

</details>
