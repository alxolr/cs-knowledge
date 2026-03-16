# News Feed Design

**Track:** Design Concepts
**Difficulty Tier:** Advanced
**Prerequisites:** [Caching Strategies](../caching-strategies/README.md), [Microservices Architecture](../microservices-architecture/README.md), [Search System Design](../search-system-design/README.md)

## Concept Overview

A news feed is the central surface of most social platforms — the continuously updating stream of posts, photos, links, and activity from people and pages a user follows. Designing a news feed at scale requires solving a deceptively hard problem: for every user who opens the app, the system must assemble a personalized, ranked list of content drawn from potentially thousands of followed accounts, each producing content at different rates. Services like Facebook, Twitter/X, LinkedIn, and Instagram serve billions of feed requests per day, each backed by a pipeline that spans ingestion, fan-out, ranking, caching, and delivery.

The core architectural tension in news feed design is the fan-out strategy. When a user publishes a post, should the system immediately push that post into every follower's pre-computed feed (fan-out on write), or should it wait until each follower requests their feed and pull relevant posts at read time (fan-out on read)? Fan-out on write delivers low-latency reads but is expensive for users with millions of followers. Fan-out on read avoids write amplification but shifts the cost to read time, where latency budgets are tight. Most production systems use a hybrid approach — fan-out on write for ordinary users and fan-out on read for celebrity accounts — combined with aggressive caching to keep p99 latency under a few hundred milliseconds.

Beyond fan-out, a production news feed must rank content by relevance rather than simply showing the most recent posts. A ranking service scores each candidate post using signals like engagement history, content type, recency, and the viewer's past interactions. The feed must also handle real-time updates (new posts appearing while the user scrolls), mixed media types (text, images, video, ads, sponsored content), and privacy controls (who can see which posts). Designing a news feed is therefore an exercise in composing ingestion pipelines, storage layers, ranking models, caching tiers, and delivery mechanisms into a system that feels instant and personal to every user.

---

## Problem 1 — Feed Publishing and Fan-Out Strategy

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Design the high-level architecture for publishing a post and distributing it to followers' news feeds. The design should cover how a new post is ingested, how it reaches the feeds of the author's followers, and how a user retrieves their personalized feed. Focus on the core write and read paths for a social platform with 50 million daily active users.

### Scenario

**Context:** A social media startup is building its core news feed feature. When a user publishes a post (text, image, or link), it should appear in the feeds of all their followers. The average user follows 200 accounts, and the median account has 300 followers. The team expects 50 million daily active users generating 10 million new posts per day. The product requires that a new post appears in followers' feeds within 5 seconds of publishing. The team needs to decide on the fan-out strategy, the storage model for feeds, and the retrieval mechanism before building the pipeline.

**Requirements:** Choose between fan-out on write, fan-out on read, or a hybrid approach and justify the decision for the given user base. Design the post ingestion flow from the author's client to persistent storage. Design the feed retrieval flow that returns a user's feed in reverse chronological order with pagination. Explain what data is stored per post and per feed entry. Address what happens when a user follows or unfollows someone.

**Expected Approach:** Use fan-out on write for the majority of users (median 300 followers makes write-time fan-out affordable). Store each user's feed as a list of post references (post IDs with timestamps) in a fast key-value store like Redis. When a post is published, a fan-out service pushes the post reference into each follower's feed list. Feed retrieval is a simple read from the pre-computed list. Follow/unfollow operations update the follower graph but do not retroactively modify existing feeds.

<details>
<summary>Hints</summary>

1. Fan-out on write pre-computes each follower's feed at publish time. For a user with 300 followers, publishing one post means 300 write operations to insert the post reference into 300 feed lists. This is affordable for most users but becomes expensive for accounts with millions of followers.
2. Store feed entries as lightweight references — `(post_id, author_id, timestamp)` — rather than duplicating the full post content. The full post is fetched from the post store at read time, allowing updates (edits, deletes) to propagate without modifying every feed entry.
3. Use a sorted set in Redis (keyed by user ID, scored by timestamp) for each user's feed. Retrieval is a `ZREVRANGE` call that returns the most recent N post references in O(log N + M) time.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use fan-out on write with a Redis-based feed cache and an asynchronous fan-out service that distributes post references to follower feeds.

1. **Post ingestion flow:**
   - The author's client sends a POST request to the API gateway with the post content (text, media URLs, metadata).
   - The API gateway authenticates the request and forwards it to the Post Service.
   - The Post Service persists the post to the Post Store (e.g., a relational database or document store) and returns a `post_id`.
   - The Post Service publishes a `PostCreated` event to a message queue (e.g., Kafka) containing `post_id`, `author_id`, and `timestamp`.

