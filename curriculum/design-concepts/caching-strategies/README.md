# Caching Strategies

**Track:** Design Concepts
**Difficulty Tier:** Beginner
**Prerequisites:** None

## Concept Overview

Caching is the practice of storing copies of frequently accessed data in a fast-access storage layer so that future requests can be served more quickly than fetching from the original source. Caches appear at every level of a modern system — from CPU registers and browser storage to in-memory stores like Redis and distributed CDN edge nodes. The core trade-off is always the same: you gain speed and reduce load on the primary data store, but you introduce the challenge of keeping cached data consistent with the source of truth.

Several well-known caching patterns govern how data flows between the cache and the backing store. In the cache-aside (lazy-loading) pattern, the application checks the cache first and only queries the database on a miss, writing the result back into the cache for next time. Write-through caching writes data to both the cache and the database synchronously, guaranteeing consistency at the cost of higher write latency. Write-behind (write-back) caching writes to the cache immediately and asynchronously flushes changes to the database, improving write performance but risking data loss if the cache fails before the flush. Read-through caching places the cache itself in front of the database so the caller never interacts with the store directly.

Equally important are the policies that control what stays in the cache and for how long. Time-to-live (TTL) assigns an expiration timestamp to each entry, ensuring stale data is eventually evicted. When the cache reaches capacity, an eviction policy decides which entry to remove. Least Recently Used (LRU) evicts the entry that has not been accessed for the longest time, while Least Frequently Used (LFU) evicts the entry with the fewest accesses. CDN caching extends these ideas to geographically distributed edge servers, bringing content closer to end users and offloading origin servers. Cache invalidation — deciding when and how to remove or refresh stale entries — is famously one of the hardest problems in computer science, and understanding strategies like event-driven invalidation, TTL-based expiration, and versioned keys is essential for building reliable systems.

---

## Problem 1 — Designing a Cache-Aside Layer for a Product Catalog

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a caching layer for an e-commerce product catalog service that uses the cache-aside (lazy-loading) pattern. The service currently reads product details directly from a relational database for every request, and response times are climbing as traffic grows. Your design should describe how the application interacts with both the cache and the database, how cache misses are handled, and what happens when product data is updated.

### Scenario

**Context:** An online store serves a product catalog containing 50,000 items. The catalog page receives 10,000 requests per minute, but only about 5% of products account for 80% of views. Every request currently hits the PostgreSQL database, causing high read latency during peak hours.

**Requirements:** Introduce an in-memory cache (e.g., Redis) between the application and the database. On a cache hit, return the product data directly from the cache. On a cache miss, query the database, populate the cache with the result, and then return it. When a product is updated through the admin panel, the stale cache entry must be handled. The design should describe the read path, the write/update path, and how the cache is populated.

**Expected Approach:** Describe the cache-aside read flow (check cache → miss → query DB → store in cache → return). Explain why the application — not the cache — is responsible for loading data. Discuss how updates are handled (invalidate the cache entry on write so the next read triggers a fresh load). Consider what happens on cache failure (the system degrades to direct DB reads rather than failing entirely).

<details>
<summary>Hints</summary>

1. Draw the read path as a sequence: application → cache lookup → (miss) → database query → cache write → response. On a hit, the database is skipped entirely.
2. For updates, the simplest strategy is to delete the cache entry when the product is modified, rather than trying to update the cache and the database simultaneously.
3. Think about graceful degradation — if the cache is unavailable, the application should fall back to querying the database directly, not return an error.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement the cache-aside pattern where the application manages all cache interactions explicitly.

1. **Read path:**
   - Application receives a request for product ID `P123`.
   - Check the cache (e.g., Redis `GET product:P123`).
   - **Cache hit:** Return the cached product data immediately.
   - **Cache miss:** Query PostgreSQL for `P123`, serialize the result, store it in Redis with a TTL (e.g., 15 minutes), and return the data.

2. **Write/update path:**
   - When an admin updates product `P123` in the database, the application deletes the corresponding cache key (`DEL product:P123`).
   - The next read request for `P123` will miss the cache and reload fresh data from the database.

3. **Cache failure handling:**
   - Wrap cache operations in a try/catch. If Redis is unreachable, skip the cache and query the database directly.
   - The system operates at reduced performance but remains functional.

