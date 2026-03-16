# Capstone: End-to-End System Design

**Track:** Design Concepts
**Difficulty Tier:** Advanced
**Prerequisites:** [URL Shortener Design](../url-shortener-design/README.md), [Chat System Design](../chat-system-design/README.md), [Payment System Design](../payment-system-design/README.md), [Video Streaming Design](../video-streaming-design/README.md), [Monitoring & Observability](../monitoring-and-observability/README.md)

## Concept Overview

End-to-end system design is the culmination of every concept explored throughout the Design Concepts track. Rather than focusing on a single subsystem or pattern in isolation, this capstone module challenges you to synthesize knowledge from across the entire curriculum — API design, caching, load balancing, message queues, data partitioning, replication, microservices, monitoring, and the full suite of end-to-end system designs you have studied — into cohesive, production-grade architectures.

Real-world systems are never built from a single design pattern. A ride-sharing platform combines real-time messaging (chat), payment processing, geospatial search, notification delivery, and observability into one unified product. An e-commerce marketplace weaves together URL routing, caching layers, distributed storage, payment flows, and fraud detection. The ability to identify which patterns apply where, how subsystems interact at their boundaries, and where trade-offs must be made between consistency, availability, latency, and cost is what separates a junior designer from a senior architect.

Each problem in this module presents a large-scale system that spans multiple domains. You are expected to decompose the system into well-defined services, specify how those services communicate, identify the data storage and caching strategies for each, design for failure and observability from the start, and justify your trade-offs with concrete reasoning. There is no single correct answer — the goal is to demonstrate breadth of knowledge, depth of reasoning, and the ability to make principled design decisions under real-world constraints.

---

## Problem 1 — Designing a Multi-Vendor E-Commerce Marketplace

**Difficulty:** Easy
**Estimated Time:** 40 minutes

### Problem Statement

Design the high-level architecture for a multi-vendor e-commerce marketplace where independent sellers list products and customers browse, search, and purchase items. The platform handles product catalog management, search, shopping cart, checkout with payment processing, and order fulfillment tracking. Focus on identifying the core services, their responsibilities, and how they interact to serve a customer journey from product discovery through order confirmation.

### Scenario

**Context:** A startup is building a marketplace similar to Etsy, where thousands of independent vendors sell handmade goods. The platform expects 50,000 daily active users at launch, growing to 500,000 within a year. Vendors manage their own product listings through a seller dashboard. Customers browse by category, search by keyword, add items from multiple vendors to a single cart, and check out with a single payment that is split among the relevant vendors. The team needs a clear service decomposition before building anything.

**Requirements:** Identify the major services and their responsibilities. Describe the end-to-end flow for a customer searching for a product, adding it to a cart, and completing checkout. Explain how a single payment is split across multiple vendors. State which communication patterns (synchronous vs. asynchronous) are used between services and why. Describe how the system handles a vendor's product going out of stock after a customer adds it to their cart but before checkout completes.

**Expected Approach:** Decompose into a Product Catalog Service, Search Service, Cart Service, Order Service, Payment Service, and Notification Service. Use synchronous REST calls for the customer-facing checkout flow and asynchronous message queues for post-checkout processing (order fulfillment, vendor notifications). Implement inventory reservation at checkout time with a TTL-based hold to handle the out-of-stock race condition.

<details>
<summary>Hints</summary>

1. Think of the marketplace as a coordination layer on top of independent vendor inventories. The Product Catalog Service owns the source of truth for listings, but the Search Service maintains its own denormalized index optimized for keyword and category queries. These two services synchronize via events — when a vendor updates a listing, the catalog publishes an event that the search service consumes to update its index.
2. The payment split can be handled by the Payment Service creating a single charge to the customer and then recording separate ledger entries for each vendor's share (minus the platform's commission). The actual payout to vendors happens asynchronously on a settlement schedule (e.g., daily or weekly), not at checkout time.
3. For the out-of-stock race condition, consider a reservation pattern: when the customer initiates checkout, the Order Service places a temporary hold on the inventory (decrementing available stock). If checkout completes, the hold becomes permanent. If checkout fails or the hold expires (e.g., after 10 minutes), the inventory is released back.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Decompose the marketplace into domain-aligned microservices with clear ownership boundaries, using synchronous calls for the latency-sensitive checkout path and asynchronous events for everything else.

1. **Core services and responsibilities:**
   - **Product Catalog Service:** CRUD operations for product listings. Vendors create, update, and delete products. Stores product details (title, description, price, images, inventory count) in a relational database. Publishes `ProductUpdated` and `ProductDeleted` events to a message queue.
   - **Search Service:** Maintains a search index (Elasticsearch) built from Product Catalog events. Supports keyword search, category filtering, price range filtering, and relevance ranking. Reads only — never writes to the catalog directly.
   - **Cart Service:** Manages per-user shopping carts. Stores cart items in a Redis-backed store for low-latency reads. Carts are ephemeral — they expire after 7 days of inactivity. Each cart item references a product ID, vendor ID, quantity, and the price at the time of addition.
   - **Order Service:** Orchestrates the checkout flow. Validates cart contents against current inventory, creates inventory reservations, creates an order record, and coordinates with the Payment Service. Publishes `OrderCreated` events for downstream processing.
   - **Payment Service:** Processes the customer's payment via an external payment gateway. Records the total charge and computes the per-vendor split (product price minus platform commission). Writes ledger entries for each vendor's share. Vendor payouts are processed in a separate batch job.
   - **Notification Service:** Sends transactional emails and push notifications. Consumes events from the Order Service (order confirmation to customer, new order alert to vendor) and Payment Service (payout confirmation to vendor).