2. **Fan-out service:**
   - The Fan-Out Service consumes `PostCreated` events from the queue.
   - For each event, it queries the Social Graph Service to retrieve the list of the author's followers.
   - For each follower, it inserts a feed entry `(post_id, author_id, timestamp)` into the follower's feed list in Redis using `ZADD feed:{user_id} {timestamp} {post_id}`.
   - The fan-out is performed asynchronously and in parallel (batch followers into chunks of 500, process concurrently).
   - For a user with 300 followers, this produces 300 Redis `ZADD` operations — completing in under 1 second.

3. **Feed retrieval flow:**
   - When a user opens the app, the client requests `GET /feed?page_size=20&before={cursor_timestamp}`.
   - The Feed Service reads the user's feed list from Redis: `ZREVRANGEBYSCORE feed:{user_id} {cursor} -inf LIMIT 0 20`.
   - This returns 20 post IDs. The Feed Service fetches the full post content from the Post Store in a batch query.
   - The response includes post content, author info, and a cursor for the next page.

4. **Feed entry storage schema:**
   ```
   Redis Sorted Set: feed:{user_id}
   Score: timestamp (Unix epoch milliseconds)
   Member: post_id

   Post Store Table: posts
   +------------------+--------------+----------------------------------------------+
   | Column           | Type         | Notes                                        |
   +------------------+--------------+----------------------------------------------+
   | post_id          | UUID         | Primary key                                  |
   | author_id        | VARCHAR(64)  | Who created the post                         |
   | content          | TEXT         | Post body                                    |
   | media_urls       | JSON         | Array of attached media URLs                 |
   | created_at       | TIMESTAMP    | Publication time                             |
   | updated_at       | TIMESTAMP    | Last edit time                               |
   +------------------+--------------+----------------------------------------------+
   ```

5. **Follow/unfollow behavior:**
   - **Follow:** The Social Graph Service records the new edge. Future posts from the followed user will be fanned out to the new follower. Past posts are not retroactively inserted into the feed (the user sees new content going forward).
   - **Unfollow:** The Social Graph Service removes the edge. Future posts are no longer fanned out. Existing entries from the unfollowed user may remain in the feed cache and naturally age out, or a background job can lazily remove them.

6. **Feed list maintenance:**
   - Each user's feed list is capped at 1,000 entries (oldest entries are evicted via `ZREMRANGEBYRANK`). This bounds memory usage per user.
   - If a user scrolls beyond the cached 1,000 entries, the Feed Service falls back to a database query that assembles older feed entries from the Post Store using the follower graph.

</details>

---

## Problem 2 — Feed Ranking and Personalization

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a ranking system that orders a user's news feed by relevance rather than pure reverse chronological order. The system must score candidate posts using multiple signals, return a ranked feed within a strict latency budget, and support experimentation so the team can iterate on ranking models without redeploying the entire feed pipeline. The design should address how ranking integrates with the fan-out and caching layers from Problem 1.

### Scenario

**Context:** The social platform from Problem 1 has grown to 200 million monthly active users. User engagement data shows that a purely chronological feed causes users to miss high-quality posts from close friends because they are buried under a flood of content from high-volume accounts. The product team wants to introduce a relevance-ranked feed that surfaces the most interesting content first. The ranking model should consider signals like: how often the viewer interacts with the author (likes, comments, shares), the post's early engagement metrics (like count, comment count within the first hour), content type preference (the viewer tends to engage more with photos than text posts), and recency. The engineering team needs the ranking step to add no more than 100 ms to the feed retrieval latency (p95). They also want an A/B testing framework so they can compare different ranking formulas on live traffic.

**Requirements:** Design the ranking pipeline — where in the feed retrieval flow does ranking occur, what inputs does the ranker consume, and what does it output? Define the feature set (signals) used for scoring and explain how each signal is computed and stored. Describe how the ranking model is served at request time within the 100 ms latency budget. Design the A/B testing mechanism that allows multiple ranking models to run simultaneously on different user segments. Explain how ranking interacts with the pre-computed feed cache — does the cache store ranked or unranked feeds?

**Expected Approach:** Store unranked candidate posts in the feed cache (from fan-out on write). At read time, retrieve the top N candidates from the cache, enrich them with ranking features, score them with a lightweight model, sort by score, and return the ranked list. Pre-compute and cache ranking features (interaction affinity scores, post engagement counts) so the ranker does not need to compute them on the fly. Use a feature store for real-time feature serving. Route users to different ranking models based on an experiment assignment service.

<details>
<summary>Hints</summary>

1. The feed cache from Problem 1 stores unranked post references. At read time, the Feed Service retrieves more candidates than needed (e.g., fetch 200 candidates to return 20 ranked results). The ranker scores and sorts these 200 candidates, then returns the top 20. This over-fetch-and-rank pattern keeps the cache simple while enabling personalized ordering.
2. Pre-compute an "affinity score" between each user pair based on interaction history (likes, comments, profile views). Store these scores in a feature store (e.g., Redis or a dedicated feature serving system) keyed by `(viewer_id, author_id)`. At ranking time, look up the affinity score for each candidate post's author — this avoids expensive real-time computation.
3. A lightweight scoring function (e.g., a linear model or small gradient-boosted tree) can score 200 candidates in under 10 ms. The total ranking overhead is: feature fetch (~30 ms) + scoring (~10 ms) + sorting (~1 ms) ≈ 41 ms, well within the 100 ms budget.
4. For A/B testing, an experiment assignment service maps each user to an experiment group at login. The Feed Service passes the group ID to the Ranking Service, which selects the corresponding model. Metrics (engagement rate, time spent, etc.) are logged per group for comparison.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Insert a ranking stage between feed cache retrieval and response assembly. Use a feature store for pre-computed signals, a lightweight model for scoring, and an experiment router for A/B testing.

