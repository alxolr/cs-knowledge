# Rate Limiting

**Track:** Design Concepts
**Difficulty Tier:** Intermediate
**Prerequisites:** [API Design](../api-design/README.md)

## Concept Overview

Rate limiting is the practice of controlling the number of requests a client can make to a service within a given time window. It protects backend systems from being overwhelmed by excessive traffic — whether from a misbehaving client, a bot scraping data, or a sudden traffic spike — and ensures fair resource allocation across all consumers. Rate limiters sit in front of APIs, gateways, and services, rejecting or throttling requests that exceed a configured threshold, typically by returning an HTTP 429 (Too Many Requests) response.

Several algorithms implement rate limiting, each with different trade-offs in accuracy, memory usage, and burst tolerance. The fixed window counter divides time into discrete intervals and counts requests per interval — simple but susceptible to burst traffic at window boundaries. The sliding window log tracks the exact timestamp of every request and counts how many fall within a rolling window — highly accurate but memory-intensive. The token bucket algorithm maintains a bucket that refills at a steady rate; each request consumes a token, and requests are rejected when the bucket is empty — this naturally allows short bursts while enforcing a long-term average rate. The sliding window counter combines fixed windows with interpolation to approximate a sliding window at lower memory cost.

In distributed systems, rate limiting becomes significantly more complex. When multiple API gateway instances share the load, they must coordinate their counters to enforce a global rate limit rather than a per-instance limit. This typically involves a centralized data store like Redis for atomic counter operations, or a distributed algorithm that synchronizes counts across nodes. Designing an effective rate limiter requires balancing accuracy (how closely the enforced rate matches the configured limit), performance (the latency overhead of checking and updating counters), and fairness (ensuring no single client can starve others of capacity).

---

## Problem 1 — Token Bucket Rate Limiter for a Public REST API

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a token bucket rate limiter for a public REST API that allows clients to make up to 100 requests per minute, with the ability to handle short bursts of up to 20 requests in rapid succession. The limiter should operate per API key and explain how tokens are consumed and refilled over time.

### Scenario

**Context:** A weather data company exposes a public REST API that serves current conditions and forecasts. Each client authenticates with an API key. The company wants to allow 100 requests per minute per key, but recognizes that some clients send small bursts of requests (e.g., loading a dashboard that fetches 15 endpoints simultaneously). A strict per-second limit would reject these legitimate bursts, so the team needs an algorithm that permits short bursts while enforcing the overall rate.

**Requirements:** Design a per-client rate limiter using the token bucket algorithm. Specify the bucket capacity (maximum burst size), the refill rate (tokens added per second), and how the limiter decides whether to allow or reject a request. Explain what happens when a client sends 20 requests instantly, then waits 10 seconds, then sends another burst. The design should describe the data stored per client and how the limiter responds when the bucket is empty.

**Expected Approach:** Configure a token bucket with capacity 20 (maximum burst) and a refill rate of 100/60 ≈ 1.67 tokens per second. Each request consumes one token. When the bucket is empty, requests are rejected with HTTP 429. After a period of inactivity, the bucket refills up to its maximum capacity, allowing another burst.

<details>
<summary>Hints</summary>

1. The bucket capacity controls the maximum burst size — set it to 20 to allow 20 rapid requests before throttling kicks in.
2. The refill rate controls the sustained throughput — 100 requests per 60 seconds means approximately 1.67 tokens are added per second.
3. You do not need a background thread to refill tokens. Instead, calculate the number of tokens to add based on the elapsed time since the last request (lazy refill).

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement a per-client token bucket with lazy refill to allow controlled bursts while enforcing a sustained rate limit.

1. **Bucket configuration per API key:**
   - Bucket capacity: 20 tokens (maximum burst size).
   - Refill rate: 1.67 tokens per second (100 tokens per 60 seconds).
   - Stored per client: `tokens` (current token count, float), `last_refill_timestamp`.

2. **Request processing logic:**
   ```
   function allowRequest(apiKey):
       bucket = getBucket(apiKey)
       now = currentTime()
       elapsed = now - bucket.last_refill_timestamp
       bucket.tokens = min(bucket.capacity, bucket.tokens + elapsed * refillRate)
       bucket.last_refill_timestamp = now
       if bucket.tokens >= 1:
           bucket.tokens -= 1
           return ALLOW
       else:
           return REJECT (HTTP 429)
   ```