2. **End-to-end customer flow:**
   ```
   1. Customer searches "handmade ceramic mug" → API Gateway routes to Search Service.
   2. Search Service queries Elasticsearch, returns ranked product results with prices and vendor names.
   3. Customer selects a product → frontend fetches full details from Product Catalog Service.
   4. Customer clicks "Add to Cart" → Cart Service stores the item (product_id, vendor_id, quantity, price snapshot).
   5. Customer clicks "Checkout" → Order Service receives the cart contents.
   6. Order Service validates each item:
      a. Calls Product Catalog Service to verify each product still exists and the price has not changed.
      b. Attempts to reserve inventory: UPDATE products SET available_stock = available_stock - quantity WHERE product_id = ? AND available_stock >= quantity. If the update affects 0 rows, the item is out of stock.
   7. If all items are valid and reserved, Order Service creates an order record (status: PENDING_PAYMENT) with a reservation expiry of 10 minutes.
   8. Order Service calls Payment Service synchronously with the total amount and the per-vendor breakdown.
   9. Payment Service charges the customer via the external gateway. On success, records ledger entries and returns a confirmation.
   10. Order Service updates the order to CONFIRMED, converts reservations to permanent stock deductions, and publishes an OrderCreated event.
   11. Notification Service sends order confirmation email to the customer and new-order alerts to each vendor.
   ```

3. **Payment split logic:**
   - The customer is charged once for the total cart amount (sum of all items + shipping).
   - The Payment Service records the split internally:
     ```
     Total charge: $85.00
     Vendor A (ceramic mug): $45.00 - $4.50 commission = $40.50
     Vendor B (wooden spoon): $25.00 - $2.50 commission = $22.50
     Shipping: $15.00 (split proportionally or charged to vendors based on policy)
     Platform revenue: $7.00 commission + shipping margin
     ```
   - Vendor payouts are batched daily. A scheduled job aggregates each vendor's confirmed orders and initiates bank transfers.

4. **Communication patterns:**
   - **Synchronous (REST/gRPC):** Cart → Order Service (checkout initiation), Order Service → Product Catalog (inventory validation), Order Service → Payment Service (charge). These are on the critical checkout path where the customer is waiting.
   - **Asynchronous (message queue):** Product Catalog → Search Service (index updates), Order Service → Notification Service (order events), Payment Service → Notification Service (payout events). These are not latency-sensitive and benefit from decoupling.

5. **Out-of-stock handling:**
   - Inventory reservation uses an atomic database operation (`UPDATE ... WHERE available_stock >= quantity`). If the product went out of stock between cart addition and checkout, the update affects 0 rows.
   - The Order Service detects this and returns an error to the customer: "Sorry, [product name] is no longer available in the requested quantity." The customer can update their cart and retry.
   - Reservations have a 10-minute TTL. A background job releases expired reservations by restoring the reserved stock. This prevents inventory from being locked indefinitely if a customer abandons checkout.

</details>

---

## Problem 2 — Designing a Ride-Sharing Platform

**Difficulty:** Medium
**Estimated Time:** 50 minutes

### Problem Statement

Design a ride-sharing platform that matches riders with nearby drivers in real time, tracks rides from request through completion, processes payments, and provides observability into system health. The design must handle geospatial matching at scale, real-time location updates, surge pricing, and graceful degradation when downstream services are unavailable.

### Scenario

**Context:** A transportation company is building a ride-sharing service for a metropolitan area with 2 million residents. At peak hours (morning and evening commutes), the system handles 5,000 ride requests per minute and receives GPS location updates from 20,000 active drivers every 3 seconds. When a rider requests a ride, the system must find the nearest available driver and present an estimated fare within 2 seconds. Drivers accept or decline the match within 15 seconds. Once a ride is in progress, both the rider and driver see a live map with the vehicle's current position. After the ride, the fare is calculated based on distance and time, the rider is charged, and the driver is credited. The operations team needs dashboards showing request volume, match success rate, average wait time, and payment failure rate in real time.

**Requirements:** Design the service architecture for rider-driver matching, including how geospatial data is stored and queried. Explain how real-time location tracking works during an active ride. Describe the surge pricing mechanism and when it activates. Design the payment flow for ride completion, including how the driver's share is calculated. Outline the monitoring and observability strategy, specifying which metrics are tracked and how alerts are configured. Address what happens when the payment service is temporarily unavailable after a ride completes.

**Expected Approach:** Use a geospatial index (e.g., geohash-based grid or R-tree) for driver location lookups. Stream driver GPS updates through a message queue into the location store. Implement surge pricing as a multiplier derived from the ratio of pending requests to available drivers in a geographic zone. Process payments asynchronously after ride completion with retry logic. Integrate structured logging, distributed tracing, and metric dashboards for observability.

<details>
<summary>Hints</summary>

1. Geospatial matching at scale requires a spatial index, not a brute-force distance calculation against all drivers. A common approach is to divide the city into geohash cells (e.g., 6-character geohashes, roughly 1.2 km × 0.6 km). When a rider requests a ride, query the rider's geohash cell and its 8 neighboring cells for available drivers. This limits the search space to a small geographic area.
2. Real-time location tracking during a ride can use WebSocket connections between the driver's app and a Location Service. The Location Service publishes updates to a topic keyed by ride_id. The rider's app subscribes to that topic and renders the driver's position on the map. This is conceptually similar to the real-time messaging patterns from the Chat System Design module.
3. Surge pricing is a supply-demand balancing mechanism. Divide the city into pricing zones (larger than geohash cells — perhaps neighborhoods). For each zone, compute `demand = pending_ride_requests / available_drivers`. If demand exceeds a threshold (e.g., 1.5), apply a surge multiplier (e.g., 1.5x–3.0x) to the base fare. The multiplier is shown to the rider before they confirm the request.
4. If the payment service is unavailable after a ride completes, the ride should still be marked as completed (the rider should not be stuck in an "active ride" state). Enqueue the payment for asynchronous processing and retry. The rider sees a "payment pending" status, and the charge is processed once the payment service recovers.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build a microservices architecture centered around a Matching Service with geospatial indexing, a Location Service for real-time tracking, a Pricing Service for dynamic surge calculation, and integrate payment processing and observability as cross-cutting concerns.

