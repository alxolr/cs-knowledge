# URL Shortener Design

**Track:** Design Concepts
**Difficulty Tier:** Intermediate
**Prerequisites:** [API Design](../api-design/README.md), [Caching Strategies](../caching-strategies/README.md), [Rate Limiting](../rate-limiting/README.md), [Microservices Architecture](../microservices-architecture/README.md)

## Concept Overview

A URL shortener is a service that converts long URLs into compact, shareable aliases and redirects visitors from the short link back to the original destination. Services like bit.ly and TinyURL popularized this pattern, but URL shortening is also a foundational system design exercise because it touches on many core concepts: unique ID generation, key-value storage, caching for read-heavy workloads, API design for creation and redirection endpoints, and rate limiting to prevent abuse.

Designing a URL shortener at scale requires careful thought about how short codes are generated. A naive approach — using an auto-incrementing database ID and encoding it in base62 — is simple but introduces a single point of failure and makes codes predictable. Alternatives include pre-generating random codes, using hash functions with collision handling, or deploying distributed ID generators like Snowflake. Each approach has trade-offs in uniqueness guarantees, code length, and system complexity.

The read path dominates traffic in a URL shortener: for every URL created, it may be accessed thousands or millions of times. This makes caching critical. A well-designed caching layer can serve the vast majority of redirects from memory, reducing database load to a fraction of total traffic. Meanwhile, the write path must handle bursts of creation requests, enforce rate limits to prevent spam, and validate input URLs. Combining these concerns into a cohesive microservices architecture — with separate services for creation, redirection, analytics, and abuse prevention — is the hallmark of a production-grade URL shortener.

---

## Problem 1 — Core API and Data Model Design

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design the REST API endpoints and underlying data model for a URL shortener service that supports creating short URLs, redirecting users to the original URL, and optionally allowing custom aliases. The design should define the request/response contracts, the database schema, and the HTTP semantics for redirection.

### Scenario

**Context:** A startup is building a URL shortener as its first product. The MVP needs two core capabilities: users submit a long URL and receive a short link, and anyone visiting the short link is redirected to the original URL. Some users want to choose their own custom alias (e.g., `sho.rt/my-brand`) instead of receiving a random code. The team needs a clean API contract and a database schema before writing any code.

**Requirements:** Define a POST endpoint for creating short URLs that accepts a long URL and an optional custom alias, returning the generated short URL. Define a GET endpoint that accepts a short code and redirects to the original URL using the appropriate HTTP status code. Design a database schema that stores the mapping between short codes and long URLs, along with metadata such as creation timestamp and expiration date. Explain why 301 vs. 302 redirect status codes matter for analytics.

**Expected Approach:** Design a RESTful API with clear request/response schemas. Choose between 301 (permanent) and 302 (temporary) redirects based on whether the service needs to track click analytics. Model the database table with a short code as the primary key, the original URL, creation time, expiration, and the creating user's ID.

<details>
<summary>Hints</summary>

1. A 301 redirect tells browsers to cache the redirect permanently — subsequent visits bypass your server entirely, which is fast but prevents you from counting clicks. A 302 redirect forces the browser to check with your server every time, enabling analytics tracking.
2. The short code should be the primary key or have a unique index for fast lookups during redirection.
3. For custom aliases, validate that the alias is not already taken and conforms to allowed characters (alphanumeric, hyphens) before accepting it.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Define a minimal RESTful API with two endpoints and a single-table data model optimized for fast reads.

1. **API endpoints:**
   ```
   POST /api/v1/urls
   Request body:
       { "longUrl": "https://example.com/very/long/path",
         "customAlias": "my-brand" (optional),
         "expiresAt": "2025-12-31T23:59:59Z" (optional) }
   Response (201 Created):
       { "shortCode": "my-brand",
         "shortUrl": "https://sho.rt/my-brand",
         "longUrl": "https://example.com/very/long/path",
         "expiresAt": "2025-12-31T23:59:59Z",
         "createdAt": "2025-01-15T10:30:00Z" }
   Error (409 Conflict): returned if customAlias is already taken.
   Error (400 Bad Request): returned if longUrl is missing or invalid.

   GET /{shortCode}
   Response (302 Found):
       Location: https://example.com/very/long/path
   Error (404 Not Found): returned if shortCode does not exist or has expired.
   ```