3. **Burst scenario walkthrough:**
   - Client sends 20 requests instantly: bucket starts at 20 tokens, each request consumes 1 token, all 20 are allowed, bucket reaches 0.
   - Client waits 10 seconds: bucket refills by 10 × 1.67 ≈ 16.7 tokens.
   - Client sends another burst of 16 requests: all 16 are allowed (16.7 tokens available), bucket drops to ~0.7 tokens. The 17th request in this burst would be rejected.

4. **Rejection response:**
   - Return HTTP 429 with a `Retry-After` header indicating how many seconds until at least one token is available: `Retry-After: ceil(1 / refillRate)` ≈ 1 second.
   - Include `X-RateLimit-Limit: 100`, `X-RateLimit-Remaining: 0`, and `X-RateLimit-Reset: <epoch timestamp of next minute boundary>` headers for client visibility.

5. **Storage:**
   - Each client requires only two values: current token count and last refill timestamp. For 100,000 active API keys, this is approximately 1.6 MB of memory (16 bytes per key) — trivially small.

**Why token bucket fits:** It naturally allows short bursts (up to the bucket capacity) while enforcing a long-term average rate (the refill rate). This matches the requirement of permitting dashboard-style burst loads without exceeding the overall 100 requests/minute limit.

</details>

---

## Problem 2 — Sliding Window Log for Precise Rate Enforcement

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Design a rate limiter using the sliding window log algorithm for an API that requires precise rate enforcement with no boundary-edge loopholes. The system must guarantee that no client exceeds exactly 1,000 requests in any rolling 60-second window, regardless of when within the window the requests arrive. Explain the data structure, the allow/reject decision process, and the memory trade-offs compared to simpler algorithms.

### Scenario

**Context:** A financial trading platform exposes an order submission API. Regulatory compliance requires that no single trading firm submits more than 1,000 orders in any 60-second rolling window. The fixed window counter approach was rejected because it allows up to 2,000 requests at a window boundary (1,000 at the end of one window and 1,000 at the start of the next). The compliance team demands exact enforcement: at any point in time, looking back 60 seconds, the count must not exceed 1,000.

**Requirements:** Design a sliding window log rate limiter that stores the timestamp of each allowed request. Describe how the limiter checks whether a new request would exceed the limit by counting timestamps within the trailing 60-second window. Explain how old timestamps are purged. Analyze the memory cost per client for a high-volume scenario (1,000 requests/minute) and discuss when this algorithm is appropriate versus when a more memory-efficient alternative should be used.

**Expected Approach:** Store a sorted list (or sorted set) of request timestamps per client. On each new request, remove all timestamps older than 60 seconds, count the remaining entries, and allow the request only if the count is below 1,000. Discuss that memory usage is proportional to the rate limit (up to 1,000 timestamps per client) and that this is acceptable for moderate limits but becomes expensive at very high rates.

<details>
<summary>Hints</summary>

1. Use a sorted set (e.g., Redis ZSET) per client, where each entry's score is the request timestamp. This allows efficient range queries and removal of expired entries.
2. The check-and-insert must be atomic to prevent race conditions where two concurrent requests both see 999 entries and both get allowed, pushing the count to 1,001.
3. Memory usage per client is proportional to the rate limit, not the window size. At 1,000 requests/minute, each client stores up to 1,000 timestamps (8 bytes each = 8 KB per client).

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Maintain a sorted log of request timestamps per client and count entries within the rolling window to enforce exact limits.

1. **Data structure per client:**
   - A sorted set (e.g., Redis ZSET) keyed by client ID.
   - Each entry: timestamp of an allowed request (stored as both the value and the score for efficient range operations).

2. **Request processing logic:**
   ```
   function allowRequest(clientId):
       now = currentTimestamp()
       windowStart = now - 60 seconds

       // Step 1: Remove all entries older than the window
       ZREMRANGEBYSCORE(clientId, 0, windowStart)

       // Step 2: Count entries in the current window
       count = ZCARD(clientId)

       // Step 3: Allow or reject
       if count < 1000:
           ZADD(clientId, now, now)    // add this request's timestamp
           return ALLOW
       else:
           return REJECT (HTTP 429)
   ```