4. **Key design decisions:**
   - Cache-aside is chosen because reads vastly outnumber writes, and the application already knows when data changes.
   - TTL acts as a safety net — even if an invalidation is missed, stale data expires automatically.
   - Only the hot 5% of products will naturally stay in cache due to access patterns, keeping memory usage efficient.

**Why cache-aside fits:** The application has full control over when data enters the cache, making it straightforward to reason about consistency. It degrades gracefully and works well for read-heavy workloads with infrequent updates.

</details>

---

## Problem 2 — Choosing Between Write-Through and Write-Behind for a Session Store

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a session storage system for a web application and decide whether to use a write-through or write-behind caching strategy. Users interact with the application frequently (clicking, adding items to a cart, updating preferences), and each interaction modifies session data. The design must balance write performance with data durability, and you should justify your choice of strategy by analyzing the trade-offs.

### Scenario

**Context:** A web application maintains user session data (shopping cart contents, UI preferences, authentication tokens) that is updated on nearly every request. Sessions are stored in a persistent database for durability, but writing to the database on every click creates a bottleneck. The team is evaluating two approaches: write-through (write to cache and database synchronously on every update) and write-behind (write to cache immediately, flush to database asynchronously in batches).

**Requirements:** Describe both write-through and write-behind strategies in the context of session storage. Analyze the trade-offs of each: write latency, data durability, complexity, and failure modes. Recommend one strategy for this use case and justify the choice. The design should address what happens if the cache node crashes before data is persisted.

**Expected Approach:** Explain write-through (every write goes to cache and DB together — strong consistency, higher latency) and write-behind (writes go to cache first, DB is updated later — lower latency, risk of data loss). Evaluate which trade-offs are acceptable for session data. Consider that losing a few seconds of cart updates is tolerable, but losing an authentication token could be problematic.

<details>
<summary>Hints</summary>

1. Write-through guarantees that the database always has the latest data, but every user click incurs a database write, which may be slow under load.
2. Write-behind batches multiple updates into a single database write, dramatically reducing DB load, but if the cache crashes before flushing, recent changes are lost.
3. Consider a hybrid: use write-behind for non-critical session data (cart, preferences) and write-through for security-sensitive data (auth tokens).

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Compare write-through and write-behind strategies, then recommend write-behind with selective write-through for critical fields.

1. **Write-through analysis:**
   - Every session update writes to both Redis and the database synchronously.
   - **Pro:** Database is always consistent with the cache — no data loss on cache failure.
   - **Con:** Every user interaction triggers a DB write, creating a bottleneck at high traffic. Write latency is the sum of cache write + DB write.

2. **Write-behind analysis:**
   - Session updates write to Redis immediately. A background process flushes changes to the database every N seconds or after N accumulated changes.
   - **Pro:** Write latency is just the cache write (sub-millisecond). DB load is reduced because multiple updates are batched into one write.
   - **Con:** If Redis crashes before a flush, recent updates are lost. Adds complexity (background flush process, retry logic).

3. **Recommendation — write-behind with selective write-through:**
   - Use write-behind for cart contents and UI preferences. Losing a few seconds of cart changes is acceptable and recoverable.
   - Use write-through for authentication tokens and security-sensitive fields. These must be durable immediately.
   - The flush interval for write-behind is set to 5 seconds, limiting maximum data loss to 5 seconds of non-critical updates.

4. **Crash recovery:**
   - On cache restart, sessions without a recent flush are reconstructed from the database (last flushed state). Users may need to re-add the last few cart items.
   - Auth tokens are always in the database (write-through), so authentication is unaffected.

**Why this hybrid fits:** Session data has mixed criticality. Write-behind handles the high-frequency, low-criticality updates efficiently, while write-through protects the small subset of security-sensitive data.

</details>

---

## Problem 3 — Configuring TTL and LRU Eviction for an API Response Cache

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Design the TTL and eviction policy configuration for an API gateway that caches responses from multiple backend services. Each backend has different data freshness requirements — some data changes every few seconds, while other data is updated daily. The cache has a fixed memory budget. Your design should specify how TTL values are assigned per endpoint, how the LRU eviction policy reclaims space, and how the two mechanisms interact.

### Scenario