1. **Service architecture:**
   - **API Gateway:** Authenticates riders and drivers, routes requests, applies rate limiting.
   - **Matching Service:** Receives ride requests, queries the geospatial index for nearby available drivers, ranks candidates by distance and rating, and dispatches match offers to drivers. Maintains driver availability state (available, on-ride, offline).
   - **Location Service:** Ingests GPS updates from driver apps via WebSocket connections. Stores current driver positions in a geospatial index (Redis with geospatial commands or a dedicated geo-database). Publishes location updates for active rides to a pub/sub topic so rider apps can track their driver in real time.
   - **Pricing Service:** Computes fare estimates and surge multipliers. Maintains per-zone demand/supply ratios updated every 30 seconds. Exposes an API: given pickup and dropoff coordinates, returns the estimated fare including any surge multiplier.
   - **Ride Service:** Manages the ride lifecycle (REQUESTED → MATCHED → DRIVER_EN_ROUTE → IN_PROGRESS → COMPLETED → PAID). Stores ride records with timestamps for each state transition.
   - **Payment Service:** Charges the rider and credits the driver after ride completion. Handles the platform commission split. Integrates with the external payment gateway.
   - **Notification Service:** Sends push notifications for ride events (driver matched, driver arriving, ride completed, payment receipt).
   - **Observability Stack:** Collects metrics (Prometheus), logs (structured JSON to a centralized log aggregator), and traces (distributed tracing with OpenTelemetry). Dashboards in Grafana.

2. **Geospatial matching flow:**
   ```
   1. Rider requests a ride with pickup coordinates (lat, lng).
   2. Matching Service calls Pricing Service to get the fare estimate and surge multiplier for the pickup zone.
   3. Rider confirms the fare. Matching Service queries the Location Service for available drivers within a 3 km radius of the pickup point.
   4. Location Service performs a geospatial query: GEORADIUS available_drivers <lng> <lat> 3 km ASC COUNT 10.
   5. Matching Service ranks the returned drivers by (distance × 0.7 + inverse_rating × 0.3) and selects the top candidate.
   6. Matching Service sends a match offer to the selected driver via push notification. The driver has 15 seconds to accept.
   7. If the driver accepts: Ride Service creates a ride record (status: MATCHED), marks the driver as unavailable in the Location Service, and notifies the rider.
   8. If the driver declines or times out: Matching Service selects the next candidate. After 3 failed attempts, the rider is notified that no drivers are available.
   ```

3. **Real-time location tracking:**
   - Drivers maintain a persistent WebSocket connection to the Location Service. Every 3 seconds, the driver app sends a GPS update `{driver_id, lat, lng, timestamp}`.
   - For drivers not on a ride, the Location Service updates the geospatial index (used for matching).
   - For drivers on an active ride, the Location Service additionally publishes the update to a pub/sub channel keyed by `ride:{ride_id}`. The rider's app subscribes to this channel via a WebSocket connection to the Location Service and renders the driver's position on the map.
   - Updates are fire-and-forget — if one GPS update is lost, the next one arrives in 3 seconds. No acknowledgment or retry is needed.

4. **Surge pricing mechanism:**
   - The city is divided into pricing zones (e.g., neighborhoods or postal code areas).
   - Every 30 seconds, the Pricing Service computes per-zone metrics:
     ```
     demand = count of ride requests in zone in last 5 minutes
     supply = count of available drivers in zone right now
     ratio = demand / max(supply, 1)
     ```
   - Surge multiplier tiers:
     - ratio < 1.0: no surge (1.0x)
     - 1.0 ≤ ratio < 1.5: mild surge (1.2x)
     - 1.5 ≤ ratio < 2.5: moderate surge (1.5x)
     - 2.5 ≤ ratio < 4.0: high surge (2.0x)
     - ratio ≥ 4.0: extreme surge (3.0x, capped)
   - The surge multiplier is applied to the base fare (base_fare = base_rate + per_km_rate × distance + per_min_rate × estimated_time). The rider sees the surged fare before confirming.
   - Surge pricing incentivizes more drivers to enter high-demand zones, naturally balancing supply and demand.

5. **Payment flow after ride completion:**
   ```
   1. Ride completes. Ride Service calculates the final fare based on actual distance traveled and ride duration, applying the surge multiplier that was locked in at request time.
   2. Ride Service publishes a RideCompleted event to the message queue.
   3. Payment Service consumes the event and charges the rider:
      Final fare: $23.40 (base $18.00 × 1.3 surge)
      Platform commission (25%): $5.85
      Driver payout: $17.55
   4. Payment Service records ledger entries and enqueues the driver payout for the next settlement batch.
   5. Notification Service sends a receipt to the rider and an earnings summary to the driver.
   ```
   - If the Payment Service is unavailable, the RideCompleted event remains in the queue and is retried with exponential backoff. The ride is marked COMPLETED (not PAID) so the rider can exit the app. Once payment succeeds, the status updates to PAID and the receipt is sent.

6. **Monitoring and observability strategy:**
   - **Key metrics (Prometheus + Grafana dashboards):**
     - `ride_requests_per_minute` (by zone): Tracks demand. Alert if it drops to zero (possible outage).
     - `match_success_rate`: Percentage of requests that result in a driver match. Alert if below 70%.
     - `average_wait_time_seconds`: Time from ride request to driver match. Alert if p95 exceeds 120 seconds.
     - `payment_failure_rate`: Percentage of ride payments that fail on first attempt. Alert if above 2%.
     - `location_update_lag_seconds`: Delay between driver GPS timestamp and server receipt. Alert if p99 exceeds 10 seconds.
     - `surge_multiplier_by_zone`: Current surge levels across the city. Displayed on an operations map.
   - **Distributed tracing (OpenTelemetry):** Each ride request generates a trace ID that propagates through Matching Service → Location Service → Pricing Service → Ride Service → Payment Service. Traces are sampled at 10% for normal traffic, 100% for errors.
   - **Structured logging:** All services emit JSON logs with fields: `timestamp`, `service`, `trace_id`, `ride_id`, `level`, `message`. Logs are shipped to a centralized aggregator for search and correlation.
   - **Alerting rules:**
     - Critical: match_success_rate < 50% for 5 minutes → page on-call engineer.
     - Warning: payment_failure_rate > 2% for 10 minutes → Slack notification to payments team.
     - Info: surge_multiplier > 2.5 in any zone → log for business analytics.

</details>

---

## Problem 3 — Designing a Live Event Ticketing System

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design a live event ticketing platform that handles flash-sale traffic patterns where tens of thousands of users attempt to purchase tickets for a popular event within seconds of the sale opening. The system must guarantee that no ticket is sold twice (no overselling), provide a fair queuing mechanism for users, process payments reliably, and deliver electronic tickets after purchase. The design must address the extreme traffic spike at sale open, the inventory consistency challenge, and the end-to-end flow from queue entry to ticket delivery.

