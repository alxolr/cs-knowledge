# Microservices Architecture

**Track:** Design Concepts
**Difficulty Tier:** Intermediate
**Prerequisites:** [Load Balancing](../load-balancing/README.md), [Message Queues](../message-queues/README.md)

## Concept Overview

Microservices architecture is a design approach that structures an application as a collection of small, independently deployable services, each owning a specific business capability. Instead of building one large monolithic application where all features share a single codebase, database, and deployment pipeline, microservices decompose the system into loosely coupled services that communicate over well-defined APIs. Each service can be developed, tested, deployed, and scaled independently by a small team, enabling organizations to move faster and isolate failures to individual components rather than bringing down the entire system.

The boundaries between microservices are typically drawn along business domain lines — an e-commerce platform might have separate services for catalog, ordering, payments, and shipping. Each service owns its data store (the "database per service" pattern), which prevents tight coupling through shared tables but introduces the challenge of maintaining data consistency across services. Communication between services happens either synchronously (HTTP/REST, gRPC) or asynchronously (message queues, event streams), and the choice between these patterns has significant implications for latency, coupling, and fault tolerance.

Operating a microservices system requires supporting infrastructure that a monolith does not need. An API gateway provides a single entry point for external clients, routing requests to the appropriate internal service and handling cross-cutting concerns like authentication, rate limiting, and response aggregation. Service discovery allows services to locate each other dynamically as instances scale up and down. Distributed tracing and centralized logging become essential for debugging requests that span multiple services. The saga pattern replaces traditional database transactions when a business operation spans several services, coordinating a sequence of local transactions with compensating actions for rollback.

Microservices are not free — they trade the complexity of a large codebase for the complexity of a distributed system. Network failures, partial outages, data inconsistency, and operational overhead are inherent challenges. The architecture is most beneficial when the organization is large enough to staff independent teams per service, when different services have different scaling requirements, and when the rate of change varies significantly across business domains.

---

## Problem 1 — Decomposing a Monolith into Services for an Online Bookstore

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

An online bookstore runs as a single monolithic application. The team wants to break it into microservices to allow independent scaling and faster feature development. Identify appropriate service boundaries, explain the reasoning behind each boundary, and describe how the services will communicate for a typical user workflow (browsing, adding to cart, and placing an order).

### Scenario

**Context:** The bookstore monolith handles catalog browsing, user accounts, shopping cart management, order processing, payment handling, and email notifications — all in one codebase backed by a single relational database. The catalog team ships features weekly, but every deployment requires redeploying the entire application, including the payment module, which changes only once a quarter. During holiday sales, the catalog browsing load increases 10× while order processing load only doubles, yet the entire monolith must be scaled uniformly.

**Requirements:** Propose a set of microservices with clear boundaries. Justify why each boundary was chosen. Describe the data each service owns. Outline the communication flow when a customer browses the catalog, adds a book to their cart, and places an order. Identify which communications can be synchronous and which should be asynchronous.

**Expected Approach:** Identify services aligned with business capabilities: Catalog Service, User Service, Cart Service, Order Service, Payment Service, and Notification Service. Each service owns its own data store. Catalog browsing is synchronous (low latency needed), while payment confirmation and email notification are asynchronous (decoupled via message queue). Justify boundaries by differing change rates, scaling needs, and team ownership.

<details>
<summary>Hints</summary>

1. A good service boundary aligns with a business capability that has its own data, its own rate of change, and can be understood by a single team. If two capabilities always change together, they probably belong in the same service.
2. The catalog needs to scale independently from payments — this is a strong signal for separate services.
3. Notifications (email confirmations) do not need to happen in real time. A message queue between the Order Service and Notification Service decouples them and adds resilience.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Decompose along business capability boundaries, assign each service its own data store, and choose sync or async communication based on latency and coupling requirements.

