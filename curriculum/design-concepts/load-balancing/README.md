# Load Balancing

**Track:** Design Concepts
**Difficulty Tier:** Intermediate
**Prerequisites:** [Caching Strategies](../caching-strategies/README.md)

## Concept Overview

Load balancing is the practice of distributing incoming network traffic across multiple servers so that no single machine bears too much demand. A load balancer sits between clients and a pool of backend servers, forwarding each request according to a chosen algorithm. The immediate benefits are higher throughput, lower response latency, and improved fault tolerance — if one server goes down, the balancer routes traffic to the remaining healthy instances. Load balancers appear at every tier of a modern architecture: in front of web servers, between application servers and databases, and even inside service meshes.

Load balancers operate at different layers of the network stack. A Layer 4 (L4) load balancer makes routing decisions based on transport-level information — source and destination IP addresses and TCP/UDP ports — without inspecting the request payload. This makes L4 balancing extremely fast and suitable for raw throughput. A Layer 7 (L7) load balancer understands application-level protocols such as HTTP, and can route based on URL paths, headers, cookies, or request content. L7 balancing enables sophisticated strategies like sending API traffic to one pool and static asset requests to another, or implementing sticky sessions based on a session cookie.

Several algorithms determine how a load balancer selects the next server. Round-robin cycles through servers in order, giving each an equal share of requests. Weighted round-robin assigns a weight to each server proportional to its capacity, so more powerful machines receive more traffic. Least-connections sends each new request to the server currently handling the fewest active connections, which adapts well to variable request durations. IP hash computes a hash of the client's IP address to consistently map the same client to the same server, providing a form of session affinity without explicit session tracking. Health checks — periodic probes sent to each backend — let the balancer detect and remove unhealthy servers from the rotation automatically, and re-add them once they recover.

---

## Problem 1 — Selecting a Round-Robin Strategy for a Stateless API Fleet

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a load-balancing scheme for a stateless REST API that runs on a fleet of identical servers. The API serves product search queries and every server can handle any request independently. Your design should explain why round-robin is a good fit, describe how requests are distributed, and address what happens when a new server is added to or removed from the fleet.

### Scenario

**Context:** A retail company runs a product search API on eight identical virtual machines behind a single load balancer. Each server has the same CPU, memory, and software version. Requests are stateless — no server stores session data — and each query takes roughly the same amount of time to process. Traffic is steady at about 4,000 requests per second during business hours.

**Requirements:** Choose an appropriate load-balancing algorithm and justify the choice. Describe the request distribution pattern. Explain how the fleet scales up (adding two servers during a sale) and scales down (removing a server for maintenance) without dropping requests. The design should also describe what clients observe during a scale event.

**Expected Approach:** Recommend plain round-robin because all servers are identical and requests are uniform. Explain the cyclic distribution (server 1, 2, 3, …, 8, 1, 2, …). Discuss how adding a server simply extends the cycle and removing one shortens it, with the load balancer updating its server list via health checks or a configuration reload.

<details>
<summary>Hints</summary>

1. Round-robin works best when servers are homogeneous and request costs are roughly equal — both conditions hold here.
2. When a new server is added, the load balancer includes it in the next cycle. Existing in-flight requests on other servers are unaffected.
3. Consider how the load balancer detects that a server has been removed — does it rely on a failed health check, or does an orchestrator update the configuration?

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use plain round-robin distribution across the identical, stateless server fleet.

1. **Algorithm choice — round-robin:**
   - All eight servers have identical hardware and software, so there is no reason to weight them differently.
   - Requests are stateless and uniform in cost, so each server can handle any request equally well.
   - Round-robin assigns requests in a fixed cycle: request 1 → server A, request 2 → server B, …, request 8 → server H, request 9 → server A, and so on.
   - At 4,000 req/s with 8 servers, each server receives approximately 500 req/s.

2. **Scaling up (adding servers):**
   - During a sale, two new servers are provisioned and registered with the load balancer (via health check discovery or config update).
   - The round-robin cycle extends to 10 servers. Each server now receives approximately 400 req/s.
   - No existing connections are disrupted — new requests simply start including the new servers in the rotation.