3. **Atomicity:**
   - Wrap steps 1–3 in a Redis transaction (MULTI/EXEC) or a Lua script to ensure the check-and-insert is atomic.
   - Without atomicity, two concurrent requests could both read count = 999 and both be allowed, violating the limit.

4. **Memory analysis:**
   - Each timestamp is 8 bytes. At the maximum rate of 1,000 requests/minute, each client stores up to 1,000 entries = 8 KB.
   - For 10,000 active clients: 10,000 × 8 KB = 80 MB. This is manageable for a Redis instance.
   - For 100,000 clients at 10,000 requests/minute each: 100,000 × 80 KB = 8 GB. At this scale, the sliding window log becomes impractical and a more memory-efficient algorithm (sliding window counter or token bucket) should be used.

5. **Purging old entries:**
   - Old timestamps are removed on every request (step 1). For clients that stop sending requests, set a TTL on the Redis key equal to the window size (60 seconds) so the entire key is automatically deleted after inactivity.

6. **Why sliding window log is appropriate here:**
   - Regulatory compliance demands exact enforcement — no boundary-edge loopholes.
   - The rate limit (1,000/minute) is moderate, so memory usage per client is acceptable.
   - The trading platform has a bounded number of clients (trading firms), so total memory is predictable.

**Trade-off vs. fixed window counter:** The fixed window counter uses only a single integer per client (8 bytes) but allows up to 2× the limit at window boundaries. The sliding window log uses up to 8 KB per client but guarantees exact enforcement. For regulatory compliance, the accuracy justifies the memory cost.

</details>

---

## Problem 3 — Fixed Window Counter with Race Condition Mitigation

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a fixed window counter rate limiter for a high-throughput API gateway that handles 500,000 requests per second. The limiter must enforce a limit of 10,000 requests per minute per client. Because of the extreme throughput, the algorithm must minimize per-request overhead. Your design should address the boundary burst problem inherent in fixed windows and describe how to handle race conditions when multiple gateway threads increment the counter concurrently.

### Scenario

**Context:** A social media platform's API gateway processes half a million requests per second across all clients. Each client (identified by OAuth token) is limited to 10,000 requests per minute. The engineering team chose the fixed window counter for its simplicity and minimal memory footprint (one integer per client per window). However, during load testing, two issues emerged: (1) at the boundary between two one-minute windows, a client could send 10,000 requests in the last second of window N and 10,000 in the first second of window N+1, effectively achieving 20,000 requests in a two-second span; (2) under high concurrency, multiple threads read the counter as 9,999, each allow the request, and each increment to 10,000 — resulting in several requests over the limit.

**Requirements:** Design the fixed window counter with atomic increment operations to eliminate the race condition. Propose a mitigation strategy for the boundary burst problem that does not require switching to a fully sliding window algorithm (which is too memory-intensive at this scale). Specify the data store and operations used for counter management. The design should quantify the worst-case boundary burst and explain why the mitigation is acceptable for this use case.

**Expected Approach:** Use Redis INCR (atomic increment) to eliminate race conditions. For the boundary burst, implement a sliding window counter approximation: maintain counters for the current and previous windows, and weight the previous window's count by the fraction of time remaining in the current window. This provides near-sliding-window accuracy with only two integers per client.

<details>
<summary>Hints</summary>

1. Redis INCR is atomic — it reads, increments, and writes in a single operation, eliminating the read-then-write race condition. Use `INCR key` and check the returned value against the limit.
2. To mitigate boundary bursts without a full sliding window log, keep counters for both the current and previous windows. Approximate the sliding window count as: `previous_count × (1 - elapsed_fraction) + current_count`, where `elapsed_fraction` is how far into the current window you are.
3. Set a TTL on each window's counter key equal to 2× the window size (120 seconds for a 60-second window) so the previous window's counter is available for the approximation but eventually cleaned up.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use atomic Redis operations for thread-safe counting and a sliding window counter approximation to mitigate boundary bursts.