**Context:** An API gateway fronts three backend services: a stock price service (data changes every second), a weather forecast service (updated every 30 minutes), and a store locator service (updated weekly). The gateway has a Redis cache with 2 GB of memory. Without eviction controls, the cache fills up with stale stock prices that crowd out more cacheable weather and store data.

**Requirements:** Assign appropriate TTL values to each endpoint category based on data freshness needs. Configure LRU as the eviction policy so that when memory is full, the least recently accessed entries are removed first. Explain how TTL expiration and LRU eviction work together — which mechanism fires first in different scenarios. The design should prevent low-value, rapidly-changing data from monopolizing cache space.

**Expected Approach:** Set short TTLs for volatile data (stock prices: 1–2 seconds) and longer TTLs for stable data (store locator: 24 hours). Explain that TTL proactively removes stale entries before they are accessed, while LRU reactively removes cold entries when space is needed. Discuss how short-TTL entries naturally free space quickly, preventing them from crowding out long-TTL entries.

<details>
<summary>Hints</summary>

1. TTL and LRU serve different purposes: TTL ensures correctness (no stale data), while LRU ensures the cache stays within its memory budget. Both can trigger removal of the same entry, but for different reasons.
2. Stock prices with a 1-second TTL will expire and free memory almost immediately, so they rarely accumulate enough to trigger LRU eviction. Weather data with a 30-minute TTL stays longer and is more likely to be evicted by LRU if memory pressure rises.
3. Consider using a prefix-based key scheme (e.g., `stock:AAPL`, `weather:NYC`, `store:12345`) so you can monitor cache usage per service and adjust TTLs if needed.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Assign per-endpoint TTLs based on data volatility and configure LRU eviction as the memory-pressure safety net.

1. **TTL assignment by endpoint:**
   - Stock price endpoints: TTL = 1 second. Data is nearly real-time; caching even for 1 second reduces backend load during bursts.
   - Weather forecast endpoints: TTL = 15 minutes. Forecasts update every 30 minutes, so a 15-minute TTL balances freshness with cache efficiency.
   - Store locator endpoints: TTL = 24 hours. Store locations rarely change; a daily refresh is sufficient.

2. **LRU eviction configuration:**
   - Set Redis `maxmemory-policy` to `allkeys-lru`.
   - When the 2 GB limit is reached, Redis evicts the key that has not been accessed for the longest time, regardless of its TTL.

3. **Interaction between TTL and LRU:**
   - Stock price entries expire via TTL within 1 second, so they rarely accumulate in memory. LRU almost never needs to evict them.
   - Weather entries live up to 15 minutes. If many cities are cached and memory fills, LRU evicts the least-accessed city forecasts first.
   - Store locator entries have long TTLs but are accessed infrequently for less popular stores. LRU may evict rarely-accessed store entries even before their 24-hour TTL expires.
   - **TTL fires first** when data becomes stale before memory pressure occurs. **LRU fires first** when memory fills before entries expire.

4. **Monitoring and tuning:**
   - Track cache hit rate per prefix (`stock:*`, `weather:*`, `store:*`).
   - If store locator hit rate drops because LRU evicts entries too aggressively, consider reducing weather TTL or increasing cache memory.

**Why TTL + LRU together:** TTL guarantees data freshness (correctness), while LRU guarantees the cache fits in memory (resource management). Neither alone is sufficient — TTL without eviction can exceed memory if entries accumulate faster than they expire, and LRU without TTL can serve stale data indefinitely.

</details>

---

## Problem 4 — Designing a CDN Caching Strategy for a Media-Heavy Website

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a CDN caching strategy for a media-heavy news website that serves articles, images, and videos to a global audience. The site publishes breaking news that must appear quickly, while older articles and media assets rarely change. Your design should specify what content is cached at the CDN edge, how cache-control headers are configured, how content is purged when corrections are made, and how the origin server's load is minimized.

### Scenario

**Context:** A news website serves 5 million page views per day across 40 countries. Each article page includes HTML, 5–10 images, and occasionally an embedded video. Breaking news articles are published every few minutes and sometimes corrected within the first hour. Older articles (more than 24 hours old) are rarely edited. All media assets (images, videos) are immutable once uploaded — corrections involve uploading a new asset with a different URL. The origin server cluster is located in a single region and struggles with global latency and traffic spikes during major events.