### Scenario

**Context:** A concert venue with 10,000 seats is selling tickets for a popular artist. Based on past events, 200,000 users will attempt to access the ticketing page within the first 60 seconds of the sale opening. The current system crashed during the last major sale — the web servers were overwhelmed, the database hit connection limits, and 300 tickets were oversold because concurrent purchase requests both read the same available count before either decremented it. The company lost $150,000 in refunds and suffered significant brand damage. The team needs a system that handles the traffic spike gracefully, prevents overselling with absolute certainty, and gives users a transparent, fair experience even when demand vastly exceeds supply.

**Requirements:** Design the queuing mechanism that absorbs the traffic spike and processes users in order. Explain how the system guarantees no overselling even under extreme concurrency. Describe the end-to-end flow from a user joining the queue to receiving their electronic ticket. Design the payment integration, including what happens when a user reaches the front of the queue but their payment fails. Explain how the system communicates queue position and estimated wait time to users. Address how the system scales to handle the initial traffic spike and then scales down after the rush.

**Expected Approach:** Place a virtual waiting room (queue) in front of the purchase flow to absorb the traffic spike. Use a centralized atomic counter (e.g., Redis DECR) or database-level row locking to guarantee inventory consistency. Implement a reservation-with-TTL pattern — when a user reaches the front of the queue, reserve their ticket for a limited time while they complete payment. If payment fails or times out, release the reservation back to the pool. Use auto-scaling for the web tier and CDN caching for static assets to handle the spike.

<details>
<summary>Hints</summary>

1. The virtual waiting room is a separate service that sits in front of the ticketing application. When the sale opens, all users are placed in a FIFO queue and assigned a position. The waiting room page is a lightweight static page (served from a CDN) that polls for the user's turn. Only a controlled number of users (e.g., 100 at a time) are released from the queue into the actual purchase flow. This converts an uncontrollable traffic spike into a steady, manageable stream.
2. For inventory consistency, the simplest bulletproof approach is an atomic decrement: `DECR available_tickets` in Redis. If the result is >= 0, the ticket is reserved. If the result is < 0, the tickets are sold out (and you INCR to restore the counter). This is a single atomic operation — no read-then-write race condition is possible.
3. The reservation TTL is critical. When a user is released from the queue, they have (for example) 5 minutes to complete payment. During this time, their ticket is reserved (counter decremented). If they do not complete payment within 5 minutes, the reservation expires: the counter is incremented, and the next user in the queue gets a chance. This prevents users from holding tickets indefinitely without paying.
4. Communicate queue position via a lightweight polling endpoint or Server-Sent Events. The waiting room page shows "You are #4,523 in line. Estimated wait: 8 minutes." The estimate is based on the current processing rate (users completing purchases per minute).

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement a virtual waiting room to shape traffic, an atomic inventory counter for oversell prevention, a reservation-with-TTL pattern for the purchase window, and auto-scaling infrastructure to handle the spike.

1. **System architecture:**
   - **CDN + Static Waiting Room Page:** The event page and waiting room UI are static HTML/JS served from a CDN. This absorbs the majority of the 200,000 concurrent connections without hitting the application servers. The page contains a JavaScript client that polls a lightweight Queue Status API.
   - **Queue Service:** Manages the virtual waiting room. When a user arrives, they are assigned a queue token (UUID) and position (monotonically increasing counter). The queue is stored in Redis as a sorted set (score = position). The service releases users in FIFO order at a controlled rate.
   - **Ticketing Service:** The core purchase flow. Only accessible to users who have been released from the queue (validated by their queue token). Handles ticket selection, reservation, and order creation.
   - **Inventory Service:** Manages the atomic ticket counter. A single Redis key `event:{event_id}:available` is initialized to the total ticket count (10,000). All inventory operations use atomic Redis commands.
   - **Payment Service:** Processes credit card charges via an external payment gateway. Integrated with the reservation TTL — if payment is not confirmed within the reservation window, the ticket is released.
   - **Ticket Delivery Service:** Generates electronic tickets (PDF or mobile pass) and delivers them via email and in-app notification after successful payment.

2. **End-to-end flow:**
   ```
   Phase 1 — Queue Entry (user perspective: "Waiting in line")
   1. Sale opens at 10:00 AM. User loads the event page (served from CDN).
   2. JavaScript client sends POST /queue/join {event_id, user_id} to the Queue Service.
   3. Queue Service assigns position #4,523 and returns a queue_token.
   4. Client polls GET /queue/status?token={queue_token} every 5 seconds.
      Response: {position: 4523, estimated_wait_minutes: 8, status: "waiting"}
   5. Queue Service releases users at a rate of 500/minute (based on the purchase flow's capacity).

   Phase 2 — Purchase Flow (user perspective: "Your turn to buy")
   6. When the user's position is reached, the status response changes to {status: "your_turn", purchase_url: "/purchase?token={queue_token}", expires_at: "10:09:00"}.
   7. Client redirects to the purchase page. The Ticketing Service validates the queue_token (one-time use, not expired).
   8. User selects ticket type and quantity (max 4 per user).
   9. Ticketing Service calls Inventory Service to reserve tickets:
      DECRBY event:{event_id}:available {quantity}
      If result >= 0: reservation successful. Store reservation in Redis with TTL of 5 minutes: SET reservation:{token} {quantity} EX 300.
      If result < 0: sold out. INCRBY to restore counter. Return "Sold Out" to user.
   10. User enters payment details. Ticketing Service calls Payment Service to charge the card.

   Phase 3 — Confirmation and Delivery
   11. Payment succeeds: Ticketing Service creates an order record (status: CONFIRMED), deletes the reservation key, and publishes an OrderConfirmed event.
   12. Ticket Delivery Service generates electronic tickets and sends them via email and push notification.
   13. User sees a confirmation page with a link to download their tickets.

   Failure Cases:
   - Payment fails: User gets 2 retry attempts within the 5-minute window. If all fail, the reservation expires and tickets are released (INCRBY to restore counter). User is notified and can rejoin the queue if tickets remain.
   - User abandons: The 5-minute TTL expires. A Redis keyspace notification or background job detects the expiry and restores the inventory counter.
   - Queue token reuse: Each token is marked as "consumed" after entering the purchase flow. Duplicate attempts are rejected.
   ```