1. **Ranking pipeline integration:**
   ```
   Feed retrieval flow (updated):
   1. Client requests feed (page_size=20).
   2. Feed Service retrieves 200 unranked candidate post IDs from Redis feed cache.
   3. Feed Service sends candidates to Ranking Service.
   4. Ranking Service fetches features from Feature Store for each candidate.
   5. Ranking Service scores and sorts candidates, returns top 20 post IDs.
   6. Feed Service fetches full post content from Post Store.
   7. Feed Service returns ranked feed to client.
   ```

2. **Ranking feature set:**
   | Signal | Description | Source | Update Frequency |
   |--------|-------------|--------|-----------------|
   | Affinity score | How often the viewer interacts with the author | Pre-computed from interaction logs | Hourly batch job |
   | Post engagement | Like count, comment count, share count | Real-time counters in Redis | Real-time increment |
   | Content type preference | Viewer's historical engagement rate by content type (photo, video, text, link) | Pre-computed from engagement logs | Daily batch job |
   | Recency | Time elapsed since post creation | Derived from post timestamp | Computed at request time |
   | Author activity | How frequently the author posts (high-volume accounts may be down-weighted) | Pre-computed from post frequency | Daily batch job |
   | Social proof | Whether the viewer's close friends have engaged with the post | Real-time lookup against engagement index | Real-time |

3. **Feature store design:**
   - **Affinity scores:** Redis hash `affinity:{viewer_id}` with fields `{author_id}: {score}`. Pre-computed hourly by a Spark job that aggregates interaction events (likes, comments, shares, profile views) over a 30-day window.
   - **Post engagement counters:** Redis hash `engagement:{post_id}` with fields `likes`, `comments`, `shares`. Incremented in real time by the Engagement Service.
   - **Content type preferences:** Redis hash `content_pref:{viewer_id}` with fields `photo`, `video`, `text`, `link` storing engagement rates. Updated daily.
   - Batch feature fetch: the Ranking Service issues a Redis pipeline to fetch all features for 200 candidates in a single round trip (~20–30 ms).

4. **Scoring model:**
   - A lightweight gradient-boosted decision tree (e.g., LightGBM) with ~50 features, trained offline on historical engagement data.
   - The model is serialized and loaded into the Ranking Service at startup. Scoring 200 candidates takes ~5–10 ms.
   - Score formula (simplified linear approximation):
     ```
     score = w1 * affinity + w2 * log(1 + likes) + w3 * content_type_pref + w4 * recency_decay + w5 * social_proof
     ```
   - Posts are sorted by descending score. The top 20 are returned.

5. **A/B testing framework:**
   - An Experiment Assignment Service assigns each user to an experiment group (e.g., "control", "model_v2", "model_v3") at login, stored in a cookie or session.
   - The Feed Service includes the experiment group in the ranking request.
   - The Ranking Service maintains multiple model versions in memory and selects the appropriate one based on the group ID.
   - All feed interactions (impressions, clicks, likes, time spent) are logged with the experiment group ID to a data warehouse.
   - A metrics pipeline computes per-group engagement metrics daily, enabling the team to compare model performance and promote the winner.

6. **Latency budget breakdown:**
   | Step | Latency (p95) |
   |------|--------------|
   | Redis feed cache read (200 candidates) | 5 ms |
   | Feature store batch fetch | 25 ms |
   | Model scoring (200 candidates) | 10 ms |
   | Sort and select top 20 | 1 ms |
   | Post Store batch fetch (20 posts) | 30 ms |
   | Serialization and response | 5 ms |
   | **Total** | **76 ms** |

   This leaves ~24 ms of headroom within the 100 ms p95 budget.

7. **Cache interaction:**
   - The feed cache stores unranked candidates. Ranking happens at read time so that the same cached candidates can be scored differently for different users or experiment groups.
   - Ranked results are not cached because ranking is personalized and changes as features update. However, if a user refreshes within a short window (e.g., 30 seconds), the Feed Service can return a short-lived cached ranked response to avoid redundant ranking calls.

</details>

---

## Problem 3 — Hybrid Fan-Out for Celebrity Accounts

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design a hybrid fan-out system that handles the extreme write amplification caused by celebrity accounts — users with millions of followers. The system must deliver posts from celebrity accounts to followers' feeds with the same latency guarantees as ordinary accounts, without overwhelming the fan-out infrastructure. The design must define how the system classifies accounts as "celebrity" versus "ordinary," how the two fan-out paths merge at read time, and how the transition works when an ordinary account crosses the celebrity threshold.