1. **Service identification:**
   - **Catalog Service:** Manages book listings, descriptions, pricing, and search. Owns the catalog database. Scales independently to handle browsing spikes.
   - **User Service:** Manages user accounts, authentication, and profiles. Owns the user database.
   - **Cart Service:** Manages shopping cart state per user. Owns a lightweight store (e.g., Redis) for cart data.
   - **Order Service:** Manages order creation, status tracking, and order history. Owns the orders database.
   - **Payment Service:** Handles payment processing and integrates with external payment gateways. Owns payment records. Changes infrequently and has strict compliance requirements.
   - **Notification Service:** Sends email confirmations, shipping updates, and promotional messages. Owns notification templates and delivery logs.

2. **Boundary justification:**
   - Catalog vs. Order: different scaling profiles (10× vs. 2× during sales), different change rates (weekly vs. quarterly).
   - Payment as a separate service: strict compliance and audit requirements, infrequent changes, and the need to isolate payment failures from the rest of the system.
   - Notification as a separate service: purely asynchronous, no user-facing latency requirement, and can be retried independently on failure.

3. **Communication flow for browse → cart → order:**
   - **Browse:** Client → Catalog Service (synchronous REST). Returns book details directly.
   - **Add to cart:** Client → Cart Service (synchronous REST). Cart Service may call Catalog Service synchronously to validate the book ID and fetch the current price.
   - **Place order:** Client → Order Service (synchronous REST). Order Service calls Payment Service synchronously to charge the customer. On success, Order Service publishes an "OrderPlaced" event to a message queue. Notification Service consumes the event asynchronously and sends a confirmation email.

4. **Data ownership:** Each service has its own database. No shared tables. If the Order Service needs book details for an order record, it stores a snapshot of the book title and price at order time rather than querying the Catalog Service later.

</details>

---

## Problem 2 — Choosing Between Synchronous and Asynchronous Inter-Service Communication

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

A ride-sharing platform has decomposed its backend into microservices. Several workflows require coordination between services, but the team is unsure when to use synchronous HTTP calls versus asynchronous messaging. Design the communication strategy for three specific workflows, choosing the appropriate pattern for each and justifying the trade-offs.

### Scenario

**Context:** The ride-sharing platform has the following services: Ride Service (manages ride requests and matching), Driver Service (manages driver availability and location), Pricing Service (calculates fare estimates), Payment Service (charges riders after a trip), and Notification Service (sends push notifications to riders and drivers). Three workflows need communication design: (1) A rider requests a fare estimate before booking — the Ride Service needs a price from the Pricing Service. (2) A ride completes and the rider must be charged — the Ride Service needs to trigger payment. (3) After payment succeeds, both the rider and driver should receive a trip summary notification.

**Requirements:** For each of the three workflows, choose synchronous or asynchronous communication and justify the choice. Describe what happens when the downstream service is unavailable in each case. Explain how the choice affects latency, coupling, and fault tolerance. Address whether any workflow benefits from a hybrid approach (synchronous call with an asynchronous fallback).

**Expected Approach:** Workflow 1 (fare estimate) is synchronous because the rider is waiting for the price in real time. Workflow 2 (payment after ride) is asynchronous because the rider does not need to wait for payment processing to complete — the Ride Service publishes a "RideCompleted" event and the Payment Service processes it from the queue. Workflow 3 (notifications) is asynchronous because notifications are fire-and-forget and can tolerate delays. Discuss retry strategies and dead-letter queues for async failures, and circuit breakers for sync failures.

<details>
<summary>Hints</summary>

1. Synchronous communication is appropriate when the caller needs the response immediately to continue its workflow (e.g., displaying a fare estimate to the user). The trade-off is tight temporal coupling — if the downstream service is slow or down, the caller is blocked.
2. Asynchronous communication is appropriate when the caller does not need an immediate response and the operation can be processed later. The trade-off is eventual consistency — the caller does not know immediately whether the operation succeeded.
3. For synchronous calls to unreliable services, a circuit breaker pattern prevents cascading failures: after N consecutive failures, the circuit "opens" and the caller fails fast (or returns a cached/default response) instead of waiting for timeouts.
4. For asynchronous messaging, a dead-letter queue captures messages that fail processing after several retries, preventing poison messages from blocking the queue.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Match each workflow to the communication pattern that fits its latency and coupling requirements, and design failure handling for each.