2. **Redirect status code choice:**
   - Use 302 (Found) for the redirect so that every click passes through the server, enabling analytics (click counting, referrer tracking, geographic data).
   - If analytics are not needed and maximum performance is the goal, 301 (Moved Permanently) allows browsers and CDNs to cache the redirect, reducing server load.

3. **Database schema:**
   ```
   Table: url_mappings
   +--------------+--------------+----------------------------------------------+
   | Column       | Type         | Notes                                        |
   +--------------+--------------+----------------------------------------------+
   | short_code   | VARCHAR(16)  | Primary key, unique                          |
   | long_url     | TEXT         | Original URL, indexed for deduplication      |
   | created_at   | TIMESTAMP    | Auto-set on creation                         |
   | expires_at   | TIMESTAMP    | Nullable; NULL means no expiration           |
   | user_id      | VARCHAR(64)  | Nullable; identifies the creator             |
   +--------------+--------------+----------------------------------------------+
   ```

4. **Custom alias handling:**
   - If `customAlias` is provided, validate it against an allowed character set (a-z, A-Z, 0-9, hyphens), check uniqueness against the `short_code` column, and use it directly.
   - If not provided, generate a random short code (covered in Problem 2).

5. **Expiration handling:**
   - On GET, check `expires_at` before redirecting. If the current time exceeds `expires_at`, return 404 and optionally delete the record or mark it as expired.
   - A background cleanup job periodically removes expired entries to reclaim storage.

**Why this design fits:** The two-endpoint API covers the core create-and-redirect workflow. Using `short_code` as the primary key ensures O(1) lookups for the read-heavy redirect path. The 302 redirect preserves analytics capability, and the schema supports both auto-generated and custom aliases with a single table.

</details>

---

## Problem 2 — Short Code Generation and Collision Handling

**Difficulty:** Medium
**Estimated Time:** 40 minutes

### Problem Statement

Design a short code generation strategy for a URL shortener that must produce unique, compact codes (6–8 characters) at a rate of 10,000 new URLs per second across multiple application servers. The design should handle collisions, ensure codes are not predictable or sequential, and work in a distributed environment without a single point of failure.

### Scenario

**Context:** The URL shortener from Problem 1 is gaining traction and now runs on 10 application servers behind a load balancer. The team initially used an auto-incrementing database ID encoded in base62, but this approach has two problems: (1) the codes are sequential and predictable, allowing anyone to enumerate all shortened URLs by incrementing the code, and (2) the auto-increment sequence is tied to a single database, creating a bottleneck and single point of failure at 10,000 writes per second.

**Requirements:** Design a code generation strategy that produces non-sequential, non-predictable codes of 6–8 characters using a base62 alphabet (a-z, A-Z, 0-9). The strategy must work across 10 servers without coordination for every request. Analyze the collision probability and describe how collisions are detected and resolved. Calculate the total address space for 7-character base62 codes and estimate how long until the space is exhausted at the given write rate. The design should not require a centralized counter or lock for every code generation.

**Expected Approach:** Consider multiple strategies — MD5/SHA256 hash of the long URL truncated to 7 characters, random base62 generation with database uniqueness checks, or a distributed ID generator (e.g., Snowflake-style) encoded in base62. Evaluate each on collision probability, predictability, and coordination requirements. A hybrid approach using random generation with retry-on-collision is often the best balance.

<details>
<summary>Hints</summary>

1. A 7-character base62 code has 62^7 ≈ 3.5 trillion possible values. At 10,000 new URLs per second, it takes over 11,000 years to exhaust the space — collisions are the concern, not exhaustion.
2. Using a hash of the long URL (e.g., first 7 characters of base62-encoded MD5) means the same long URL always produces the same code — useful for deduplication but problematic if different users want different short links for the same URL.
3. Random generation with a database uniqueness constraint (INSERT with unique index) is simple and effective. On collision, generate a new random code and retry. At low fill ratios (billions of codes in a trillion-sized space), the collision probability per attempt is negligible.
4. A Snowflake-style ID generator assigns each server a unique prefix, eliminating collisions entirely but producing longer, structured codes.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use random base62 code generation with a database uniqueness constraint and retry-on-collision, supplemented by a pre-generation buffer for high throughput.