1. **Basic fixed window counter:**
   - Key format: `ratelimit:{clientId}:{windowId}` where `windowId = floor(currentTimestamp / 60)`.
   - On each request:
     ```
     key = "ratelimit:" + clientId + ":" + floor(now / 60)
     count = INCR(key)
     if count == 1:
         EXPIRE(key, 120)    // TTL = 2× window for sliding approximation
     if count > 10000:
         return REJECT (HTTP 429)
     else:
         return ALLOW
     ```
   - INCR is atomic, so concurrent threads cannot produce a race condition. If two threads call INCR simultaneously, one gets 10,000 and the other gets 10,001 — the second is correctly rejected.

2. **Boundary burst mitigation — sliding window counter approximation:**
   - Maintain counters for both the current window and the previous window.
   - On each request, compute the weighted count:
     ```
     currentWindowId = floor(now / 60)
     elapsedInCurrentWindow = now - (currentWindowId × 60)
     elapsedFraction = elapsedInCurrentWindow / 60

     previousKey = "ratelimit:" + clientId + ":" + (currentWindowId - 1)
     currentKey = "ratelimit:" + clientId + ":" + currentWindowId

     previousCount = GET(previousKey) or 0
     currentCount = GET(currentKey) or 0

     weightedCount = previousCount × (1 - elapsedFraction) + currentCount
     ```
   - If `weightedCount >= 10,000`, reject the request. Otherwise, INCR the current window counter and allow.

3. **Boundary burst analysis:**
   - Without the approximation: a client could send 10,000 requests at second 59 of window N and 10,000 at second 0 of window N+1 = 20,000 in 2 seconds.
   - With the approximation: at second 0 of window N+1, `elapsedFraction = 0`, so `weightedCount = previousCount × 1.0 + currentCount`. If the previous window had 10,000 requests, the weighted count is already 10,000 and new requests are rejected. The worst-case burst is reduced to approximately 10,000 × (1 + elapsedFraction) at any point — far closer to the intended limit.

4. **Performance at scale:**
   - Each request requires 1–2 Redis operations (INCR + optional GET for the previous window). At 500,000 req/s, this is manageable with a Redis cluster.
   - Memory per client: two integers (current and previous window counters) + key overhead ≈ 200 bytes. For 1 million clients: ~200 MB.

5. **TTL and cleanup:**
   - Each counter key has a TTL of 120 seconds (2× the window size). After two windows, the key is automatically deleted.
   - No background cleanup process is needed — Redis handles expiration.

**Why this approach fits:** The fixed window counter with sliding approximation provides near-exact rate enforcement with minimal memory (two integers per client) and minimal latency (one atomic INCR per request). It is the right trade-off for a high-throughput gateway where the full sliding window log would consume too much memory and the pure fixed window allows unacceptable boundary bursts.

</details>

---

## Problem 4 — Distributed Rate Limiting Across Multiple Gateway Instances

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design a distributed rate limiting system for an API platform where requests are handled by 20 geographically distributed gateway instances. The system must enforce a global rate limit of 5,000 requests per minute per client, regardless of which gateway instance handles the request. Your design should address the trade-off between strict global accuracy and per-request latency, and propose a synchronization strategy that works across data centers with 50–100 ms network latency between them.

### Scenario

**Context:** A global SaaS platform runs API gateways in four regions (US-East, US-West, EU, Asia-Pacific), with five gateway instances per region. Each client's traffic may hit any gateway depending on DNS-based geographic routing, but some clients have users worldwide and their requests are spread across all regions. The platform enforces a global limit of 5,000 requests per minute per client. A naive approach — each gateway enforcing 5,000/20 = 250 requests per minute locally — fails because traffic distribution is uneven: a client with 90% US traffic would be limited to 1,250 req/min (5 US gateways × 250) instead of the full 5,000.

**Requirements:** Design a synchronization mechanism that allows all 20 gateways to collectively enforce the global limit. Consider a centralized counter (e.g., a global Redis instance) and analyze its latency impact. Propose a hybrid approach that combines local counters with periodic synchronization to reduce per-request latency while maintaining acceptable accuracy. Quantify the accuracy trade-off (how far over the limit a client might go) and explain why it is acceptable. Address what happens when the central store is temporarily unreachable.