3. **Scaling down (removing a server):**
   - A server is drained for maintenance: the load balancer stops sending new requests to it (removed from the rotation) while existing in-flight requests complete.
   - The cycle shortens to 7 servers, each receiving approximately 571 req/s.
   - Clients observe no errors because the load balancer only routes to healthy, active servers.

4. **Health checks:**
   - The load balancer sends periodic HTTP health probes (e.g., `GET /health` every 5 seconds) to each server.
   - A server that fails 3 consecutive checks is removed from the rotation automatically.
   - Once the server passes checks again, it is re-added to the cycle.

**Why round-robin fits:** It is the simplest algorithm, introduces no computational overhead for routing decisions, and distributes load perfectly evenly when servers and requests are uniform. More complex algorithms (weighted, least-connections) add overhead without benefit in this homogeneous, stateless scenario.

</details>

---

## Problem 2 — Designing Weighted Round-Robin for a Mixed-Capacity Server Pool

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Design a load-balancing configuration for an application that runs on servers with different hardware specifications. Some servers are high-capacity machines and others are smaller, older instances kept in the pool to handle baseline traffic. Your design should assign weights proportional to each server's capacity, describe how the weighted round-robin algorithm distributes requests, and explain how to adjust weights when server performance changes.

### Scenario

**Context:** A video transcoding service runs on a mixed fleet: three large servers (32 CPU cores, 64 GB RAM each) and six small servers (8 CPU cores, 16 GB RAM each). Transcoding jobs vary in duration but are CPU-bound. Under plain round-robin, the small servers become overloaded while the large servers sit at 40% utilization. The team wants to send more traffic to the large servers without removing the small ones from the pool.

**Requirements:** Assign a weight to each server class based on relative capacity. Describe the resulting request distribution pattern. Explain how the load balancer cycles through the weighted list. Address how to detect that weights are misconfigured (e.g., a large server is underperforming due to a hardware issue) and how to adjust weights dynamically. The design should include a monitoring strategy to validate that the weights produce balanced utilization.

**Expected Approach:** Assign weights proportional to CPU capacity (e.g., large = 4, small = 1). Describe the weighted cycle where a large server receives 4 requests for every 1 request sent to a small server. Discuss monitoring CPU utilization across the fleet and adjusting weights if utilization diverges significantly.

<details>
<summary>Hints</summary>

1. If a large server has 4× the CPU of a small server, a weight of 4 vs. 1 is a reasonable starting point. The effective distribution becomes: large servers collectively receive 12 out of every 18 requests (3 × 4), small servers receive 6 out of 18 (6 × 1).
2. Weighted round-robin can be implemented by expanding the server list — e.g., [L1, L1, L1, L1, S1, L2, L2, L2, L2, S2, …] — and cycling through it, or by using a counter-based approach that tracks remaining weight per server.
3. If a large server's CPU utilization is consistently higher than expected, its effective capacity may be lower than its weight suggests — consider reducing its weight or investigating hardware issues.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Assign capacity-proportional weights and use weighted round-robin to distribute transcoding jobs fairly across the mixed fleet.

1. **Weight assignment:**
   - Large servers (32 cores): weight = 4.
   - Small servers (8 cores): weight = 1.
   - Total weight = (3 × 4) + (6 × 1) = 18.
   - Each large server receives 4/18 ≈ 22% of traffic. Each small server receives 1/18 ≈ 5.6% of traffic.
   - Collectively, large servers handle 12/18 ≈ 67% of jobs, small servers handle 6/18 ≈ 33%.

2. **Distribution cycle:**
   - The load balancer maintains a weighted schedule. In a smooth weighted round-robin implementation, requests are spread evenly across the cycle rather than sending 4 consecutive requests to one large server.
   - Example cycle for one large (L, weight 4) and two small (S1, S2, weight 1 each): L, L, S1, L, S2, L — then repeat. This avoids bursting all weighted requests to one server at once.

3. **Dynamic weight adjustment:**
   - Monitor CPU utilization per server. Target: all servers at roughly 70% utilization under normal load.
   - If a large server consistently runs at 90% while small servers are at 50%, its effective capacity is lower than assumed — reduce its weight from 4 to 3 and re-evaluate.
   - Weight changes are applied via a configuration reload without restarting the load balancer.