1. **Code generation algorithm:**
   ```
   function generateShortCode():
       code = randomBase62String(length=7)
       return code
   ```
   - Generate a 7-character string by selecting 7 random characters from the base62 alphabet (a-z, A-Z, 0-9).
   - This is non-sequential and non-predictable because each character is chosen independently at random.

2. **Collision detection and resolution:**
   ```
   function createShortUrl(longUrl):
       for attempt in 1..MAX_RETRIES:
           code = generateShortCode()
           try:
               INSERT INTO url_mappings (short_code, long_url, created_at)
               VALUES (code, longUrl, now())
               return code
           catch UniqueConstraintViolation:
               continue    // collision — retry with a new code
       raise ServiceUnavailable("Could not generate unique code")
   ```
   - The `short_code` column has a unique index. If a collision occurs, the INSERT fails and the server retries with a new random code.
   - MAX_RETRIES = 3 is sufficient because collision probability is extremely low.

3. **Collision probability analysis:**
   - Address space: 62^7 ≈ 3.52 × 10^12 (3.52 trillion).
   - After 1 billion URLs are stored, the probability of a single collision is 1,000,000,000 / 3,520,000,000,000 ≈ 0.028% per attempt.
   - The probability of 3 consecutive collisions: (0.00028)^3 ≈ 2.2 × 10^-11 — effectively zero.
   - At 10,000 URLs/second, reaching 1 billion URLs takes about 1.16 days of continuous operation. Even at 100 billion URLs (0.32 years), collision probability per attempt is only 2.8%.

4. **Pre-generation buffer for throughput:**
   - Each application server maintains a local buffer of 1,000 pre-generated and pre-validated codes.
   - A background thread periodically generates codes, checks them against the database in batch, and fills the buffer.
   - When a creation request arrives, the server pops a code from the buffer — no database round-trip for code generation.
   - This decouples code generation latency from request latency and handles burst traffic smoothly.

5. **Alternative strategies considered:**
   - **Hash-based (MD5/SHA256 truncation):** Deterministic — same URL always gets the same code. Pro: natural deduplication. Con: different users cannot have different short links for the same URL; truncation increases collision rate compared to full random.
   - **Snowflake-style distributed ID:** Each server gets a unique 3-bit server ID prefix, and the remaining 4 characters are a local counter encoded in base62. Pro: zero collisions. Con: codes are structured and partially predictable; requires server ID coordination.
   - **Random with retry (chosen):** Best balance of simplicity, non-predictability, and distributed operation. No coordination needed between servers; collisions are handled gracefully.

6. **Space exhaustion timeline:**
   - At 10,000 URLs/second: 3.52 × 10^12 / 10,000 = 3.52 × 10^8 seconds ≈ 11,161 years to exhaust the 7-character space.
   - If the service grows to 1 million URLs/second, exhaustion occurs in ~111 years. At that scale, extending to 8 characters (62^8 ≈ 218 trillion) provides another 50× runway.

**Why random generation fits:** It requires no coordination between servers, produces non-predictable codes, and handles collisions with a simple retry. The enormous address space of 7-character base62 codes makes collisions negligible for years of operation, and the pre-generation buffer eliminates any latency impact.

</details>

---

## Problem 3 — Caching Layer for Read-Heavy Redirect Traffic

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a multi-tier caching strategy for a URL shortener that handles 500,000 redirect requests per second with a 99th-percentile latency target of 10 ms. The read-to-write ratio is 1000:1, meaning the vast majority of traffic is redirect lookups. The design should specify cache placement, eviction policies, cache invalidation for expired or deleted URLs, and how to handle cache misses gracefully.

### Scenario

**Context:** The URL shortener has grown to serve 500,000 redirects per second globally. The database can handle 5,000 read queries per second — three orders of magnitude less than the redirect traffic. Without caching, the database would be overwhelmed. The team needs a caching architecture that absorbs the read load while ensuring that expired or deleted URLs are not served from stale cache entries. Some URLs are accessed millions of times per day (viral links), while most are accessed fewer than 10 times total (long-tail links).