3. **Oversell prevention guarantee:**
   - The inventory counter uses Redis atomic operations. `DECRBY` is a single atomic command — even if 1,000 requests execute simultaneously, Redis processes them sequentially. There is no read-then-write race condition.
   - The counter can temporarily go negative (if a DECRBY overshoots). The application checks the result: if negative, it immediately restores the counter with INCRBY and returns "sold out." The window of negativity is microseconds and has no real-world impact because no order is created.
   - As a defense-in-depth measure, the Order Service also checks a database-level constraint before confirming: `SELECT COUNT(*) FROM orders WHERE event_id = ? AND status = 'CONFIRMED'` must not exceed the venue capacity. This catches any edge case that bypasses the Redis counter.

4. **Queue position and wait time communication:**
   - The Queue Status API returns the user's current position and an estimated wait time.
   - Estimated wait = `(position - current_release_position) / release_rate_per_minute`.
   - The release rate is dynamically adjusted based on the purchase flow's throughput. If payments are slow (gateway latency), the release rate decreases to avoid overwhelming the payment service.
   - The polling interval is 5 seconds. For users near the front (position < 100), the interval decreases to 2 seconds for a more responsive experience.

5. **Scaling strategy:**
   - **Before the sale:** Pre-scale the web tier to handle 200,000 concurrent connections. The CDN handles static assets. The Queue Service and its Redis backend are provisioned for the expected load.
   - **During the spike (first 5 minutes):** The Queue Service absorbs all traffic. The Ticketing Service and Payment Service handle a controlled 500 users/minute. Auto-scaling adds Ticketing Service instances if the purchase completion rate drops.
   - **After the spike (5–30 minutes):** Traffic decreases as the queue drains. Auto-scaling removes excess instances. The Queue Service can be scaled down once the queue is empty.
   - **After sellout:** The event page is updated to show "Sold Out" (static CDN update). The Queue Service stops accepting new entries. Remaining queued users are notified that tickets are no longer available.

6. **Monitoring during the sale:**
   - Real-time dashboard showing: queue length, release rate, inventory remaining, payment success rate, average purchase completion time.
   - Alert if payment failure rate exceeds 10% (may indicate gateway issues — reduce release rate to avoid wasting reservations).
   - Alert if inventory counter goes negative (should self-correct immediately, but warrants investigation).
   - Alert if queue processing stalls (release rate drops to zero for more than 30 seconds).

</details>

---

## Problem 4 — Designing a Global Content Delivery Network for a Streaming Service

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design the content delivery architecture for a global video streaming service that serves millions of concurrent viewers across multiple continents. The system must minimize playback latency, handle regional traffic spikes (e.g., a popular show premiering in a specific timezone), ensure content availability even when origin servers or entire regions fail, and provide visibility into cache performance and viewer experience metrics. The design should address both video-on-demand (VOD) content and live streaming events.

### Scenario

**Context:** A streaming service has 50 million subscribers across North America, Europe, and Asia-Pacific. During peak evening hours, each region sees 5 million concurrent streams. The content library includes 100,000 VOD titles (movies and TV episodes) and occasional live events (sports, concerts) with 10+ million simultaneous viewers. The current architecture uses a single origin data center in the US, causing high latency for European and Asian viewers (300–500 ms round-trip). When a popular new season drops globally at midnight UTC, the origin servers are overwhelmed, causing buffering and playback failures. The team needs a globally distributed architecture that brings content closer to viewers, handles traffic spikes gracefully, and maintains high availability.

**Requirements:** Design the multi-tier caching architecture from origin to edge. Explain how VOD content is distributed and cached across regions. Describe how live streaming differs from VOD in terms of content distribution and caching. Design the failover strategy when an edge location or an entire region becomes unavailable. Explain how the system decides which content to cache at each tier (cache warming, eviction policies). Outline the observability strategy for monitoring cache hit rates, viewer experience (buffering ratio, startup time), and regional health.

**Expected Approach:** Implement a three-tier architecture: origin (authoritative content storage), mid-tier regional caches, and edge caches close to viewers. Use adaptive bitrate streaming (HLS/DASH) with content segmented into small chunks that are independently cacheable. For live streaming, push content to edge caches proactively rather than waiting for viewer requests. Implement DNS-based or anycast routing to direct viewers to the nearest healthy edge. Monitor cache hit rates and viewer QoE metrics per region.

<details>
<summary>Hints</summary>

1. Video content is segmented into small chunks (typically 2–10 seconds each) for adaptive bitrate streaming. Each chunk is an independent file that can be cached separately. A 2-hour movie at 5 bitrate levels might consist of 3,600 chunks × 5 bitrates = 18,000 cacheable objects. The manifest file (playlist) tells the player which chunks to request and in what order.
2. For VOD, a pull-based caching model works well: edge caches fetch chunks from mid-tier caches on first request, and mid-tier caches fetch from origin. Popular content naturally propagates to the edge through viewer demand. For live streaming, a push-based model is better: the origin pushes new chunks to mid-tier and edge caches as soon as they are encoded, so they are available before viewers request them.
3. Cache warming for anticipated demand: before a popular show premieres, pre-populate edge caches with the first few episodes. This avoids a thundering herd of cache misses at premiere time. Use viewership predictions (based on trailer views, social media buzz) to decide which content to warm and in which regions.
4. Failover at the edge: if an edge location fails, DNS or anycast routing automatically directs viewers to the next-nearest edge. The viewer's player may experience a brief rebuffering as it reconnects, but playback resumes from the new edge. Mid-tier caches provide a fallback if multiple edges fail simultaneously.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build a three-tier CDN architecture with origin, regional mid-tier, and edge caches. Use pull-based caching for VOD and push-based distribution for live. Implement intelligent cache warming, multi-layer failover, and comprehensive observability.