1. **Workflow 1 — Fare estimate (synchronous):**
   - The rider opens the app and requests a fare estimate. The Ride Service calls the Pricing Service via synchronous HTTP (REST or gRPC).
   - **Why synchronous:** The rider is waiting on-screen for the price. The response is needed in under 500 ms to feel responsive. There is no useful action the Ride Service can take without the price.
   - **Failure handling:** If the Pricing Service is unavailable, the Ride Service uses a circuit breaker. After 3 consecutive failures or timeouts (2-second timeout), the circuit opens and the Ride Service returns a cached fare estimate (last known price for the route) with a disclaimer ("estimated fare, final price may vary"). The circuit half-opens after 30 seconds to test if the Pricing Service has recovered.
   - **Trade-off:** Tight coupling to the Pricing Service's availability, mitigated by caching and circuit breaker.

2. **Workflow 2 — Post-ride payment (asynchronous):**
   - When a ride completes, the Ride Service publishes a "RideCompleted" event to a message queue containing the ride ID, rider ID, driver ID, and final fare.
   - The Payment Service consumes the event, charges the rider's payment method, and publishes a "PaymentProcessed" event (or "PaymentFailed" event).
   - **Why asynchronous:** The rider has already exited the car. There is no screen waiting for payment confirmation. Payment can be processed seconds or even minutes later without degrading the user experience.
   - **Failure handling:** If the Payment Service is temporarily down, the message remains in the queue and is retried with exponential backoff (1s, 2s, 4s, up to 5 retries). After all retries fail, the message moves to a dead-letter queue for manual investigation. The Ride Service is completely unaffected by Payment Service downtime.
   - **Trade-off:** The rider does not see immediate payment confirmation in the app. The Ride Service must handle the case where payment eventually fails (e.g., notify the rider later, flag the account).

3. **Workflow 3 — Trip summary notifications (asynchronous):**
   - The Payment Service publishes a "PaymentProcessed" event. The Notification Service consumes it and sends push notifications with the trip summary to both rider and driver.
   - **Why asynchronous:** Notifications are informational and non-critical. A 10-second delay is acceptable. If the Notification Service is down, the trip is not affected — notifications can be sent when the service recovers.
   - **Failure handling:** Same retry and dead-letter queue pattern as Workflow 2. Additionally, the Notification Service is idempotent — processing the same event twice sends the notification once (deduplicated by ride ID).
   - **Trade-off:** Notifications may arrive with a slight delay. In rare failure cases, a notification might not be sent at all (acceptable for this use case).

4. **Summary of patterns:**
   | Workflow | Pattern | Latency | Coupling | Fault Tolerance |
   |----------|---------|---------|----------|-----------------|
   | Fare estimate | Synchronous | Low (real-time) | High | Circuit breaker + cache |
   | Payment | Asynchronous | Higher (eventual) | Low | Retry + dead-letter queue |
   | Notification | Asynchronous | Higher (eventual) | Low | Retry + idempotency |

</details>

---

## Problem 3 — Designing an API Gateway for a Multi-Service Backend

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

A SaaS project management tool has been decomposed into microservices, but the frontend team is struggling with the complexity of calling multiple backend services directly. Design an API gateway that provides a unified entry point for the frontend, handles cross-cutting concerns, and aggregates responses from multiple services into a single API call where needed.

### Scenario

**Context:** The project management tool has five backend services: Project Service, Task Service, User Service, Comment Service, and File Service. The single-page application (SPA) frontend currently makes direct HTTP calls to each service. Loading the project dashboard requires four separate API calls: one to the Project Service for project details, one to the Task Service for recent tasks, one to the User Service for team member info, and one to the Comment Service for recent comments. This results in high latency on the client (four sequential round trips), CORS configuration headaches (five different origins), and duplicated authentication logic in every service. The team wants a single entry point that simplifies the frontend's interaction with the backend.