**Requirements:** Design a caching architecture with at least two tiers (e.g., application-level in-memory cache and a distributed cache like Redis). Specify the cache key structure, TTL strategy, and eviction policy for each tier. Explain how the system handles a cache miss (the lookup path from cache to database). Describe how expired or deleted URLs are invalidated across all cache tiers. Analyze the expected cache hit ratio given the access pattern (a few viral links and a long tail) and estimate the database query rate after caching.

**Expected Approach:** Use a two-tier cache: a local in-memory cache (e.g., LRU with a short TTL) on each application server for the hottest URLs, and a distributed Redis cache for broader coverage. On a redirect request, check the local cache first, then Redis, then the database. Cache entries include the long URL and the expiration timestamp. Use TTLs aligned with URL expiration and publish cache invalidation events when URLs are deleted.

<details>
<summary>Hints</summary>

1. The access pattern (few viral links, long tail) means a relatively small cache can capture most of the traffic. If the top 20% of URLs account for 80% of redirects (Pareto distribution), caching those URLs eliminates 80% of database reads.
2. Use a short TTL (e.g., 60 seconds) for the local in-memory cache to limit staleness, and a longer TTL (e.g., 1 hour) for the Redis cache. The local cache handles burst traffic for viral links; Redis handles the broader working set.
3. For cache invalidation on URL deletion or expiration, publish an event to a message bus (e.g., Kafka or Redis Pub/Sub). Each application server subscribes and evicts the corresponding local cache entry. The Redis entry can be deleted directly.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement a two-tier caching architecture with local in-memory caches and a distributed Redis layer, using TTL-based expiration and event-driven invalidation.

1. **Cache architecture overview:**
   ```
   Request → Local In-Memory Cache (L1) → Redis Cache (L2) → Database (L3)
   ```
   - L1: Per-server in-memory LRU cache (e.g., Guava Cache, Caffeine). Capacity: 100,000 entries per server. TTL: 60 seconds.
   - L2: Distributed Redis cluster shared across all servers. Capacity: 50 million entries. TTL: 1 hour (or aligned with URL expiration, whichever is shorter).
   - L3: Primary database (PostgreSQL or MySQL). Source of truth.

2. **Cache key structure:**
   - Key: `url:{shortCode}` (e.g., `url:aB3xK9z`).
   - Value: JSON object `{ "longUrl": "https://...", "expiresAt": "2025-12-31T23:59:59Z" }`.

3. **Redirect lookup path:**
   ```
   function redirect(shortCode):
       // L1: Check local cache
       entry = localCache.get(shortCode)
       if entry exists and not expired:
           return 302 redirect to entry.longUrl

       // L2: Check Redis
       entry = redis.get("url:" + shortCode)
       if entry exists:
           if entry.expiresAt and now() > entry.expiresAt:
               redis.delete("url:" + shortCode)
               return 404
           localCache.put(shortCode, entry)
           return 302 redirect to entry.longUrl

       // L3: Check database
       row = db.query("SELECT long_url, expires_at FROM url_mappings WHERE short_code = ?", shortCode)
       if row is null or (row.expires_at and now() > row.expires_at):
           // Cache negative result briefly to prevent repeated DB misses
           redis.setex("url:" + shortCode, 300, NULL_MARKER)
           return 404
       entry = { longUrl: row.long_url, expiresAt: row.expires_at }
       redis.setex("url:" + shortCode, 3600, serialize(entry))
       localCache.put(shortCode, entry)
       return 302 redirect to entry.longUrl
   ```

4. **Eviction policy:**
   - L1 (local): LRU eviction when capacity (100,000 entries) is reached. The short TTL (60 seconds) ensures entries are refreshed frequently, limiting staleness.
   - L2 (Redis): TTL-based eviction. Each key's TTL is set to `min(1 hour, time until URL expiration)`. Redis automatically removes expired keys.

5. **Cache invalidation for deleted/expired URLs:**
   - When a URL is explicitly deleted via the API, the deletion service:
     1. Deletes the database row.
     2. Deletes the Redis key: `redis.delete("url:" + shortCode)`.
     3. Publishes an invalidation event to a Redis Pub/Sub channel: `PUBLISH cache-invalidation shortCode`.
   - Each application server subscribes to the `cache-invalidation` channel and evicts the corresponding entry from its local cache upon receiving the event.
   - For expired URLs, the lookup path checks `expiresAt` at read time (step 3 above), so stale entries are caught even if the cache has not been explicitly invalidated.