**Expected Approach:** Use a hybrid approach: each gateway maintains a local counter and periodically synchronizes with a central Redis cluster. Local counters allow low-latency decisions, while periodic sync (every 1–5 seconds) keeps the global count approximately accurate. Analyze the worst case: if all 20 gateways allow requests just before syncing, the overshoot is bounded by `20 × (sync_interval × local_rate)`. Include a fallback strategy for central store unavailability.

<details>
<summary>Hints</summary>

1. A centralized Redis counter (INCR per request) gives perfect accuracy but adds 50–100 ms of cross-region latency to every request — unacceptable for an API gateway that targets sub-10 ms overhead.
2. A hybrid approach: each gateway maintains a local counter and periodically reports its count to a central store, fetching the global total in return. Between syncs, the gateway makes decisions based on its local count plus the last-known global count.
3. The sync interval controls the accuracy/latency trade-off. A 2-second sync interval means each gateway could allow up to `local_rate × 2 seconds` of requests between syncs without knowing the global total. With 20 gateways, the worst-case overshoot is `20 × local_rate × 2`.
4. When the central store is unreachable, gateways can fall back to a conservative local limit (e.g., 5,000 / 20 = 250 per gateway) until connectivity is restored.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement a hybrid local-plus-global rate limiting system with periodic synchronization and a degraded-mode fallback.

1. **Architecture overview:**
   - Each of the 20 gateway instances maintains a local request counter per client.
   - A central Redis cluster (deployed in a primary region with read replicas in other regions) stores the global counter per client.
   - Gateways synchronize with the central store every `S` seconds (sync interval, configurable — default 2 seconds).

2. **Per-request decision (local fast path):**
   ```
   function allowRequest(clientId):
       local = getLocalCounter(clientId)
       globalSnapshot = getLastKnownGlobal(clientId)  // from last sync
       estimatedGlobal = globalSnapshot + local.countSinceLastSync

       if estimatedGlobal >= 5000:
           return REJECT (HTTP 429)
       else:
           local.countSinceLastSync += 1
           return ALLOW
   ```
   - This check uses only local memory — no network call — so it adds sub-microsecond overhead.

3. **Periodic synchronization (every S seconds):**
   ```
   function syncWithCentral(clientId):
       localDelta = local.countSinceLastSync
       globalTotal = INCRBY(centralKey(clientId), localDelta)  // atomic add to central
       local.countSinceLastSync = 0
       local.lastKnownGlobal = globalTotal
   ```
   - The gateway reports its local count to the central store and receives the updated global total.
   - After sync, the gateway has an accurate global snapshot and resets its local delta.

4. **Accuracy analysis (worst-case overshoot):**
   - Between syncs, each gateway can allow up to `S × (local request rate)` requests without knowing the global total.
   - Worst case: all 20 gateways receive requests for the same client simultaneously, each allows requests for `S` seconds before syncing.
   - If the client sends evenly across all gateways at the maximum rate: overshoot ≈ `20 × (5000/60) × S` = `20 × 83.3 × 2` ≈ 3,333 extra requests with a 2-second sync interval.
   - To reduce overshoot, decrease `S` (e.g., 500 ms sync → overshoot ≈ 833) or use a token-bucket approach where each gateway requests a batch of tokens from the central store.

5. **Token batch optimization:**
   - Instead of syncing counts, each gateway requests a batch of tokens from the central store (e.g., 100 tokens at a time).
   - The gateway allows requests until its local batch is exhausted, then requests a new batch.
   - The central store tracks how many tokens have been issued globally. When 5,000 tokens are issued, no more batches are granted.
   - Overshoot is bounded by `20 × batch_size` = 2,000 in the worst case (all gateways use their last batch simultaneously). Smaller batches reduce overshoot but increase sync frequency.

6. **Fallback when central store is unreachable:**
   - If a gateway cannot reach the central Redis for more than 3 consecutive sync attempts, it switches to degraded mode.
   - Degraded mode: enforce a conservative local limit of `5,000 / 20 = 250` requests per minute per client on this gateway.
   - This under-limits some clients (those whose traffic is concentrated on this gateway) but prevents any client from exceeding the global limit.
   - When connectivity is restored, the gateway syncs its accumulated local count and resumes normal hybrid operation.