**Requirements:** Design an API gateway that sits between the frontend and the backend services. Define the gateway's responsibilities: request routing, authentication, response aggregation, and rate limiting. Describe how the gateway routes requests to the correct backend service based on URL path. Design an aggregation endpoint that combines data from multiple services into a single response for the dashboard. Address how the gateway handles partial failures (e.g., the Comment Service is down but the other three services respond). Explain how authentication is handled at the gateway level so individual services do not need to implement it.

**Expected Approach:** The gateway exposes a unified API (e.g., `/api/projects/{id}`, `/api/tasks`, `/api/users/{id}`). It authenticates requests by validating JWT tokens before forwarding to backend services. For the dashboard, a `/api/dashboard/{projectId}` aggregation endpoint fans out requests to Project, Task, User, and Comment services in parallel, merges the responses, and returns a single JSON payload. If one service fails, the gateway returns partial data with a degraded flag rather than failing the entire request.

<details>
<summary>Hints</summary>

1. The API gateway acts as a reverse proxy with added intelligence. Simple requests are routed 1:1 to a backend service based on the URL path prefix (e.g., `/api/tasks/*` → Task Service). Aggregation endpoints fan out to multiple services and merge responses.
2. Authentication at the gateway means the gateway validates the JWT token on every incoming request. If valid, it forwards the request to the backend service with the user's identity in a trusted header (e.g., `X-User-Id`). Backend services trust this header because only the gateway can set it (internal network only).
3. For partial failures in aggregation, use a timeout per backend call (e.g., 2 seconds). If a service does not respond in time, the gateway returns the data it has with a field indicating which sections are unavailable (e.g., `"comments": null, "degraded": ["comments"]`).
4. Rate limiting at the gateway protects all backend services with a single policy. Per-user rate limits (e.g., 100 requests/minute) prevent abuse without requiring each service to implement its own limiter.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Deploy an API gateway as the single entry point for the frontend, handling routing, authentication, aggregation, and rate limiting centrally.

1. **Request routing:**
   - The gateway maps URL path prefixes to backend services:
     - `/api/projects/*` → Project Service
     - `/api/tasks/*` → Task Service
     - `/api/users/*` → User Service
     - `/api/comments/*` → Comment Service
     - `/api/files/*` → File Service
   - For simple CRUD operations, the gateway proxies the request directly to the target service, forwarding headers and the request body unchanged.

2. **Authentication:**
   - Every incoming request must include a JWT token in the `Authorization` header.
   - The gateway validates the token (checks signature, expiration, and issuer) before routing the request.
   - If the token is invalid or missing, the gateway returns `401 Unauthorized` immediately — the request never reaches a backend service.
   - On successful validation, the gateway extracts the user ID and role from the token and adds trusted headers (`X-User-Id`, `X-User-Role`) to the forwarded request. Backend services use these headers for authorization decisions without re-validating the token.

3. **Response aggregation for the dashboard:**
   - The gateway exposes a custom endpoint: `GET /api/dashboard/{projectId}`.
   - On receiving this request, the gateway fans out four parallel requests:
     - Project Service: `GET /projects/{projectId}` (project details)
     - Task Service: `GET /tasks?projectId={projectId}&limit=10` (recent tasks)
     - User Service: `GET /users?projectId={projectId}` (team members)
     - Comment Service: `GET /comments?projectId={projectId}&limit=5` (recent comments)
   - The gateway waits for all four responses (with a 2-second timeout per call), merges them into a single JSON response, and returns it to the frontend.

4. **Partial failure handling:**
   - If the Comment Service times out or returns an error, the gateway still returns the project, task, and user data.
   - The response includes a `degraded` array listing unavailable sections:
     ```json
     {
       "project": { ... },
       "tasks": [ ... ],
       "members": [ ... ],
       "comments": null,
       "degraded": ["comments"]
     }
     ```
   - The frontend renders available sections and shows a "Comments temporarily unavailable" message for degraded sections.

5. **Rate limiting:**
   - The gateway enforces per-user rate limits: 100 requests per minute for standard users, 500 for premium users.
   - Rate limit state is stored in a shared Redis instance so that all gateway instances enforce the same limits.
   - When a user exceeds the limit, the gateway returns `429 Too Many Requests` with a `Retry-After` header.