6. **Cache hit ratio analysis:**
   - Assume a Zipf distribution: the top 1% of URLs (by access frequency) account for ~50% of all redirects, and the top 10% account for ~80%.
   - L1 (100,000 entries per server) captures the hottest URLs. With 10 servers, the aggregate L1 covers up to 1 million unique codes. Expected L1 hit ratio: ~60–70% of requests.
   - L2 (50 million entries) covers the broader working set. Expected L2 hit ratio for L1 misses: ~90–95%.
   - Combined hit ratio: ~97–99%. At 500,000 req/s, only 1–3% reach the database: 5,000–15,000 queries/s, within the database's capacity.

7. **Negative caching:**
   - Cache 404 results in Redis with a short TTL (5 minutes) to prevent repeated database lookups for non-existent or expired codes. Use a sentinel value (e.g., `NULL_MARKER`) to distinguish a cached 404 from a cache miss.

**Why two-tier caching fits:** The local cache provides sub-millisecond latency for the hottest URLs (viral links), while Redis provides millisecond-latency coverage for the broader working set. Together, they reduce database load by 97–99%, meeting the 10 ms p99 latency target and protecting the database from the extreme read-to-write ratio.

</details>

---

## Problem 4 — Scaling the URL Shortener to Multiple Regions

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design a multi-region deployment architecture for a URL shortener that serves users across North America, Europe, and Asia-Pacific with sub-50 ms redirect latency from any region. The system must handle 1 million redirects per second globally, support URL creation from any region, and maintain consistency so that a URL created in one region is resolvable from all others within a bounded time window. Address data replication, conflict resolution for custom aliases, and failover when an entire region goes offline.

### Scenario

**Context:** The URL shortener has expanded globally. Users in Europe experience 200 ms redirect latency because all traffic routes to a single US-East data center. The product team wants to deploy the service in three regions (US-East, EU-West, Asia-Pacific) so that redirects are served from the nearest region. URL creation also happens in all regions — a marketing team in London creates a short link that must be accessible from Tokyo within seconds. The team chose custom aliases as a premium feature, but two users in different regions could simultaneously request the same custom alias, creating a conflict.

**Requirements:** Design the multi-region architecture including DNS routing, regional application stacks, and database replication. Specify whether the database uses single-leader, multi-leader, or leaderless replication and justify the choice. Describe how a URL created in EU-West becomes available in Asia-Pacific, including the replication lag and its impact on user experience. Design a conflict resolution strategy for custom alias collisions across regions. Explain the failover process when one region becomes unavailable — how traffic is rerouted and how data consistency is maintained during and after the outage.

**Expected Approach:** Use geographic DNS routing to direct users to the nearest region. Deploy a full application stack (API servers, Redis cache, database replica) in each region. Use a single-leader database with read replicas in each region for redirects (reads), and route all writes to the leader. For custom alias conflicts, use a synchronous check against the leader before confirming the alias. For failover, promote a read replica to leader and update DNS. Accept eventual consistency for redirects (a few seconds of replication lag) while enforcing strong consistency for alias uniqueness.

<details>
<summary>Hints</summary>

1. Geographic DNS routing (e.g., AWS Route 53 latency-based routing) directs each user to the nearest regional endpoint. This alone reduces redirect latency from 200 ms to sub-50 ms for most users.
2. Single-leader replication is simpler for writes (no conflict resolution needed for auto-generated codes) but introduces write latency for non-leader regions. Multi-leader replication allows local writes but requires conflict resolution for custom aliases.
3. For custom alias uniqueness across regions, consider a two-phase approach: the regional server tentatively reserves the alias locally, then confirms with the leader (or a global coordination service) before committing. If the alias is taken, the user receives a 409 Conflict response.
4. During a region failover, cached redirects continue to work from the surviving regions' caches. New URL creations may be briefly unavailable if the failed region was the write leader — promote a replica and update the leader endpoint.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Deploy a multi-region architecture with geographic DNS routing, single-leader database replication with regional read replicas, and a global coordination layer for custom alias uniqueness.