### Scenario

**Context:** The social platform now has 500 million users. Most users have fewer than 1,000 followers, but approximately 50,000 accounts have more than 1 million followers each. The top accounts have 50–100 million followers. The existing fan-out-on-write system from Problem 1 works well for ordinary users, but when a celebrity with 30 million followers publishes a post, the fan-out service must perform 30 million Redis writes. At 10,000 writes per second per fan-out worker, this takes 3,000 seconds (50 minutes) — far exceeding the 5-second delivery target. During this time, the fan-out queue backs up, delaying delivery for all users. The engineering team needs a hybrid approach that avoids this write amplification for celebrity accounts while preserving the low-latency read experience for followers.

**Requirements:** Define the criteria for classifying an account as a "celebrity" account (follower count threshold, posting frequency, or a combination). Design the fan-out-on-read path for celebrity posts — how are celebrity posts retrieved and merged into a follower's feed at read time? Design the merge logic that combines pre-computed feed entries (from fan-out on write for ordinary accounts) with dynamically fetched celebrity posts (from fan-out on read) into a single ranked feed. Analyze the read-time latency impact of merging the two sources. Handle the edge case where an account crosses the celebrity threshold — how does the system transition without duplicating or dropping posts? Address the scenario where a user follows 50 celebrity accounts, each posting 10 times per day.

**Expected Approach:** Set a follower threshold (e.g., 500,000 followers) above which accounts are classified as celebrities. Celebrity posts are not fanned out on write; instead, they are stored in a per-author timeline. At read time, the Feed Service retrieves the user's pre-computed feed (ordinary posts) and also fetches recent posts from each celebrity the user follows, then merges and ranks the combined set. Cache celebrity timelines aggressively since they are read by millions of followers. Use a transition buffer to handle accounts crossing the threshold.

<details>
<summary>Hints</summary>

1. The celebrity threshold should not be a single hard cutoff — use a hysteresis band (e.g., become celebrity at 500K followers, revert to ordinary at 400K) to prevent accounts near the boundary from flipping back and forth, which would cause fan-out inconsistencies.
2. Celebrity timelines (recent posts per author) are highly cacheable because every follower reads the same data. A single Redis sorted set `celebrity_timeline:{author_id}` holding the last 100 posts can serve millions of read requests with minimal cache misses.
3. At read time, the merge step must interleave posts from the pre-computed feed and multiple celebrity timelines by timestamp (or ranking score). If a user follows 50 celebrities, the Feed Service fetches 50 celebrity timelines — but each fetch is a single Redis read, and the timelines are almost always cache-hot, so the total added latency is ~10–20 ms with pipelining.
4. When an account transitions to celebrity status, stop fanning out new posts on write. Posts already in followers' feeds remain there. A brief overlap period where some followers see a post from both the pre-computed feed and the celebrity timeline is resolved by deduplication on the post ID during the merge step.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Classify accounts by follower count with a hysteresis band, skip fan-out on write for celebrity posts, store celebrity posts in per-author timelines, and merge the two feed sources at read time with deduplication.

1. **Celebrity classification:**
   - An account is classified as "celebrity" when its follower count exceeds 500,000.
   - An account reverts to "ordinary" when its follower count drops below 400,000 (hysteresis band prevents oscillation).
   - The Social Graph Service maintains a `is_celebrity` flag on each account, updated asynchronously when follower counts change.
   - A secondary signal — posting frequency — can further refine classification: an account with 500K followers posting once a month causes minimal fan-out cost, so the system may keep it on the write path. An account with 500K followers posting 20 times a day should definitely be on the read path.

2. **Dual-path fan-out:**
   - **Ordinary accounts (fan-out on write):** Same as Problem 1. The Fan-Out Service pushes post references into each follower's feed list in Redis.
   - **Celebrity accounts (fan-out on read):** The Fan-Out Service does not push to follower feeds. Instead, the post is appended to the celebrity's timeline: `ZADD celebrity_timeline:{author_id} {timestamp} {post_id}`. This is a single write operation regardless of follower count.
   - The Fan-Out Service checks the `is_celebrity` flag before deciding which path to use.

3. **Feed retrieval with merge:**
   ```
   function getFeed(userId, pageSize=20, cursor):
       // Step 1: Get pre-computed feed (ordinary posts)
       ordinaryPosts = redis.zrevrangebyscore("feed:{userId}", cursor, "-inf", LIMIT 0, 200)

       // Step 2: Get list of celebrities the user follows
       celebrityIds = socialGraph.getFollowedCelebrities(userId)

       // Step 3: Fetch recent posts from each celebrity timeline
       pipeline = redis.pipeline()
       for each celeb in celebrityIds:
           pipeline.zrevrangebyscore("celebrity_timeline:{celeb}", cursor, "-inf", LIMIT 0, 20)
       celebrityPosts = pipeline.execute()  // single round trip

       // Step 4: Merge and deduplicate
       allCandidates = merge(ordinaryPosts, flatten(celebrityPosts))
       deduplicated = removeDuplicatesByPostId(allCandidates)

       // Step 5: Rank (using ranking service from Problem 2)
       ranked = rankingService.rank(userId, deduplicated)

       // Step 6: Return top pageSize results
       return ranked[:pageSize]
   ```