7. **Monitoring:**
   - Track the delta between estimated global counts and actual global counts after each sync. Large deltas indicate the sync interval is too long.
   - Alert if any client's actual global count exceeds 110% of the limit (a 10% tolerance for sync lag).

**Why hybrid fits:** A purely centralized approach adds unacceptable latency (50–100 ms per request). A purely local approach cannot enforce a global limit. The hybrid approach provides sub-millisecond per-request decisions with bounded overshoot, and degrades gracefully when the central store is unavailable.

</details>

---

## Problem 5 — API Rate Limiting with Tiered Subscription Plans

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a rate limiting system for a commercial API platform that offers three subscription tiers — Free, Professional, and Enterprise — each with different rate limits, burst allowances, and throttling behaviors. The system must support dynamic tier upgrades (a client upgrades from Free to Professional mid-session), per-endpoint rate limits (some endpoints have stricter limits than others), and graceful degradation (throttling rather than hard rejection for paying customers). Your design should describe the configuration model, the request evaluation pipeline, and how tier changes take effect.

### Scenario

**Context:** A data analytics API platform serves three customer segments. Free-tier users get 100 requests per hour with no burst allowance — requests beyond the limit are rejected immediately. Professional-tier users get 10,000 requests per hour with a burst allowance of 200 requests per minute, and requests beyond the burst limit are queued and retried after a short delay rather than rejected. Enterprise-tier users get 100,000 requests per hour with a burst allowance of 2,000 requests per minute, and the platform guarantees that rate-limited Enterprise requests are queued for up to 30 seconds before being rejected. Additionally, certain expensive endpoints (e.g., `/analytics/export`) have a separate, stricter limit of 10 requests per hour regardless of tier. A client that upgrades from Free to Professional expects the new limits to take effect within seconds, not at the next billing cycle.

**Requirements:** Design the configuration model that maps tiers to rate limits, burst allowances, and throttling policies. Describe the request evaluation pipeline that checks both the global per-client limit and the per-endpoint limit, applying the more restrictive one. Explain how tier upgrades are propagated to the rate limiter in near-real-time. Design the throttling mechanism for Professional and Enterprise tiers (queuing instead of immediate rejection). Address how the system handles a client that is at 99% of their hourly limit — should the remaining 1% be available as a burst or metered out over the remaining time?

**Expected Approach:** Model each tier as a configuration object containing hourly limit, burst capacity, burst window, and throttle policy (reject, queue-with-timeout). Use a two-stage evaluation pipeline: first check the per-endpoint limit, then the global per-client limit. Implement tier upgrades by updating the client's tier in a configuration store that the rate limiter reads on each request (or caches with a short TTL). For throttling, use a request queue with a per-tier timeout — Professional requests wait up to 5 seconds, Enterprise up to 30 seconds.

<details>
<summary>Hints</summary>

1. Model the tier configuration as a data structure: `{ hourlyLimit, burstCapacity, burstWindow, throttlePolicy, throttleTimeout }`. Store these in a configuration service or database, keyed by tier name.
2. The evaluation pipeline checks limits in order of specificity: per-endpoint limit first (most restrictive), then per-client global limit. A request must pass both checks to be allowed.
3. For tier upgrades, store the client-to-tier mapping in a fast-access store (e.g., Redis hash). When a client upgrades, update the mapping. The rate limiter reads the tier on each request (or caches it with a 5-second TTL). Reset the client's counters on upgrade so they benefit from the new limits immediately.
4. The throttle queue can be implemented as a per-client bounded buffer. When a paying client exceeds the burst limit, the request is placed in the buffer with a timeout. A background consumer drains the buffer at the allowed rate, forwarding requests to the backend. If the timeout expires before the request is processed, it is rejected with HTTP 429.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build a tiered rate limiting system with a configuration-driven evaluation pipeline, per-endpoint overrides, near-real-time tier propagation, and a throttle queue for paying customers.