1. **Regional architecture:**
   ```
   Each region (US-East, EU-West, Asia-Pacific):
   ├── Load Balancer
   ├── Application Servers (redirect + creation services)
   ├── Redis Cache (regional, populated from local DB replica)
   └── Database Read Replica (async replication from leader)

   Global:
   ├── Database Leader (US-East, primary)
   ├── DNS (latency-based routing to nearest region)
   └── Global Alias Coordination Service (optional, or use leader directly)
   ```

2. **DNS and traffic routing:**
   - Use latency-based DNS routing so that a user in Paris resolves the shortener domain to the EU-West endpoint, and a user in Tokyo resolves to Asia-Pacific.
   - Each regional endpoint handles both redirects (reads) and URL creation (writes).

3. **Database replication strategy — single-leader with regional read replicas:**
   - The database leader resides in US-East. All write operations (URL creation) are forwarded to the leader.
   - Each region has an asynchronous read replica. Redirect lookups query the local replica (or the regional Redis cache).
   - Replication lag: typically 50–200 ms for cross-region async replication. A URL created in EU-West is written to the US-East leader and replicated to Asia-Pacific within 200 ms.
   - **Why single-leader:** Auto-generated short codes are random and unique by construction (Problem 2), so write conflicts for auto-generated codes are impossible. The only conflict risk is custom aliases, which are handled separately.

4. **Write path for URL creation:**
   ```
   function createUrl(longUrl, customAlias, region):
       code = customAlias or generateShortCode()

       // Forward write to the leader (US-East)
       result = leaderDb.insert(code, longUrl)
       if result == UniqueConstraintViolation:
           if customAlias:
               return 409 Conflict
           else:
               retry with new random code (up to 3 times)

       // Populate regional Redis cache immediately (don't wait for replication)
       regionalRedis.set("url:" + code, { longUrl, expiresAt })
       return 201 Created
   ```
   - Write latency from EU-West or Asia-Pacific includes the cross-region round trip to US-East (~100–150 ms). This is acceptable for URL creation (infrequent operation) but not for redirects.

5. **Custom alias conflict resolution:**
   - Custom alias uniqueness is enforced by the leader database's unique constraint on `short_code`.
   - If two users in different regions simultaneously request the same alias, both writes go to the leader. The first INSERT succeeds; the second fails with a unique constraint violation and returns 409 Conflict.
   - No distributed locking is needed — the database's unique index is the coordination mechanism.

6. **Read path for redirects (optimized for latency):**
   ```
   function redirect(shortCode, region):
       // Check regional Redis cache (sub-ms)
       entry = regionalRedis.get("url:" + shortCode)
       if entry: return 302 redirect

       // Check regional DB read replica (~5 ms)
       row = regionalReplica.query(shortCode)
       if row:
           regionalRedis.set("url:" + shortCode, row, TTL=3600)
           return 302 redirect

       // URL not found (may be replication lag — optionally check leader)
       return 404
   ```
   - For newly created URLs that have not yet replicated, the creating region's Redis cache is populated immediately (step 4), so redirects from the same region work instantly. Other regions may see a 404 for up to 200 ms until replication completes.

7. **Failover when a region goes offline:**
   - **If a non-leader region fails (e.g., EU-West):** DNS health checks detect the failure and reroute EU traffic to the next-nearest region (e.g., US-East). No data loss — the leader and other replicas are unaffected. When EU-West recovers, its replica catches up from the leader's replication log.
   - **If the leader region fails (US-East):** Promote the EU-West read replica to leader (manual or automated failover). Update the internal leader endpoint so all regions send writes to the new leader. Accept that any writes not yet replicated from the old leader may be lost (typically a few hundred milliseconds of data). After the old leader recovers, it rejoins as a replica.
   - **During failover:** Redirects continue to work from regional caches and replicas. URL creation is briefly unavailable (seconds to minutes) until the new leader is promoted.

8. **Consistency guarantees:**
   - Redirects: eventually consistent (replication lag of 50–200 ms). Acceptable because a user creating a URL and immediately sharing it will typically wait longer than 200 ms before someone clicks the link.
   - Custom alias uniqueness: strongly consistent (enforced at the single leader). No two users can claim the same alias.

**Why single-leader multi-region fits:** The URL shortener is overwhelmingly read-heavy (1000:1 ratio). Reads are served from local replicas and caches with sub-50 ms latency. Writes are infrequent and can tolerate the cross-region round trip to the leader. Single-leader replication avoids the complexity of multi-leader conflict resolution while providing strong uniqueness guarantees for custom aliases.