1. **Three-tier architecture:**
   ```
   [Origin] ←── [Mid-Tier Regional Caches] ←── [Edge Caches] ←── [Viewers]
      │                    │                        │
   US-East            US-West, EU, APAC        100+ edge locations
   (authoritative)    (regional aggregation)   (close to viewers)
   ```
   - **Origin (1–2 locations):** Stores the authoritative copy of all content. Handles encoding, transcoding, and DRM packaging. Located in a primary data center with a hot standby in a secondary region for disaster recovery.
   - **Mid-Tier Regional Caches (4–6 locations):** One per major region (US-West, US-East, EU-West, EU-East, APAC-North, APAC-South). Aggregates requests from multiple edge locations. Reduces load on origin by serving cache hits for content popular within the region.
   - **Edge Caches (100+ locations):** Deployed in ISP data centers and internet exchange points close to viewers. Each edge serves a metropolitan area or country. Handles the majority of viewer requests with sub-50 ms latency.

2. **VOD content distribution (pull-based):**
   ```
   1. Viewer in Paris requests chunk 47 of "Movie X" at 1080p.
   2. Request hits the nearest edge cache (Paris).
   3. Cache miss at Paris edge → request forwarded to EU-West mid-tier cache.
   4. Cache miss at EU-West mid-tier → request forwarded to origin.
   5. Origin returns the chunk. EU-West mid-tier caches it (TTL: 7 days). Paris edge caches it (TTL: 24 hours).
   6. Subsequent requests from Paris viewers are served from the Paris edge (cache hit).
   7. Requests from other EU cities (London, Berlin) hit their local edges, miss, and are served from the EU-West mid-tier (cache hit).
   ```
   - Cache eviction: LRU (Least Recently Used) at the edge, LFU (Least Frequently Used) at mid-tier. Edge caches prioritize recency (what's being watched now), mid-tier prioritizes popularity (what's watched often across the region).
   - TTLs are content-aware: new releases have shorter TTLs (in case of encoding fixes), catalog content has longer TTLs.

3. **Live streaming distribution (push-based):**
   ```
   1. Live event encoder produces a new 4-second chunk every 4 seconds.
   2. Origin immediately pushes the chunk to all mid-tier caches via persistent connections.
   3. Mid-tier caches push to all edge caches in their region.
   4. By the time viewers request the chunk (typically 4–8 seconds after encoding), it is already at the edge.
   5. Live chunks have a short TTL (60 seconds) and are evicted quickly after the live window passes.
   ```
   - For high-profile live events (10M+ viewers), the origin may push directly to edge caches, bypassing mid-tier to reduce latency.
   - Manifest files for live streams are updated every chunk interval. Edges cache the manifest with a very short TTL (2–4 seconds) to ensure viewers get the latest chunk list.

4. **Cache warming strategy:**
   - **Predictive warming:** Before a major premiere (e.g., new season of a popular series), the content team triggers a cache warm job. The first 3 episodes are pushed to all edge caches 2 hours before the premiere. This is based on historical data: 80% of premiere viewers watch the first episode, 60% continue to episode 2.
   - **Reactive warming:** If an edge cache sees a sudden spike in requests for content it does not have (e.g., a show goes viral on social media), it proactively fetches the next few episodes from mid-tier before viewers request them.
   - **Regional warming:** Content popular in one region is pre-warmed in adjacent regions. If a Korean drama is trending in APAC, it is warmed in US-West (large Korean diaspora) before demand materializes.

5. **Failover strategy:**
   - **Edge failure:** DNS-based routing (GeoDNS) or anycast directs viewers to the nearest healthy edge. Health checks run every 10 seconds. If an edge fails, DNS TTL (30 seconds) ensures viewers are redirected within a minute. The player's adaptive bitrate logic handles the brief interruption by rebuffering and resuming from the new edge.
   - **Mid-tier failure:** Edges fall back to the next-nearest mid-tier or directly to origin. Origin capacity is provisioned to handle 20% of total traffic (enough to survive one mid-tier region failing).
   - **Origin failure:** The hot standby origin in the secondary region takes over. DNS failover switches the origin endpoint within 60 seconds. Content replication between primary and secondary origin is synchronous for live streams, asynchronous (with <5 minute lag) for VOD.
   - **Regional isolation:** If an entire region is unreachable (e.g., undersea cable cut), viewers in that region are routed to the next-nearest region. Latency increases but playback continues.

6. **Observability strategy:**
   - **Cache metrics (per tier, per location):**
     - Cache hit rate: Target >95% at edge, >80% at mid-tier. Alert if edge hit rate drops below 90%.
     - Cache fill rate: Bytes fetched from upstream per second. Spike indicates cache misses (possibly a new release or cache eviction issue).
     - Storage utilization: Percentage of cache capacity used. Alert if >90% (risk of excessive eviction).
   - **Viewer experience metrics (collected from player telemetry):**
     - Startup time: Time from play button click to first frame rendered. Target <2 seconds. Alert if p95 exceeds 4 seconds.
     - Buffering ratio: Percentage of playback time spent buffering. Target <0.5%. Alert if >1%.
     - Bitrate stability: Frequency of bitrate switches during playback. Excessive switching indicates network instability or cache misses.
   - **Regional health dashboard:**
     - Map view showing cache hit rate, viewer count, and buffering ratio per edge location.
     - Color-coded: green (healthy), yellow (degraded), red (critical).
     - Drill-down to individual edge metrics and recent incidents.
   - **Alerting:**
     - Critical: Buffering ratio >2% in any region for 5 minutes → page on-call.
     - Warning: Cache hit rate <90% at any edge for 10 minutes → notify CDN team.
     - Info: New content trending (request rate >10x baseline) → log for capacity planning.

</details>

---

## Problem 5 — Designing a Healthcare Appointment and Telemedicine Platform

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a healthcare appointment and telemedicine platform that allows patients to book in-person or video appointments with doctors, conducts real-time video consultations, processes insurance-based and out-of-pocket payments, delivers prescriptions electronically, and maintains a complete audit trail for regulatory compliance. The system must integrate multiple complex domains — scheduling, real-time communication, payment processing, notification delivery, and data privacy — into a cohesive, secure architecture.

### Scenario