4. **Latency analysis:**
   | Step | Latency (p95) |
   |------|--------------|
   | Pre-computed feed read | 5 ms |
   | Social graph lookup (celebrity list) | 5 ms |
   | Celebrity timeline batch fetch (50 timelines, pipelined) | 15 ms |
   | Merge and deduplication | 2 ms |
   | Ranking (from Problem 2) | 35 ms |
   | Post content fetch | 30 ms |
   | **Total** | **92 ms** |

   The celebrity timeline fetch adds ~20 ms compared to the ordinary-only path, but remains within the 100 ms budget because celebrity timelines are cache-hot (millions of followers read the same data, so cache hit rate exceeds 99.9%).

5. **Celebrity timeline caching:**
   - Each celebrity's timeline is stored as a Redis sorted set capped at 200 entries.
   - Since every follower of a celebrity reads the same timeline, the cache hit rate is extremely high. A celebrity with 30 million followers generates 30 million reads of the same sorted set — Redis handles this trivially.
   - Timeline entries are lightweight references `(post_id, timestamp)`, consuming ~100 bytes each. 200 entries × 50,000 celebrities = ~1 GB of Redis memory for all celebrity timelines.

6. **Threshold transition handling:**
   - **Ordinary → Celebrity:** When an account crosses 500K followers:
     1. The Social Graph Service sets `is_celebrity = true`.
     2. The Fan-Out Service stops fanning out new posts on write.
     3. A background job creates the `celebrity_timeline:{author_id}` sorted set and backfills it with the author's recent posts (last 200).
     4. Posts already fanned out to follower feeds remain there. The merge step's deduplication (by post ID) prevents duplicates if a post appears in both the pre-computed feed and the celebrity timeline.
   - **Celebrity → Ordinary:** When an account drops below 400K followers:
     1. The Social Graph Service sets `is_celebrity = false`.
     2. The Fan-Out Service resumes fanning out new posts on write.
     3. The celebrity timeline is retained for a grace period (e.g., 7 days) to serve posts published during the celebrity phase, then deleted.

7. **Edge case — user follows 50 celebrities posting 10 times/day:**
   - Each celebrity's timeline has 10 new posts per day. The Feed Service fetches the top 20 from each timeline (covering 2 days of posts).
   - The merge step combines 200 ordinary candidates + up to 1,000 celebrity candidates (50 × 20). Deduplication and ranking reduce this to the top 20.
   - The additional data volume is modest: 1,000 post references × ~100 bytes = 100 KB, processed in memory in under 2 ms.

</details>

---

## Problem 4 — Real-Time Feed Updates and Cache Invalidation

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design the real-time update mechanism for a news feed that pushes new posts, post edits, post deletions, and engagement count updates to users who are actively viewing their feed — without requiring a full page refresh. The design must also address cache invalidation: when a post is edited or deleted, all cached feed entries referencing that post must reflect the change promptly. The system must handle the tension between serving stale-but-fast cached feeds and delivering fresh-but-expensive real-time updates, all while supporting hundreds of millions of concurrent feed sessions.

### Scenario

**Context:** Users have complained that their feed feels "stale" — they open the app, see the same posts they saw 10 minutes ago, and must pull-to-refresh to see new content. The product team wants three real-time behaviors: (1) new posts from followed accounts should appear at the top of the feed within seconds, with a "New posts available" banner the user can tap to load them; (2) if a post the user is currently viewing is edited or deleted by the author, the feed should update in place; (3) engagement counts (likes, comments) on visible posts should update in near real time so users see social proof accumulating. The platform serves 100 million concurrent feed sessions during peak hours. A naive approach of pushing every update to every active session would generate trillions of events per hour. The team needs a design that delivers freshness without overwhelming the infrastructure.

**Requirements:** Design the mechanism for detecting and delivering new posts to active feed sessions (push via WebSocket, server-sent events, or pull-based polling — justify the choice). Design the cache invalidation strategy for post edits and deletions — how does the system ensure that cached feed entries are updated or removed promptly? Design the real-time engagement counter update mechanism for posts currently visible on a user's screen. Analyze the event volume for each update type and describe how the system limits fan-out. Address the consistency model — is it acceptable for different users to see slightly different engagement counts for the same post, and for how long?