1. **Tier configuration model:**
   ```
   FreeTier:
       hourlyLimit: 100
       burstCapacity: 0          // no burst allowance
       burstWindow: N/A
       throttlePolicy: REJECT    // immediate 429
       throttleTimeout: 0

   ProfessionalTier:
       hourlyLimit: 10,000
       burstCapacity: 200
       burstWindow: 60 seconds
       throttlePolicy: QUEUE     // queue and retry
       throttleTimeout: 5 seconds

   EnterpriseTier:
       hourlyLimit: 100,000
       burstCapacity: 2,000
       burstWindow: 60 seconds
       throttlePolicy: QUEUE
       throttleTimeout: 30 seconds
   ```

2. **Per-endpoint overrides:**
   ```
   EndpointOverrides:
       "/analytics/export":
           limit: 10 per hour
           appliesTo: ALL_TIERS
       "/analytics/query":
           limit: 1,000 per hour
           appliesTo: [Free]      // only restricts Free tier further
   ```

3. **Request evaluation pipeline:**
   ```
   function evaluateRequest(clientId, endpoint):
       tier = getTier(clientId)           // read from Redis, cached 5s
       config = getTierConfig(tier)

       // Stage 1: Per-endpoint limit check
       endpointOverride = getEndpointOverride(endpoint, tier)
       if endpointOverride exists:
           if endpointCounter(clientId, endpoint) >= endpointOverride.limit:
               return applyThrottlePolicy(config, request)

       // Stage 2: Burst limit check (token bucket)
       if config.burstCapacity > 0:
           if not burstBucket(clientId).tryConsume(1):
               return applyThrottlePolicy(config, request)

       // Stage 3: Hourly global limit check
       hourlyCount = INCR(hourlyKey(clientId))
       if hourlyCount > config.hourlyLimit:
           return applyThrottlePolicy(config, request)

       return ALLOW
   ```

4. **Throttle policy implementation:**
   ```
   function applyThrottlePolicy(config, request):
       if config.throttlePolicy == REJECT:
           return HTTP 429 immediately
       if config.throttlePolicy == QUEUE:
           enqueue(request, timeout=config.throttleTimeout)
           // request waits in queue; a consumer drains at allowed rate
           // if timeout expires before processing, return HTTP 429
           // if processed within timeout, forward to backend and return response
   ```
   - The queue is implemented as a per-client bounded buffer (e.g., in-memory with a max size of `burstCapacity`).
   - A background consumer processes queued requests at the sustained rate (hourlyLimit / 3600 requests per second).

5. **Tier upgrade propagation:**
   - Client-to-tier mapping is stored in Redis: `HSET client:tiers {clientId} "Professional"`.
   - When a client upgrades, the billing service updates this mapping.
   - The rate limiter reads the tier on each request (or caches with a 5-second TTL). Within 5 seconds of the upgrade, the new limits take effect.
   - On upgrade, reset the client's hourly counter and burst bucket so they immediately benefit from the higher limits. This prevents a client who used 95 of their 100 Free-tier requests from being stuck at 5 remaining after upgrading to Professional.

6. **Near-limit behavior (99% consumed):**
   - For Free tier: the remaining 1% (1 request) is available immediately — no metering. Once consumed, requests are rejected.
   - For paying tiers: the remaining 1% is available as a burst (up to the burst capacity). If the client sends a burst that exceeds the remaining hourly allowance, excess requests enter the throttle queue. This provides a smooth experience where paying customers are slowed down rather than cut off.

7. **Response headers (all tiers):**
   - `X-RateLimit-Limit`: the client's hourly limit for their tier.
   - `X-RateLimit-Remaining`: requests remaining in the current hour.
   - `X-RateLimit-Reset`: epoch timestamp when the hourly window resets.
   - `X-RateLimit-Tier`: the client's current tier name.
   - `Retry-After`: included only on 429 responses, indicating seconds to wait.

**Why this design fits:** Commercial API platforms must differentiate service levels while maintaining fairness. The tiered configuration model makes it easy to add or modify tiers without code changes. The two-stage evaluation pipeline ensures per-endpoint limits are respected regardless of tier. The throttle queue provides a premium experience for paying customers (graceful degradation instead of hard rejection), and near-real-time tier propagation ensures upgrades are immediately valuable to the customer.

</details>