</details>

---

## Problem 5 — Abuse Prevention and Analytics Pipeline

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design an abuse prevention system and a real-time analytics pipeline for a URL shortener that processes 1 million link creations per day and 500 million redirects per day. The abuse prevention system must detect and block malicious URLs (phishing, malware, spam) at creation time and retroactively disable links that are later reported or identified as harmful. The analytics pipeline must track click metrics (total clicks, unique visitors, geographic distribution, referrer sources) per short URL and make them available to link creators within 60 seconds of the click occurring.

### Scenario

**Context:** The URL shortener is being exploited by bad actors who create short links pointing to phishing sites and distribute them via email and social media. Because the short URL masks the true destination, users cannot tell they are clicking a malicious link. Meanwhile, legitimate users are requesting analytics dashboards showing how their links perform — click counts, geographic breakdown, and top referrers. The current system has no abuse detection and no analytics beyond a simple total click counter stored in the database.

**Requirements:** Design an abuse prevention pipeline that scans URLs at creation time against known blocklists, performs heuristic analysis (e.g., newly registered domains, suspicious URL patterns), and assigns a risk score. URLs above a risk threshold should be blocked; borderline URLs should be flagged for manual review. Design a mechanism to retroactively disable URLs that are reported by users or flagged by periodic re-scanning. For analytics, design a pipeline that captures click events (timestamp, short code, IP address, User-Agent, referrer, country) and aggregates them into per-URL metrics available within 60 seconds. The analytics system must handle 500 million events per day without impacting redirect latency.

**Expected Approach:** For abuse prevention, implement a multi-stage scanning pipeline at URL creation time: check against blocklists (Google Safe Browsing, PhishTank), run heuristic checks, and compute a risk score. Use an asynchronous re-scanning job for existing URLs. For analytics, emit click events to a message queue (Kafka), process them with a stream processor (Flink or Kafka Streams) that aggregates metrics in tumbling windows, and store the results in a time-series or OLAP database. Serve analytics queries from this pre-aggregated store.

<details>
<summary>Hints</summary>

1. Abuse scanning at creation time should be synchronous for blocklist checks (fast, sub-100 ms) but asynchronous for deeper analysis (e.g., fetching the destination page and scanning its content). Allow the URL to be created in a "pending" state while deep scanning completes, and activate it only after it passes.
2. For retroactive disabling, add a `status` field to the URL mapping (active, disabled, pending_review). The redirect service checks this field (cached) and returns a warning page or 404 for disabled URLs instead of redirecting.
3. The analytics pipeline must not add latency to the redirect path. Emit click events asynchronously — fire-and-forget to a Kafka topic — so the redirect response is sent before the event is processed.
4. For 500 million events per day (~5,800 events/second average, with peaks of 20,000+/s), Kafka with a stream processor can aggregate metrics in near-real-time. Store aggregated results (not raw events) for dashboard queries to keep storage manageable.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build a two-part system: a multi-stage abuse prevention pipeline integrated into the URL creation flow, and an event-driven analytics pipeline that processes click data asynchronously via Kafka and a stream processor.

1. **Abuse prevention — creation-time scanning pipeline:**
   ```
   function createUrl(longUrl, customAlias):
       // Stage 1: Blocklist check (synchronous, <50 ms)
       if isBlocklisted(longUrl):    // Google Safe Browsing API, PhishTank
           return 403 Forbidden ("URL is on a known blocklist")

       // Stage 2: Heuristic risk scoring (synchronous, <100 ms)
       riskScore = computeRiskScore(longUrl)
       // Heuristics: domain age < 7 days, excessive URL-encoded characters,
       // IP address instead of domain, known URL shortener chaining,
       // suspicious TLD (.xyz, .tk with no established reputation)

       if riskScore > HIGH_THRESHOLD:
           return 403 Forbidden ("URL flagged as potentially malicious")

       // Stage 3: Create URL with appropriate status
       if riskScore > MEDIUM_THRESHOLD:
           status = "pending_review"
       else:
           status = "active"

       code = customAlias or generateShortCode()
       db.insert(code, longUrl, status)

       // Stage 4: Asynchronous deep scan (non-blocking)
       if status == "pending_review":
           enqueue(deepScanQueue, { shortCode: code, longUrl: longUrl })

       return 201 Created with { shortCode, status }
   ```