**Requirements:** Define caching rules for three content types: HTML article pages, image assets, and video assets. Specify `Cache-Control` header values for each type. Design a purge mechanism for correcting breaking news articles. Explain how the CDN reduces origin load during traffic spikes. Address how users in different regions experience content freshness.

**Expected Approach:** Cache immutable assets (images, videos) aggressively with long TTLs and versioned URLs. Cache article HTML with shorter TTLs and use CDN purge APIs for urgent corrections. Use `stale-while-revalidate` to serve slightly stale content while fetching updates in the background. Consider cache key design to avoid serving the wrong content variant.

<details>
<summary>Hints</summary>

1. Immutable assets with versioned URLs (e.g., `/images/photo-abc123.jpg`) can be cached for a year because a new version gets a new URL — you never need to invalidate the old one.
2. For article HTML, a short TTL (e.g., 60 seconds) combined with `stale-while-revalidate` lets the CDN serve a cached version instantly while fetching a fresh copy in the background. Users see fast responses with at most 60 seconds of staleness.
3. CDN purge APIs let you invalidate a specific URL across all edge nodes when a breaking news correction is published. This is faster than waiting for TTL expiration.
4. Consider using `Vary` headers or separate cache keys if the same URL serves different content based on device type or language.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Layer CDN caching rules by content type, using aggressive caching for immutable assets and short-TTL caching with purge support for dynamic content.

1. **Image assets:**
   - `Cache-Control: public, max-age=31536000, immutable`
   - Images use content-hashed filenames (e.g., `/img/hero-a3f8c2.jpg`). A new image gets a new URL, so the old cached version never needs invalidation.
   - CDN caches images for up to 1 year. Cache hit rate for images should approach 99%.

2. **Video assets:**
   - Same strategy as images: `Cache-Control: public, max-age=31536000, immutable` with content-hashed URLs.
   - Videos are large, so CDN edge caching dramatically reduces origin bandwidth.

3. **Article HTML pages:**
   - `Cache-Control: public, max-age=60, stale-while-revalidate=300`
   - CDN serves the cached HTML for up to 60 seconds. After 60 seconds, the next request triggers a background revalidation while still serving the stale version (for up to 300 seconds total).
   - For breaking news corrections, the CMS triggers a CDN purge API call for the specific article URL, forcing all edge nodes to fetch the corrected version on the next request.

4. **Purge mechanism for corrections:**
   - The CMS publishes an event when an article is updated. A webhook calls the CDN's purge API with the article URL path.
   - Purge propagates to all edge nodes within seconds. The next request from any region fetches the corrected article from the origin.

5. **Origin load reduction during spikes:**
   - During a major news event, millions of users request the same article. The CDN serves the cached HTML from edge nodes, and only one request per edge per TTL window reaches the origin (request collapsing).
   - Images and videos are served entirely from edge cache, eliminating origin bandwidth for static assets.

6. **Regional freshness:**
   - All regions see the same TTL behavior. A user in Tokyo and a user in London both get content that is at most 60 seconds stale under normal conditions, or immediately fresh after a purge.

**Why this layered approach fits:** Different content types have fundamentally different change frequencies. Immutable assets benefit from aggressive, long-lived caching. Dynamic HTML needs short TTLs with purge support for urgent updates. The combination maximizes cache hit rates while maintaining acceptable freshness.

</details>

---

## Problem 5 — Designing a Cache Invalidation Strategy for a Social Media Profile Service

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a cache invalidation strategy for a social media platform's profile service. User profiles are read millions of times per day (viewed by followers, shown in comment threads, displayed in search results) but updated infrequently (a user changes their display name or avatar perhaps once a month). The challenge is ensuring that when a profile is updated, all cached copies across multiple services and cache layers are invalidated promptly, without introducing excessive cache misses or complex coordination between services.

### Scenario

**Context:** A social media platform has a profile service backed by a database. Profile data is cached in three places: (1) a centralized Redis cluster used by the profile API, (2) a local in-process cache in each of the 12 application server instances, and (3) a CDN edge cache for public profile pages. When a user updates their display name, the change must propagate to all three cache layers. Currently, the team relies solely on TTL-based expiration (5-minute TTL on Redis, 60-second TTL on local caches, 10-minute TTL on CDN), which means a user can see their old name for up to 10 minutes after changing it. Users have complained about this delay.