**Expected Approach:** Use a lightweight notification channel (WebSocket or SSE) to push a "new posts available" signal to active sessions, but do not push the full post content — let the client pull the updated feed on demand. For post edits and deletions, publish invalidation events to a pub/sub system; feed servers subscribe and update their local caches. For engagement counters, use eventual consistency with client-side optimistic updates and periodic server reconciliation rather than pushing every increment to every viewer.

<details>
<summary>Hints</summary>

1. Pushing full post content to every active session on every new post is prohibitively expensive. Instead, push a lightweight "new content available" signal. The client displays a banner ("3 new posts"), and when the user taps it, the client fetches the updated feed via the normal API. This reduces push volume from "full post × all sessions" to "1 signal × relevant sessions."
2. For cache invalidation on edits/deletes, use a pub/sub channel per post (or per post shard). When a post is edited, the Post Service publishes an invalidation event. Feed cache servers subscribe to events for posts they have cached and update or evict the affected entries. Since most posts are only cached on a few servers, the fan-out is manageable.
3. Engagement counters do not need strong consistency across all viewers. Use a "read-your-writes" model: when a user likes a post, their client optimistically increments the counter locally. The server updates the authoritative counter asynchronously. Other viewers see the updated count within 5–10 seconds via periodic polling or the next feed refresh.
4. To limit WebSocket fan-out for new post signals, the Feed Service only pushes to sessions that follow the post's author. Maintain a reverse index: `author_id → set of active session IDs`. When a post is published, look up the author's active followers and push the signal only to them.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use WebSocket-based lightweight signals for new post notifications, pub/sub-driven cache invalidation for edits and deletes, and eventually consistent engagement counters with client-side optimistic updates.

1. **New post notification (push signal, pull content):**
   - When a user opens the feed, their client establishes a WebSocket connection to a Notification Gateway.
   - The gateway registers the session in a reverse index: for each author the user follows, add the session to `active_sessions:{author_id}` in Redis (with a TTL matching the session heartbeat).
   - When a new post is published, the Fan-Out Service (in addition to writing feed entries) publishes a `NewPostSignal` to a pub/sub topic partitioned by author ID.
   - Notification Gateway instances subscribe to relevant author topics. On receiving a signal, the gateway pushes a lightweight message to each active session: `{ type: "new_posts", count: 1 }`.
   - The client accumulates these signals and displays "N new posts available." When the user taps the banner, the client calls the feed API to fetch the latest posts.
   - **Volume analysis:** 10 million posts/day ÷ 86,400 seconds ≈ 116 posts/second. Average fan-out to active sessions per post ≈ 50 (only followers who are currently viewing their feed). Total: ~5,800 lightweight pushes/second — trivial for a WebSocket cluster.

2. **Cache invalidation for edits and deletes:**
   - When a post is edited or deleted, the Post Service publishes an `InvalidatePost` event to a Kafka topic with the `post_id`.
   - Feed cache servers consume this topic. Each server checks if it has the `post_id` in any cached feed. If so:
     - **Edit:** The server fetches the updated post content and replaces the cached version.
     - **Delete:** The server removes the post reference from all feed lists that contain it.
   - For active sessions currently displaying the affected post, the Notification Gateway pushes an update: `{ type: "post_updated", post_id: "...", action: "edited" | "deleted" }`. The client fetches the updated post or removes it from the UI.
   - **Staleness window:** Cache invalidation propagates within 2–5 seconds. Users who loaded the post before the invalidation may see the old version until they refresh. This is acceptable for a social feed.

3. **Real-time engagement counters:**
   - Engagement counts (likes, comments, shares) are stored in Redis: `engagement:{post_id}` with fields `likes`, `comments`, `shares`.
   - When a user likes a post:
     1. The client optimistically increments the displayed count (+1) immediately.
     2. The client sends the like action to the Engagement Service.
     3. The Engagement Service atomically increments the counter in Redis and persists the event to the database.
   - Other users viewing the same post see updated counts via one of two mechanisms:
     - **Periodic polling:** The client polls engagement counts for visible posts every 30 seconds. A single batch request fetches counts for all ~10 visible posts.
     - **Passive refresh:** When the user scrolls and a post re-enters the viewport, the client fetches the latest count.
   - **Why not push every increment:** A viral post receiving 10,000 likes per minute would generate 10,000 push events per viewer. With 1 million viewers, that is 10 billion pushes per minute — infeasible. Polling every 30 seconds for 10 visible posts generates 20 requests per minute per user, which is manageable.

4. **Consistency model:**
   - **New posts:** Eventually consistent within 5 seconds. A follower may not see a new post for up to 5 seconds after publication.
   - **Edits/deletes:** Eventually consistent within 5 seconds. A user may see a deleted post for a few seconds after deletion.
   - **Engagement counts:** Eventually consistent within 30 seconds (polling interval). The user who performed the action sees the update immediately (optimistic local update). Other users see it on the next poll cycle.
   - **Read-your-writes:** Guaranteed for the acting user (likes, comments, edits, deletes). The client applies changes locally before server confirmation.