2. **Abuse prevention — deep scanning (asynchronous):**
   - A background worker consumes from `deepScanQueue`.
   - It fetches the destination page, scans the HTML content for phishing indicators (fake login forms, brand impersonation), checks SSL certificate validity, and runs the URL through machine learning classifiers.
   - Based on the deep scan result, the worker updates the URL status to "active" (safe) or "disabled" (malicious).
   - Deep scan completes within 30–60 seconds of URL creation.

3. **Abuse prevention — retroactive scanning and reporting:**
   - **User reports:** A `POST /api/v1/urls/{shortCode}/report` endpoint allows anyone to flag a URL. Reported URLs are immediately set to "pending_review" and enqueued for deep scanning.
   - **Periodic re-scanning:** A daily batch job re-checks all active URLs against updated blocklists. URLs that appear on new blocklist entries are disabled automatically.
   - **Redirect behavior for non-active URLs:**
     ```
     function redirect(shortCode):
         entry = lookupWithStatus(shortCode)
         if entry.status == "active":
             return 302 redirect to entry.longUrl
         if entry.status == "pending_review":
             return interstitial warning page ("This link is under review. Proceed at your own risk.")
         if entry.status == "disabled":
             return 404 with message ("This link has been disabled for violating our terms.")
     ```

4. **Analytics pipeline — event capture:**
   - The redirect service emits a click event asynchronously after sending the 302 response:
     ```
     function redirect(shortCode):
         entry = cache.get(shortCode)
         response = 302 redirect to entry.longUrl
         // Fire-and-forget: emit event after response is sent
         emitAsync(kafkaTopic="click-events", {
             shortCode, timestamp: now(), ip, userAgent, referrer, country: geoLookup(ip)
         })
         return response
     ```
   - The event emission does not block the redirect response. If Kafka is temporarily unavailable, events are buffered locally and retried.

5. **Analytics pipeline — stream processing:**
   ```
   Kafka Topic: click-events
       → Stream Processor (Kafka Streams or Flink)
           → Tumbling window aggregation (60-second windows)
           → Output: per-shortCode metrics per window
       → Analytics Store (ClickHouse, TimescaleDB, or DynamoDB)
   ```
   - The stream processor consumes click events and aggregates them in 60-second tumbling windows:
     - Total clicks per short code per window.
     - Unique visitors (approximate, using HyperLogLog).
     - Click count by country.
     - Top 10 referrers.
   - Aggregated results are written to the analytics store, keyed by `(shortCode, windowTimestamp)`.

6. **Analytics query API:**
   ```
   GET /api/v1/urls/{shortCode}/analytics?period=24h
   Response:
       { "totalClicks": 45230,
         "uniqueVisitors": 31200,
         "clicksByCountry": { "US": 18000, "UK": 8500, "DE": 5200, ... },
         "topReferrers": [ "twitter.com", "facebook.com", "linkedin.com", ... ],
         "clicksOverTime": [ { "timestamp": "...", "clicks": 1200 }, ... ] }
   ```
   - Queries read from the pre-aggregated analytics store, not from raw events. This keeps query latency low (sub-100 ms) regardless of total event volume.

7. **Scale analysis:**
   - 500 million clicks/day = ~5,800 events/second average. Kafka handles this easily with a single partition; use 8–16 partitions for parallelism and fault tolerance.
   - Storage: each aggregated window record is ~200 bytes. At 10 million unique short codes with 1,440 windows/day (one per minute), daily storage is ~2.9 TB. Retain detailed per-minute data for 30 days and roll up to hourly/daily aggregates for long-term storage.
   - The analytics pipeline is fully decoupled from the redirect path — a spike in analytics processing does not affect redirect latency.

**Why this design fits:** Abuse prevention is critical for trust — users will not click short links if the service is known for hosting malicious content. The multi-stage approach balances creation latency (fast blocklist check) with thoroughness (async deep scan). The analytics pipeline uses proven event streaming patterns (Kafka + stream processing) to deliver near-real-time metrics without impacting the latency-sensitive redirect path.

</details>