4. **Detecting misconfiguration:**
   - Set up alerts for utilization divergence: if any server's CPU utilization deviates more than 20 percentage points from the fleet average, trigger a review.
   - Track request latency per server. A server with disproportionately high latency relative to its weight may have a hardware degradation issue.

5. **Health check integration:**
   - A server that fails health checks is removed from the weighted rotation entirely (its weight effectively becomes 0).
   - When it recovers, it re-enters with its configured weight.

**Why weighted round-robin fits:** The fleet has predictable, static capacity differences. Weighted round-robin is simple to configure, easy to reason about, and distributes load proportionally without the overhead of tracking active connections. It is the natural extension of round-robin for heterogeneous server pools.

</details>

---

## Problem 3 — Implementing Least-Connections Balancing for Variable-Duration Requests

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a load-balancing strategy for a service where request processing times vary dramatically — from 50 milliseconds for simple lookups to 30 seconds for complex report generation. Under round-robin, servers handling long-running requests accumulate a backlog while servers that just finished fast requests sit idle. Your design should explain how the least-connections algorithm solves this problem, describe the routing decision process, and address edge cases such as server startup and connection count ties.

### Scenario

**Context:** A business analytics platform exposes an API that serves two types of requests: dashboard widget queries (average 100 ms) and full report exports (average 15 seconds). The ratio is roughly 80% widget queries to 20% report exports, but the distribution is unpredictable — a single user might trigger five report exports in a row. The platform runs on six identical servers behind a load balancer. Under round-robin, a server that receives several consecutive report exports becomes saturated (50+ active connections) while neighboring servers have only 2–3 active connections.

**Requirements:** Replace round-robin with least-connections routing. Describe how the load balancer tracks active connection counts per server and selects the server with the fewest active connections for each new request. Explain how this naturally adapts to variable request durations. Address what happens when a new server joins the pool with zero connections (cold start thundering herd), and how ties are broken when multiple servers have the same connection count. The design should include a mechanism to prevent a newly added server from being overwhelmed.

**Expected Approach:** Explain that least-connections routes each request to the server with the lowest current active connection count. Long-running requests keep a server's count elevated, so new requests naturally flow to less-loaded servers. For cold start, use a slow-start mechanism that gradually increases the new server's allowed connections. Break ties with round-robin among tied servers.

<details>
<summary>Hints</summary>

1. The load balancer maintains a real-time counter for each server: incremented when a request is forwarded, decremented when the response completes (or the connection closes). The server with the lowest counter gets the next request.
2. When a new server joins with 0 active connections, it will receive all incoming requests until its count matches the others — this can overwhelm it. A slow-start ramp (e.g., artificially inflating its count or capping new connections for the first 30 seconds) prevents this.
3. Tie-breaking matters when servers are lightly loaded and many have the same count. Round-robin among tied servers ensures even distribution during low-traffic periods.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use least-connections routing to dynamically adapt to variable request durations, with slow-start for new servers and round-robin tie-breaking.

1. **Least-connections routing logic:**
   - The load balancer maintains an active connection counter for each of the six servers.
   - When a new request arrives, the balancer selects the server with the lowest active connection count.
   - When a request completes (response sent or connection closed), the counter for that server is decremented.
   - A server processing three 15-second report exports has an active count of 3, while a server that just finished a batch of fast widget queries has a count of 0. The next request goes to the server at 0.

2. **Adaptation to variable durations:**
   - Report exports keep a server's count elevated for 15+ seconds, naturally diverting new requests to other servers.
   - Widget queries complete in ~100 ms, quickly freeing up connection slots.
   - The algorithm self-balances without any knowledge of request types — it only needs the connection count.

3. **Cold start / thundering herd prevention:**
   - When a new server is added, it starts with 0 connections and would attract all incoming traffic.
   - Implement a slow-start ramp: for the first 30 seconds after a server joins, the load balancer limits the rate of new connections to that server (e.g., max 5 new connections per second, gradually increasing to no limit).
   - Alternatively, the load balancer can set an artificial initial connection count equal to the fleet average, so the new server is treated as equally loaded until real counts take over.