5. **Session lifecycle and cleanup:**
   - WebSocket sessions have a heartbeat interval of 30 seconds. If a heartbeat is missed, the gateway removes the session from all `active_sessions:{author_id}` sets.
   - When the user backgrounds the app, the client closes the WebSocket. The gateway cleans up the session's subscriptions.
   - On app foreground, the client re-establishes the WebSocket and fetches the latest feed (catching up on any posts missed while backgrounded).

6. **Feed cache architecture:**
   ```
   Layer 1: Redis feed cache (per-user sorted sets of post IDs)
       - Written by Fan-Out Service on post creation
       - Invalidated by cache invalidation consumer on edit/delete
       - Read by Feed Service on feed retrieval

   Layer 2: Post content cache (per-post content in Redis or Memcached)
       - Written on first read from Post Store
       - Invalidated by InvalidatePost events
       - TTL: 1 hour (auto-eviction for unpopular posts)

   Layer 3: Post Store (database, source of truth)
       - Always consistent
       - Queried on cache miss
   ```

</details>

---

## Problem 5 — Content Moderation and Ads Integration in the Feed Pipeline

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design the content moderation and advertisement insertion layers for a news feed system. The moderation layer must filter out policy-violating content (spam, hate speech, misinformation) before it reaches users' feeds, handling both pre-publication screening and post-publication takedowns. The ads layer must insert sponsored posts into the ranked feed at controlled intervals without degrading the organic content experience. The design must address how these two systems integrate with the existing fan-out, ranking, and caching pipeline without introducing unacceptable latency or creating loopholes where violating content bypasses moderation.

### Scenario

**Context:** The social platform has grown to 1 billion monthly active users and is under regulatory pressure to prevent harmful content from spreading through the feed. Simultaneously, the business team needs to monetize the feed by inserting targeted advertisements between organic posts. For moderation, the team wants a multi-stage pipeline: an automated classifier that flags potentially violating content at publish time, a queue for human review of borderline cases, and a mechanism to retroactively remove content that was initially approved but later reported by users. For ads, the business requires that one sponsored post appears for every 5 organic posts in the feed, that ads are relevant to the user (targeted by demographics, interests, and behavior), and that the ad selection does not add more than 20 ms to feed latency. The challenge is integrating both systems into the feed pipeline without creating a bottleneck at publish time or a gap where violating content is briefly visible before moderation catches it.

**Requirements:** Design the pre-publication moderation pipeline — what happens between a user submitting a post and the post entering the fan-out pipeline? Define the automated classification stage (what signals it uses, its accuracy/latency trade-offs, and how borderline cases are handled). Design the post-publication takedown mechanism — when a post is reported or a human reviewer flags it, how is it removed from all feeds that already contain it? Design the ad selection and insertion system — how are ads chosen for a specific user, where in the feed pipeline are they inserted, and how does the system ensure the 1:5 ad-to-organic ratio? Address the interaction between moderation and ads — should ads go through the same moderation pipeline? Analyze the latency impact of both systems on the end-to-end feed retrieval path.

**Expected Approach:** Insert a synchronous moderation gate in the post ingestion path that runs a fast ML classifier (~50 ms). Posts classified as "safe" proceed to fan-out immediately; posts classified as "violating" are blocked; borderline posts are published with a "pending review" flag and queued for human review. For post-publication takedowns, use the same cache invalidation mechanism from Problem 4. For ads, run the ad selection in parallel with feed ranking — while the Ranking Service scores organic posts, the Ad Service selects targeted ads — then merge ads into the ranked feed at fixed intervals before returning the response.

<details>
<summary>Hints</summary>

1. The moderation classifier must be fast enough to run synchronously in the publish path without noticeably delaying post creation. A lightweight text classifier (e.g., a fine-tuned BERT model served via TensorFlow Serving) can classify a post in ~30–50 ms. Image and video moderation are slower (~200–500 ms) and should run asynchronously — the post is published immediately with text-only moderation, and media moderation results arrive shortly after.
2. For borderline content (classifier confidence between 0.4 and 0.7), publish the post but restrict its distribution — do not fan it out to followers until a human reviewer approves it. This avoids blocking legitimate content while preventing viral spread of potentially harmful content.
3. Ad selection can run in parallel with feed ranking because the two are independent. The Feed Service fires both requests simultaneously: one to the Ranking Service (score organic posts) and one to the Ad Service (select ads for this user). When both return, the Feed Service interleaves ads into the ranked organic feed at the specified ratio.
4. Ads should go through a separate, pre-approved moderation pipeline. Advertisers submit ads in advance, and the moderation team reviews them before they enter the ad inventory. This means ad serving does not need real-time moderation — only organic posts do.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Add a synchronous moderation gate in the publish path with async escalation for borderline content, reuse cache invalidation for takedowns, and run ad selection in parallel with feed ranking for zero-overhead insertion.