**Requirements:** Design an invalidation strategy that reduces the propagation delay to under 5 seconds for the user who made the change and under 30 seconds for other users viewing the profile. The strategy must handle all three cache layers. It should not require tight coupling between the profile service and every consuming service. The design must account for failure scenarios (e.g., a message is lost, a cache node is temporarily unreachable). Explain the trade-offs between consistency and performance in your approach.

**Expected Approach:** Use event-driven invalidation (publish a profile-updated event) to actively invalidate caches rather than waiting for TTL expiration. For the updating user, use a read-your-writes pattern to bypass the cache and read directly from the database immediately after the update. For other users, the event propagates to each cache layer. Combine event-driven invalidation with short TTLs as a fallback safety net. Address failure modes with retry logic and TTL as the ultimate consistency guarantee.

<details>
<summary>Hints</summary>

1. A publish/subscribe system (e.g., Redis Pub/Sub, Kafka, or an internal event bus) can broadcast a "profile updated" event to all cache layers simultaneously, rather than having the profile service call each cache individually.
2. For the updating user, the "read-your-writes" pattern means the profile API detects that the current user just made an update (e.g., via a short-lived flag or version token) and bypasses the cache to read directly from the database for the next few seconds.
3. TTL should remain as a safety net even with event-driven invalidation. If an invalidation event is lost, the stale entry still expires within the TTL window. This is the "belt and suspenders" approach.
4. For the CDN layer, trigger a purge API call as part of the invalidation event handler, similar to how a CMS purges article pages.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Combine event-driven invalidation across all cache layers with a read-your-writes guarantee for the updating user, and retain TTLs as a fallback consistency mechanism.

1. **Event-driven invalidation pipeline:**
   - When a user updates their profile, the profile service writes to the database and publishes a `profile.updated` event (containing the user ID and a timestamp) to a message bus (e.g., Kafka or Redis Streams).
   - Three consumers subscribe to this event:
     - **Redis cache consumer:** Deletes the key `profile:{userId}` from the centralized Redis cluster.
     - **Local cache consumer:** Each application server instance subscribes to the event and evicts `profile:{userId}` from its in-process cache.
     - **CDN cache consumer:** Calls the CDN purge API for the public profile URL `/profiles/{userId}`.

2. **Read-your-writes for the updating user:**
   - After the profile update API returns, the response includes a version token (e.g., a timestamp or version number).
   - The client includes this token in subsequent requests (via a header or cookie).
   - The profile API compares the token against the cached entry's version. If the cache entry is older than the token, it bypasses the cache and reads from the database.
   - This guarantees the updating user sees their own changes immediately (under 5 seconds), regardless of cache propagation delays.

3. **TTL as a safety net:**
   - Retain existing TTLs: Redis = 5 minutes, local cache = 60 seconds, CDN = 10 minutes.
   - If an invalidation event is lost (message bus hiccup, consumer crash), the stale entry still expires within the TTL window.
   - TTL is the "worst case" consistency guarantee; event-driven invalidation is the "best case" that usually resolves staleness in seconds.

4. **Failure handling:**
   - **Lost event:** TTL expiration handles it. Maximum staleness = longest TTL (10 minutes for CDN).
   - **Cache node unreachable:** The consumer retries the invalidation with exponential backoff. If the node recovers within the TTL window, the retry succeeds. If not, TTL handles it.
   - **Message bus down:** The profile service still writes to the database. Caches serve stale data until TTLs expire. An alert fires so the team can investigate.

5. **Consistency vs. performance trade-offs:**
   - The updating user gets strong consistency (read-your-writes) at the cost of one uncached database read.
   - Other users get eventual consistency (typically under 5 seconds via events, worst case under 10 minutes via TTL).
   - The system avoids the "thundering herd" problem by invalidating specific keys rather than flushing entire caches.

**Why event-driven invalidation with TTL fallback fits:** Pure TTL-based invalidation is too slow for user-facing profile updates. Pure event-driven invalidation is fragile — lost messages cause indefinite staleness. Combining both gives fast propagation in the common case and guaranteed eventual consistency in failure scenarios. The read-your-writes pattern solves the most visible problem (a user seeing their own stale data) without adding complexity to the general read path.

</details>