4. **Tie-breaking:**
   - When multiple servers share the lowest connection count, the load balancer uses round-robin among the tied servers.
   - This prevents the same server from always winning ties (e.g., the first server in the list) and ensures even distribution during low-traffic periods.

5. **Monitoring:**
   - Track active connection counts per server on a dashboard. A healthy distribution shows counts within a narrow range (e.g., all servers between 5 and 10 active connections).
   - Alert if any server's count exceeds 2× the fleet average, which may indicate a stuck connection or a server that is not completing requests.

**Why least-connections fits:** Round-robin assumes all requests take the same time, which fails badly when request durations vary by 100×. Least-connections naturally adapts to workload imbalance by routing new requests away from busy servers, making it the standard choice for services with unpredictable or variable request processing times.

</details>

---

## Problem 4 — Designing L4 vs. L7 Load Balancing for a Multi-Service Platform

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design the load-balancing architecture for a platform that serves both a high-throughput TCP streaming service and an HTTP-based REST API from the same infrastructure. The two workloads have fundamentally different routing needs: the streaming service requires raw TCP forwarding with minimal latency, while the REST API needs content-based routing (path-based routing, header inspection, and TLS termination). Your design should specify where L4 and L7 load balancers are placed, what each layer handles, and how the two tiers interact.

### Scenario

**Context:** A financial data platform serves two workloads. The first is a real-time market data stream delivered over persistent TCP connections — clients connect and receive a continuous feed of price updates. This service handles 50,000 concurrent connections and is extremely latency-sensitive (every microsecond matters). The second is a REST API for historical data queries, portfolio management, and account operations. The API handles 10,000 requests per second, requires TLS termination, path-based routing (e.g., `/api/v2/quotes` goes to the quotes service, `/api/v2/portfolio` goes to the portfolio service), and authentication header inspection. Both workloads share the same public IP range and must coexist behind a unified entry point.

**Requirements:** Design a two-tier load-balancing architecture. Specify which workload uses L4 balancing and which uses L7. Explain why each layer is appropriate for its workload. Describe how TLS is handled at each tier. Address how health checks differ between the TCP streaming servers and the HTTP API servers. The design should include a diagram or clear description of the request flow from client to backend for both workloads.

**Expected Approach:** Place an L4 load balancer at the edge to handle initial traffic splitting — TCP streaming traffic is forwarded directly to streaming servers based on port or IP, while HTTP traffic is forwarded to an L7 load balancer. The L7 balancer terminates TLS, inspects HTTP paths and headers, and routes to the appropriate API backend pool. Discuss why L4 is chosen for streaming (low overhead, no payload inspection needed) and L7 for the API (content-based routing, TLS termination, header inspection).

<details>
<summary>Hints</summary>

1. L4 load balancers operate on TCP/UDP packets without decrypting or inspecting payloads. This makes them ideal for the streaming service where latency is critical and there is no need to understand the application protocol.
2. L7 load balancers decrypt TLS, parse HTTP requests, and can route based on URL path, Host header, cookies, or other application-level data. This is necessary for the REST API's path-based routing and authentication header inspection.
3. A common pattern is a two-tier architecture: L4 at the edge for raw traffic distribution, L7 behind it for application-aware routing. The L4 tier can also split traffic by port (e.g., port 443 for HTTPS → L7, port 9000 for TCP streaming → streaming servers).
4. Health checks for TCP services typically use a TCP connect probe (can the balancer open a socket?), while HTTP services use an HTTP GET to a health endpoint (does the server return 200 OK?).

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Deploy a two-tier load-balancing architecture with L4 at the edge for traffic splitting and L7 behind it for HTTP-aware routing.

1. **Architecture overview:**
   - **Tier 1 — L4 load balancer (edge):** Receives all inbound traffic on the platform's public IP. Routes based on destination port:
     - Port 9000 (TCP streaming) → forwarded directly to the streaming server pool.
     - Port 443 (HTTPS) → forwarded to the L7 load balancer pool.
   - **Tier 2 — L7 load balancer (HTTP tier):** Receives HTTPS traffic from the L4 tier. Terminates TLS, inspects HTTP request path and headers, and routes to the appropriate API backend pool.