**Context:** A healthcare startup is building a platform serving 500 clinics and 2,000 doctors across a country. Patients use a mobile app to search for doctors by specialty and location, view available time slots, book appointments (in-person or telemedicine), conduct video consultations, receive prescriptions, and pay their copay or full fee. Doctors use a web dashboard to manage their schedule, conduct video visits, write prescriptions, and view patient history. The platform must comply with healthcare data privacy regulations (e.g., HIPAA in the US or GDPR in the EU), meaning all patient health information (PHI) must be encrypted at rest and in transit, access must be logged, and data retention policies must be enforced. The system handles 10,000 appointments per day, with 40% being telemedicine visits. During flu season, appointment volume can triple within a week.

**Requirements:** Design the service architecture covering appointment scheduling, real-time video consultations, payment processing (insurance and self-pay), electronic prescriptions, and notifications. Explain how the scheduling system handles concurrent booking attempts for the same time slot. Describe the real-time video consultation flow, including how the system handles network degradation during a call. Design the payment flow that supports both insurance-based billing (claim submission) and direct patient payment. Explain how the system ensures regulatory compliance for patient data (encryption, access logging, data retention). Outline the monitoring strategy, including both technical health metrics and business metrics (appointment completion rate, no-show rate, average wait time for telemedicine).

**Expected Approach:** Decompose into domain services: Scheduling Service, Video Service, Payment/Billing Service, Prescription Service, Notification Service, and a Patient Records Service with strict access controls. Use optimistic locking or database constraints to prevent double-booking. Integrate a WebRTC-based video service with TURN/STUN servers for NAT traversal. Implement a dual payment path: insurance claims are submitted asynchronously via an EDI gateway, while patient copays are charged synchronously. Encrypt all PHI at rest (AES-256) and in transit (TLS 1.3), with field-level encryption for sensitive data. Log all data access events to an immutable audit log.

<details>
<summary>Hints</summary>

1. Appointment scheduling with concurrent booking prevention is similar to the inventory reservation problem in the ticketing system (Problem 3). When a patient selects a time slot, the system must atomically check availability and create the booking. Use a database unique constraint on `(doctor_id, slot_start_time)` or an optimistic locking pattern where the slot has a version number that must match at booking time.
2. For telemedicine video, WebRTC provides peer-to-peer video with low latency. However, in practice, many patients are behind restrictive NATs or firewalls. TURN servers relay video traffic when direct peer-to-peer connections fail. The platform should operate its own TURN servers (or use a managed service) to ensure reliable connectivity. If video quality degrades (detected by packet loss metrics), the system should automatically reduce resolution or switch to audio-only.
3. Insurance billing is fundamentally different from direct payment. Insurance claims are submitted electronically (via EDI 837 format in the US) to the insurance company, which adjudicates the claim (approves, partially approves, or denies) days or weeks later. The platform must track claim status and handle denials (rebill the patient or appeal). This is an asynchronous, long-running workflow — very different from the synchronous credit card charge.
4. For regulatory compliance, think in terms of defense in depth: encrypt data at rest and in transit, enforce role-based access control (a receptionist can see appointment times but not clinical notes), log every access to PHI in an immutable audit log, and implement data retention policies (e.g., retain records for 7 years, then purge). The audit log itself must be tamper-proof — consider append-only storage with cryptographic chaining.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build a domain-driven microservices architecture with strict data isolation between services, defense-in-depth security for PHI, dual payment paths for insurance and self-pay, WebRTC-based video with adaptive quality, and comprehensive audit logging.

1. **Service architecture:**
   - **API Gateway:** Authenticates users (patients, doctors, clinic staff) via OAuth 2.0 + JWT. Enforces role-based access control (RBAC). Terminates TLS. Routes requests to internal services.
   - **Scheduling Service:** Manages doctor availability and appointment bookings. Stores time slots, handles concurrent booking prevention, sends reminders, and tracks appointment status (scheduled, checked-in, in-progress, completed, no-show, cancelled).
   - **Video Service:** Manages telemedicine video sessions. Provisions WebRTC connections, operates TURN/STUN servers, monitors call quality, and handles recording (if consent is given). Integrates with a third-party WebRTC infrastructure or a self-hosted media server.
   - **Patient Records Service:** Stores patient demographics, medical history, and clinical notes. All data is encrypted at rest with AES-256. Field-level encryption for highly sensitive fields (SSN, diagnosis codes). Access is strictly controlled by RBAC and logged.
   - **Prescription Service:** Allows doctors to create electronic prescriptions. Validates drug interactions against a formulary database. Transmits prescriptions to pharmacies via an e-prescribing network (e.g., Surescripts in the US).
   - **Billing Service:** Handles both insurance claims and patient payments. Submits insurance claims via EDI, tracks adjudication status, and processes patient copays or self-pay charges via a payment gateway.
   - **Notification Service:** Sends appointment reminders (24 hours and 1 hour before), telemedicine join links, prescription confirmations, and billing statements via SMS, email, and push notification.
   - **Audit Service:** Immutable, append-only log of all access to PHI. Every read, write, or delete of patient data is recorded with timestamp, user ID, action, resource, and IP address.

2. **Concurrent booking prevention:**
   ```sql
   -- Doctor availability is stored as time slots:
   CREATE TABLE appointment_slots (
       slot_id BIGSERIAL PRIMARY KEY,
       doctor_id BIGINT NOT NULL,
       slot_start TIMESTAMP NOT NULL,
       slot_end TIMESTAMP NOT NULL,
       status VARCHAR(20) NOT NULL DEFAULT 'AVAILABLE',  -- AVAILABLE, BOOKED, BLOCKED
       version INT NOT NULL DEFAULT 0,
       UNIQUE (doctor_id, slot_start)
   );

   -- Booking uses optimistic locking:
   BEGIN;
   SELECT slot_id, version FROM appointment_slots
       WHERE doctor_id = ? AND slot_start = ? AND status = 'AVAILABLE';
   -- Application checks that a row was returned
   UPDATE appointment_slots SET status = 'BOOKED', version = version + 1
       WHERE slot_id = ? AND version = ?;
   -- If UPDATE affects 0 rows, another booking won the race → return "Slot no longer available"
   -- If UPDATE affects 1 row, booking succeeds
   INSERT INTO appointments (patient_id, doctor_id, slot_id, type, status) VALUES (...);
   COMMIT;
   ```
   - The `UNIQUE (doctor_id, slot_start)` constraint provides a database-level guarantee against double-booking even if the application logic has a bug.
   - For telemedicine appointments, the Scheduling Service also provisions a video room ID (from the Video Service) and includes the join link in the appointment confirmation.