6. **Benefits over direct service calls:**
   - Frontend makes one call instead of four for the dashboard (reduced latency and complexity).
   - CORS is configured once at the gateway, not in five services.
   - Authentication logic lives in one place, reducing duplication and the risk of inconsistent enforcement.

</details>

---

## Problem 4 — Designing a Service Discovery Mechanism for Dynamic Scaling

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

A microservices platform runs on a container orchestration system where service instances are created and destroyed frequently as traffic fluctuates. Hardcoded service addresses are no longer viable. Design a service discovery mechanism that allows services to find each other dynamically, handles instance registration and deregistration, and remains resilient when the discovery system itself experiences issues.

### Scenario

**Context:** A food delivery platform runs 15 microservices on a container orchestration platform. During lunch and dinner peaks, the Order Service scales from 5 to 40 instances, and the Restaurant Service scales from 3 to 20 instances. Instances receive dynamically assigned IP addresses and ports each time they start. Currently, service URLs are stored in configuration files that must be manually updated whenever instances change — this causes stale routing, failed requests, and deployment delays. The team needs a dynamic service discovery solution that keeps pace with the rapid scaling.

**Requirements:** Design a service discovery system that supports automatic registration of new instances, deregistration of terminated instances, and real-time lookup of healthy instances by service name. Choose between client-side and server-side discovery and justify the choice. Describe how health checks ensure that only healthy instances are returned in lookups. Address what happens when the service registry itself becomes unavailable — how do services continue to route requests? The design should handle the scenario where an instance crashes without gracefully deregistering.

**Expected Approach:** Use a service registry (e.g., Consul, etcd, or a built-in orchestrator registry) where each instance registers itself on startup and deregisters on shutdown. Health checks (HTTP or TCP probes) run periodically to detect crashed instances that did not deregister. Choose client-side discovery (services query the registry and load-balance locally) for lower latency and fewer single points of failure, or server-side discovery (a load balancer queries the registry) for simpler client logic. Address registry unavailability with local caching of the last known instance list and a TTL-based fallback.

<details>
<summary>Hints</summary>

1. In client-side discovery, each service instance queries the registry to get a list of target instances and performs load balancing locally (e.g., round-robin across the returned list). This avoids a centralized load balancer as a bottleneck but requires every service to include discovery client logic.
2. In server-side discovery, a load balancer or router sits between the caller and the target service. The load balancer queries the registry and routes requests. The caller only needs to know the load balancer's address. This is simpler for clients but adds a network hop and a potential single point of failure.
3. Health checks should be active (the registry probes instances) rather than passive (instances self-report). An instance that crashes cannot self-report its failure, but an active probe will detect the missing heartbeat.
4. When the registry is unavailable, services should fall back to a locally cached copy of the instance list. The cache has a TTL (e.g., 30 seconds). If the registry is down for longer than the TTL, the cached list may become stale, but it is better than having no routing information at all.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Deploy a distributed service registry with client-side discovery, active health checks, and local caching for resilience.