2. **L4 tier — streaming traffic:**
   - The L4 balancer forwards raw TCP packets to streaming servers without decrypting or inspecting them.
   - Routing algorithm: least-connections (streaming connections are long-lived, so least-connections prevents any single server from accumulating too many persistent connections).
   - TLS for the streaming service is handled end-to-end: the client and streaming server negotiate TLS directly. The L4 balancer passes encrypted packets through without termination.
   - Health check: TCP connect probe to port 9000 every 3 seconds. If a server fails to accept a TCP connection on 3 consecutive probes, it is removed from the pool.

3. **L7 tier — REST API traffic:**
   - The L7 balancer terminates TLS (holds the platform's SSL certificate) and decrypts incoming HTTPS requests.
   - Path-based routing rules:
     - `/api/v2/quotes/*` → quotes service backend pool.
     - `/api/v2/portfolio/*` → portfolio service backend pool.
     - `/api/v2/account/*` → account service backend pool.
   - The balancer inspects the `Authorization` header to reject unauthenticated requests before they reach backends (offloading auth validation).
   - Routing algorithm: round-robin within each backend pool (API requests are short-lived and roughly uniform in cost).
   - Health check: HTTP GET to `/health` on each backend every 5 seconds. A server returning non-200 on 2 consecutive checks is removed.

4. **TLS handling summary:**
   - Streaming: TLS passthrough at L4 (end-to-end encryption between client and server).
   - REST API: TLS termination at L7 (the L7 balancer decrypts, inspects, and re-encrypts or forwards plain HTTP to backends over a trusted internal network).

5. **Failure scenarios:**
   - If an L7 balancer instance fails, the L4 tier detects it via health checks and stops forwarding HTTPS traffic to it. Other L7 instances absorb the load.
   - If a streaming server fails, the L4 tier removes it. Clients on that server experience a disconnection and must reconnect (the reconnection is routed to a healthy server).

**Why two tiers:** A single L7 balancer cannot efficiently handle raw TCP streaming (it would need to decrypt and re-encrypt every packet, adding unacceptable latency). A single L4 balancer cannot perform path-based HTTP routing. The two-tier design lets each layer do what it does best: L4 for fast, protocol-agnostic forwarding; L7 for intelligent, content-aware routing.

</details>

---

## Problem 5 — Designing Session Persistence with Sticky Sessions and Failover

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a load-balancing strategy that maintains session persistence (sticky sessions) for a stateful web application while still providing failover when a server goes down. The application stores user session data in server-local memory, so a user must be routed to the same server for the duration of their session. However, if that server fails, the user's session must not be permanently lost. Your design should describe how sticky sessions are implemented, how failover is handled, and the trade-offs between session affinity and load distribution.

### Scenario

**Context:** A legacy e-commerce application stores shopping cart data and authentication state in server-local memory (not in a shared store like Redis). The application runs on ten servers behind an L7 load balancer. Users interact with the site for 15–30 minutes per session, adding items to their cart, applying coupons, and proceeding to checkout. If a user is routed to a different server mid-session, their cart is empty and they must log in again. The business cannot tolerate this experience. At the same time, the team has observed that sticky sessions cause uneven load distribution — popular sessions cluster on a few servers while others are underutilized. Occasionally, a server crashes and all users pinned to it lose their sessions entirely.

**Requirements:** Design a sticky session mechanism that pins users to a specific server for the duration of their session. Choose between cookie-based and IP-hash-based affinity and justify the choice. Address the load imbalance problem caused by sticky sessions. Design a failover strategy so that when a server crashes, affected users can recover their sessions with minimal disruption. The design must balance session affinity, load distribution, and fault tolerance. Explain the trade-offs and any compromises made.

**Expected Approach:** Use cookie-based sticky sessions (the load balancer inserts a cookie identifying the target server) because it is more reliable than IP hash for users behind NAT or proxies. Address load imbalance by combining sticky sessions with least-connections for initial assignment. For failover, replicate session data to a secondary server or a shared store so that when the primary fails, the user can be re-routed to the backup with their session intact. Discuss the trade-off between full session replication (high consistency, high overhead) and accepting partial session loss on failure.

<details>
<summary>Hints</summary>

1. Cookie-based sticky sessions work by having the load balancer set a cookie (e.g., `SERVERID=server-3`) on the first response. Subsequent requests from the same browser include this cookie, and the load balancer routes to the specified server. This is more reliable than IP hash because it works correctly for users behind shared NATs or proxies.
2. Load imbalance from sticky sessions can be mitigated by using least-connections for the initial server assignment (the first request from a new session goes to the least-loaded server), and then pinning subsequent requests to that server via the cookie.
3. For failover, consider session replication: each server asynchronously replicates its session data to one designated backup server. If the primary fails, the load balancer re-routes the user to the backup, which has a recent copy of the session. This is cheaper than replicating to all servers.
4. The ideal long-term solution is to externalize session storage (move sessions to Redis), but the problem asks you to work within the constraint of server-local storage.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement cookie-based sticky sessions with least-connections initial assignment, paired with pairwise session replication for failover.

1. **Cookie-based sticky sessions:**
   - When a user's first request arrives, the load balancer selects a server using the least-connections algorithm (ensuring the initial assignment is load-aware).
   - The load balancer inserts a cookie in the response: `Set-Cookie: SERVERID=server-7; Path=/; HttpOnly`.
   - All subsequent requests from that browser include the `SERVERID` cookie. The load balancer reads the cookie and routes directly to `server-7`, bypassing the normal selection algorithm.
   - When the session ends (user logs out or the cookie expires), the affinity is released and the next session starts with a fresh least-connections assignment.

2. **Why cookie-based over IP hash:**
   - IP hash maps a client IP to a server deterministically. However, many users share the same public IP (corporate NATs, mobile carrier NATs), so IP hash can cluster thousands of users onto one server.
   - Cookie-based affinity is per-browser, not per-IP, giving much finer granularity and more even distribution.
   - Cookies also survive IP changes (e.g., a mobile user switching from Wi-Fi to cellular), maintaining session continuity.

3. **Mitigating load imbalance:**
   - The initial assignment uses least-connections, so new sessions are directed to the least-loaded server. This prevents the "snowball" effect where one server accumulates sessions faster than others.
   - Set a maximum session duration (e.g., 2 hours). After expiration, the cookie is cleared and the next request is re-balanced, preventing indefinite pinning.
   - Monitor per-server active session counts. If imbalance exceeds a threshold (e.g., one server has 2× the average), new sessions are temporarily excluded from that server.

4. **Failover via pairwise session replication:**
   - Each server is paired with a designated backup (e.g., server-1 ↔ server-2, server-3 ↔ server-4, etc.).
   - Each server asynchronously replicates session data to its partner every 5 seconds (or on every session mutation, depending on acceptable overhead).
   - If server-7 crashes, the load balancer detects the failure via health checks and re-routes all requests with `SERVERID=server-7` to server-7's backup partner (server-8).
   - Server-8 has a recent copy of the session data. The user may lose at most 5 seconds of session changes (the replication interval), but their cart and login state are preserved.
   - The load balancer updates the cookie to `SERVERID=server-8` so subsequent requests go directly to the new server.

5. **Trade-offs:**
   - **Session affinity vs. load distribution:** Sticky sessions inherently limit the balancer's ability to redistribute load. The least-connections initial assignment and session expiration mitigate this but do not eliminate it.
   - **Replication overhead vs. failover quality:** Pairwise replication doubles the memory usage on each backup server for session data and adds network overhead. Full replication to all servers would provide better failover flexibility but at 10× the cost.
   - **Data loss window:** The 5-second replication interval means up to 5 seconds of session changes can be lost on failover. Reducing the interval improves durability but increases replication traffic.
   - **Long-term recommendation:** Externalize sessions to a shared store (e.g., Redis) to eliminate the need for sticky sessions and server-side replication entirely. This design addresses the constraint of server-local storage as a pragmatic interim solution.

**Why cookie-based sticky sessions with pairwise replication fits:** It provides reliable session affinity that works across NATs and IP changes, load-aware initial assignment to prevent imbalance, and a practical failover mechanism that preserves most session data without the cost of full cluster-wide replication.

</details>