3. **Telemedicine video consultation flow:**
   ```
   1. 15 minutes before the appointment, Notification Service sends a "Join your video visit" link to the patient (SMS + push) and a reminder to the doctor (dashboard notification).
   2. Patient clicks the link → mobile app opens the Video Service client.
   3. Video Service provisions a WebRTC session:
      a. Client fetches ICE server configuration (STUN + TURN server addresses).
      b. Client attempts a direct peer-to-peer connection via STUN.
      c. If direct connection fails (NAT/firewall), falls back to TURN relay.
   4. Doctor joins from the web dashboard. Both participants are connected to the same session.
   5. During the call, the Video Service monitors quality metrics:
      - Packet loss > 5%: Reduce video resolution from 720p to 480p.
      - Packet loss > 15%: Switch to audio-only mode with a static image.
      - Connection lost: Attempt reconnection for 30 seconds. If unsuccessful, notify both parties and offer to rejoin or reschedule.
   6. Doctor can share their screen (e.g., to show test results) using WebRTC screen sharing.
   7. When the consultation ends, the doctor clicks "End Visit," which triggers:
      - Appointment status → COMPLETED.
      - If recording was enabled (with patient consent), the recording is encrypted and stored in the Patient Records Service.
      - Doctor can write clinical notes and prescriptions in the dashboard.
   ```

4. **Dual payment path:**
   - **Insurance-based billing (asynchronous):**
     ```
     1. After the appointment, the Billing Service generates an insurance claim with diagnosis codes (ICD-10), procedure codes (CPT), and patient/provider information.
     2. The claim is submitted electronically via an EDI 837 transaction to the patient's insurance company through a clearinghouse.
     3. The insurance company adjudicates the claim (typically 7–30 days):
        - Approved: Insurance pays the allowed amount. Patient owes the copay/coinsurance.
        - Partially approved: Insurance pays a reduced amount. Patient owes the remainder.
        - Denied: Patient owes the full amount (or the clinic appeals).
     4. The Billing Service receives the adjudication result via an EDI 835 remittance advice.
     5. If the patient owes a balance, the Billing Service charges the patient's card on file or sends a billing statement via the Notification Service.
     ```
   - **Self-pay (synchronous):**
     ```
     1. At booking time, the patient sees the consultation fee (e.g., $150 for a telemedicine visit).
     2. At checkout (after the appointment), the Billing Service charges the patient's card via the payment gateway.
     3. If payment fails, the patient receives a billing statement with a link to pay online. Retry attempts are made at 3, 7, and 14 days.
     ```
   - The Billing Service maintains a ledger (similar to the Payment System Design module) tracking all charges, payments, adjustments, and refunds per patient and per appointment.

5. **Regulatory compliance (defense in depth):**
   - **Encryption at rest:** All databases storing PHI use AES-256 encryption (transparent data encryption at the storage layer). Highly sensitive fields (SSN, diagnosis) use application-level field encryption with keys managed in a dedicated key management service (KMS). Keys are rotated annually.
   - **Encryption in transit:** All service-to-service communication uses mutual TLS (mTLS). External API traffic uses TLS 1.3. Video streams are encrypted end-to-end via WebRTC's built-in DTLS-SRTP.
   - **Access control:** RBAC enforced at the API Gateway and within each service:
     - Patient: Can view their own records, appointments, and bills.
     - Doctor: Can view records of patients they have an appointment with. Cannot view other patients.
     - Receptionist: Can view appointment schedules and patient demographics. Cannot view clinical notes or diagnoses.
     - Admin: Can manage users and system configuration. Cannot view PHI without explicit audit justification.
   - **Audit logging:** Every access to PHI is logged to the Audit Service:
     ```json
     {
       "timestamp": "2024-03-15T14:23:01Z",
       "user_id": "doctor_4521",
       "action": "READ",
       "resource": "patient_record",
       "resource_id": "patient_78901",
       "fields_accessed": ["demographics", "medical_history"],
       "ip_address": "203.0.113.42",
       "reason": "appointment_12345"
     }
     ```
     Audit logs are stored in append-only storage with cryptographic chaining (each entry includes a hash of the previous entry) to detect tampering. Retained for 7 years per regulatory requirements.
   - **Data retention:** Patient records are retained for the legally required period (7 years in most US states). After expiry, records are purged via a scheduled job that also logs the deletion in the audit trail. Patients can request data export (right of access) or deletion (where legally permitted) through the app.

6. **Monitoring strategy:**
   - **Technical health metrics:**
     - Service latency (p50, p95, p99) per endpoint. Alert if scheduling API p99 exceeds 500 ms.
     - Video call connection success rate. Alert if below 95%.
     - Video quality score (based on packet loss and bitrate). Alert if average score drops below 3.5/5.
     - Payment gateway response time and failure rate. Alert if failure rate exceeds 3%.
     - Database connection pool utilization. Alert if above 80%.
   - **Business metrics:**
     - Appointment completion rate: Percentage of scheduled appointments that reach COMPLETED status. Target >85%.
     - No-show rate: Percentage of appointments where the patient does not check in. Alert if above 15% (may indicate reminder system failure).
     - Telemedicine adoption rate: Percentage of appointments that are video visits vs. in-person.
     - Average wait time for telemedicine: Time from scheduled start to when the doctor joins. Target <5 minutes.
     - Insurance claim denial rate: Percentage of claims denied on first submission. Alert if above 10% (may indicate coding errors).
   - **Compliance monitoring:**
     - Failed access attempts (user tried to access a resource they are not authorized for). Alert on any occurrence for investigation.
     - Unusual access patterns (e.g., a doctor viewing 50+ patient records in an hour). Alert for potential data breach investigation.
     - Encryption key age. Alert if any key has not been rotated in 13 months.

</details>