1. **Service registry:**
   - Deploy a distributed, highly available registry cluster (e.g., three Consul nodes or the orchestrator's built-in service registry).
   - Each microservice instance registers itself on startup by sending its service name, instance ID, IP address, port, and metadata (version, region) to the registry.
   - On graceful shutdown, the instance sends a deregistration request to remove itself from the registry.

2. **Client-side discovery:**
   - When Service A needs to call Service B, Service A's discovery client queries the registry: "Give me all healthy instances of Service B."
   - The registry returns a list of IP:port pairs. Service A's client-side load balancer selects one using round-robin or least-connections.
   - **Why client-side:** Eliminates a centralized load balancer as a bottleneck and single point of failure. Each service makes its own routing decisions. In a system with 15 services and up to 40 instances each, a centralized discovery load balancer would need to handle enormous query volume. Client-side discovery distributes this load.

3. **Health checks for crash detection:**
   - The registry runs active health checks against every registered instance: an HTTP GET to `/health` every 10 seconds.
   - If an instance fails 3 consecutive health checks (30 seconds of unresponsiveness), the registry marks it as unhealthy and excludes it from query results.
   - This handles the crash scenario: an instance that terminates abruptly without deregistering is detected within 30 seconds and removed from the pool.
   - When the instance recovers (or a new instance takes its place), it re-registers and begins passing health checks, automatically appearing in query results.

4. **Registry unavailability — local caching:**
   - Each service's discovery client maintains a local cache of the last successful registry response for each target service.
   - The cache has a TTL of 30 seconds. Under normal operation, the client refreshes the cache every 15 seconds (before the TTL expires).
   - If the registry is unreachable, the client uses the cached instance list until the TTL expires. After TTL expiration, the client continues using the stale cache but logs warnings and triggers alerts.
   - This ensures that a brief registry outage (under 30 seconds) has zero impact on service-to-service communication. A longer outage degrades gracefully — routing continues with potentially stale data, which is better than no routing at all.

5. **Handling stale cache entries:**
   - If a cached instance is actually down, the calling service will experience a connection failure. The client-side load balancer retries the request on a different instance from the cached list.
   - After 2 consecutive failures to a specific instance, the client removes it from the local cache (circuit breaker per instance).
   - This self-healing behavior means that even with a stale cache, unhealthy instances are quickly excluded from local routing.

6. **Registration lifecycle summary:**
   | Event | Action | Detection Time |
   |-------|--------|---------------|
   | Instance starts | Self-registers with registry | Immediate |
   | Instance stops gracefully | Self-deregisters | Immediate |
   | Instance crashes | Registry health check fails | ~30 seconds |
   | Registry unavailable | Clients use local cache | 0 (seamless fallback) |

</details>

---

## Problem 5 — Implementing the Saga Pattern for a Distributed Order Workflow

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

A travel booking platform processes reservations that span multiple microservices — flights, hotels, and car rentals. A single booking must either succeed across all three services or roll back entirely. Traditional database transactions cannot span multiple services. Design a saga that coordinates the distributed booking workflow, handles partial failures with compensating actions, and ensures the system does not leave bookings in an inconsistent state.

### Scenario

**Context:** The travel platform has three independent microservices: Flight Service, Hotel Service, and Car Rental Service. Each service owns its own database. When a customer books a vacation package, the system must reserve a flight, a hotel room, and a rental car. If any reservation fails (e.g., the hotel is fully booked), the previously successful reservations must be cancelled to avoid charging the customer for a partial trip. Currently, the team attempts to call all three services in sequence and manually handles failures, but race conditions and partial failures frequently leave orphaned reservations that require manual cleanup.

**Requirements:** Design a saga (choose between orchestration and choreography) to coordinate the three-step booking process. Define the sequence of steps and the compensating action for each step. Describe how the saga handles a failure at each stage (flight succeeds but hotel fails; flight and hotel succeed but car rental fails). Address idempotency — what happens if a compensating action is retried? Explain how the system tracks the saga's state so that it can recover if the saga coordinator itself crashes mid-execution. The design should prevent the customer from being charged for services that were rolled back.

**Expected Approach:** Use an orchestration-based saga with a central Booking Saga Orchestrator. The orchestrator executes steps in sequence: (1) reserve flight, (2) reserve hotel, (3) reserve car. If step N fails, the orchestrator executes compensating actions for steps N-1 through 1 in reverse order (e.g., cancel hotel, then cancel flight). Each service exposes both a "reserve" and a "cancel" endpoint. The orchestrator persists saga state to a durable store so it can resume after a crash. All compensating actions are idempotent — cancelling an already-cancelled reservation is a no-op.

<details>
<summary>Hints</summary>

1. In an orchestration-based saga, a central coordinator (the orchestrator) tells each service what to do and when. The orchestrator knows the full workflow and drives it step by step. This is easier to reason about than choreography (where each service reacts to events and triggers the next step) for a linear, multi-step workflow.
2. Compensating actions are the "undo" for each step. They must be safe to retry (idempotent) because the orchestrator may re-send a cancel command if it is unsure whether the first attempt succeeded (e.g., after a network timeout).
3. The orchestrator must persist the saga's current state (which steps have completed, which are pending) to a durable store (database or log). If the orchestrator crashes and restarts, it reads the persisted state and resumes from where it left off — re-executing the current step or triggering compensating actions.
4. Each service's "reserve" operation should create the reservation in a "pending" state. The reservation is only confirmed after the entire saga succeeds. This prevents the customer from being charged for a reservation that might be rolled back.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement an orchestration-based saga with a Booking Saga Orchestrator that drives the three-step reservation workflow, persists state for crash recovery, and uses idempotent compensating actions for rollback.

1. **Saga steps and compensating actions:**
   | Step | Action | Service | Compensating Action |
   |------|--------|---------|-------------------|
   | 1 | Reserve flight | Flight Service | Cancel flight reservation |
   | 2 | Reserve hotel | Hotel Service | Cancel hotel reservation |
   | 3 | Reserve car rental | Car Rental Service | Cancel car rental reservation |

2. **Orchestrator workflow (happy path):**
   - Customer submits a booking request. The orchestrator creates a saga record in its database with status "STARTED" and a unique saga ID.
   - Step 1: Orchestrator calls Flight Service `POST /reservations` with saga ID. Flight Service creates a reservation in "PENDING" state and returns a reservation ID. Orchestrator updates saga state: "FLIGHT_RESERVED".
   - Step 2: Orchestrator calls Hotel Service `POST /reservations`. Hotel creates a "PENDING" reservation. Saga state: "HOTEL_RESERVED".
   - Step 3: Orchestrator calls Car Rental Service `POST /reservations`. Car rental creates a "PENDING" reservation. Saga state: "CAR_RESERVED".
   - All steps succeeded: Orchestrator sends a "confirm" command to all three services, which transition reservations from "PENDING" to "CONFIRMED". Saga state: "COMPLETED". Customer is charged.

3. **Failure scenarios with compensating actions:**
   - **Hotel fails (step 2):** Flight was reserved in step 1. Orchestrator detects the hotel failure, updates saga state to "COMPENSATING", and executes compensating actions in reverse: cancel flight reservation (step 1 compensation). Saga state: "ROLLED_BACK". Customer is not charged.
   - **Car rental fails (step 3):** Flight and hotel were reserved. Orchestrator compensates in reverse: cancel hotel (step 2 compensation), then cancel flight (step 1 compensation). Saga state: "ROLLED_BACK".
   - **Flight fails (step 1):** No prior steps to compensate. Saga state: "ROLLED_BACK". Customer is informed immediately.

4. **Idempotency of compensating actions:**
   - Each cancel endpoint is idempotent: calling `DELETE /reservations/{id}` on an already-cancelled reservation returns success (200 OK) without side effects.
   - The orchestrator includes the saga ID in every request. Services use the saga ID to deduplicate — if they receive a duplicate reserve or cancel request for the same saga, they return the existing result without creating a new reservation or cancelling twice.
   - This is critical because the orchestrator may retry a compensating action after a network timeout when it is unsure whether the first attempt succeeded.

5. **Saga state persistence and crash recovery:**
   - The orchestrator persists the saga state (current step, completed steps, reservation IDs) to a durable database after each step transition.
   - If the orchestrator crashes mid-saga (e.g., after reserving the flight but before calling the hotel), it reads the persisted state on restart and resumes from the last recorded step.
   - If the orchestrator crashes during compensation, it restarts compensation from the last recorded compensating step. Because compensating actions are idempotent, re-executing an already-completed compensation is safe.

6. **Pending reservation pattern:**
   - Reservations are created in "PENDING" state during the saga. The customer is not charged and the reservation is not visible to other customers (the seat/room/car is held but not confirmed).
   - Only after the orchestrator sends the "confirm" command do reservations transition to "CONFIRMED" and the customer is charged.
   - If the saga is rolled back, pending reservations are cancelled and the held inventory is released. A background job periodically cleans up any "PENDING" reservations older than a timeout (e.g., 5 minutes) as a safety net for sagas that were abandoned.

</details>