1. **Pre-publication moderation pipeline:**
   ```
   User submits post → API Gateway → Post Service → Moderation Service → decision:
       SAFE (confidence > 0.85)       → persist post, publish to fan-out queue
       VIOLATING (confidence < 0.15)   → reject post, return error to user
       BORDERLINE (0.15–0.85)          → persist post with status="pending_review",
                                         publish to fan-out queue with restricted=true,
                                         enqueue for human review
   ```
   - **Text moderation:** A fine-tuned transformer model classifies the post text for spam, hate speech, violence, and misinformation. Latency: ~40 ms. Runs synchronously in the publish path.
   - **Image/video moderation:** A separate vision model analyzes attached media. Latency: ~300 ms for images, ~2 seconds for video (first-frame + sampling). Runs asynchronously — the post is published with text moderation only, and media moderation results update the post status within seconds.
   - **Restricted distribution:** Posts with `restricted=true` are fanned out only to the author's feed (visible to the author) but not to followers. If the human reviewer approves the post, the restriction is lifted and normal fan-out proceeds. If rejected, the post is deleted.

2. **Human review queue:**
   - Borderline posts are enqueued in a priority queue sorted by estimated reach (follower count × historical engagement rate). High-reach borderline posts are reviewed first.
   - Target review SLA: 95% of borderline posts reviewed within 15 minutes.
   - Reviewers use a moderation dashboard that shows the post content, classifier confidence scores, and similar previously-reviewed posts for reference.
   - Review outcomes: approve (lift restriction, fan out normally), reject (delete post, notify author), or escalate (send to senior reviewer for policy edge cases).

3. **Post-publication takedown:**
   - When a post is reported by users or flagged by a reviewer, the Moderation Service publishes a `TakedownPost` event (equivalent to the `InvalidatePost` event from Problem 4).
   - The cache invalidation pipeline removes the post from all feed caches and pushes a `post_deleted` signal to active sessions displaying the post.
   - The Post Store marks the post as `status=removed` (soft delete for audit trail).
   - **Viral content safeguard:** If a post accumulates more than 1,000 reports within 10 minutes, an automated circuit breaker immediately restricts the post (removes from fan-out, hides from feeds) without waiting for human review. A reviewer then confirms or reverses the action.

4. **Ad selection and insertion:**
   - **Ad Service architecture:** The Ad Service maintains an inventory of pre-approved ad campaigns, each with targeting criteria (demographics, interests, behavioral segments) and a budget/bid.
   - **Ad selection flow:**
     1. The Feed Service sends a request to the Ad Service with the user's profile (anonymized targeting attributes) and the number of ad slots needed (page_size / 5, rounded up).
     2. The Ad Service queries the ad index for campaigns matching the user's targeting attributes.
     3. Matching ads are ranked by a combination of bid price and predicted click-through rate (eCPM = bid × predicted CTR).
     4. The top N ads are returned to the Feed Service.
   - **Parallel execution:** The Feed Service fires the ad selection request simultaneously with the ranking request. Both return within ~30 ms. The Feed Service then interleaves ads into the ranked organic feed.

5. **Ad insertion logic:**
   ```
   function buildFeedResponse(rankedOrganic, selectedAds):
       feed = []
       adIndex = 0
       for i, post in enumerate(rankedOrganic):
           feed.append(post)
           if (i + 1) % 5 == 0 and adIndex < len(selectedAds):
               feed.append(selectedAds[adIndex])
               adIndex += 1
       return feed
   ```
   - Ads are inserted after every 5th organic post, maintaining the 1:5 ratio.
   - Each ad is marked with `is_sponsored: true` so the client renders it with a "Sponsored" label.
   - The Feed Service logs ad impressions (which ads were shown to which users) to the Ad Analytics pipeline for billing and performance tracking.

6. **Ad moderation:**
   - Ads go through a separate pre-approval pipeline. Advertisers submit ad creatives (text, images, video) via a self-service portal.
   - The same ML classifiers used for organic content review the ad creative, followed by a mandatory human review.
   - Only approved ads enter the ad inventory. This means the real-time feed path does not need to moderate ads — they are pre-vetted.
   - If an approved ad is later reported by users, it follows the same takedown flow as organic content.

7. **Latency impact analysis:**
   | Component | Added Latency | Notes |
   |-----------|--------------|-------|
   | Pre-publication text moderation | +40 ms (publish path) | Synchronous, does not affect feed read |
   | Pre-publication media moderation | +0 ms (publish path) | Asynchronous, no publish delay |
   | Ad selection (parallel with ranking) | +0 ms (feed read path) | Runs in parallel, hidden behind ranking latency |
   | Ad insertion into feed | +1 ms (feed read path) | Simple array interleave |
   | **Net impact on feed read latency** | **+1 ms** | |
   | **Net impact on post publish latency** | **+40 ms** | |

   The moderation gate adds 40 ms to post creation (acceptable — users expect a brief delay when publishing). The ad system adds effectively zero latency to feed reads because it runs in parallel with ranking.

</details>
