# Video Streaming Design

**Track:** Design Concepts
**Difficulty Tier:** Advanced
**Prerequisites:** [Load Balancing](../load-balancing/README.md), [Caching Strategies](../caching-strategies/README.md), [Microservices Architecture](../microservices-architecture/README.md), [Distributed File Storage Design](../distributed-file-storage-design/README.md)

## Concept Overview

Video streaming is one of the most bandwidth-intensive applications on the internet, accounting for the majority of global downstream traffic. Services like YouTube, Netflix, and Twitch must ingest, process, store, and deliver video content to millions of concurrent viewers across diverse devices and network conditions. Designing a video streaming platform requires orchestrating a complex pipeline that spans upload ingestion, transcoding, content storage, content delivery, and adaptive playback.

At the ingestion layer, raw video files — often many gigabytes in size — must be received reliably, validated, and queued for processing. The transcoding pipeline converts each source video into multiple renditions (resolutions and bitrates) so that clients on fast connections can enjoy high-definition playback while clients on slower networks receive a lower-quality stream without buffering. Encoding formats such as H.264, H.265/HEVC, VP9, and AV1 each offer different trade-offs between compression efficiency, decode complexity, and device compatibility.

Storage must handle an ever-growing library of video segments efficiently. Rather than storing each rendition as a single monolithic file, modern systems split videos into short segments (typically 2–10 seconds) that can be fetched independently. This segmentation is the foundation of adaptive bitrate (ABR) streaming protocols like HLS and DASH, which allow the player to switch quality levels on a segment-by-segment basis in response to changing network conditions.

Delivery at scale relies on content delivery networks (CDNs) that cache popular segments at edge locations close to viewers, reducing origin server load and minimizing latency. Cache warming, cache eviction policies, and multi-tier caching hierarchies all play a role in keeping the hit rate high. For live streaming, the challenge intensifies: segments must be produced, distributed, and cached within seconds to keep end-to-end latency low.

Beyond the core pipeline, a production video platform must handle metadata management (titles, thumbnails, recommendations), digital rights management (DRM), analytics and quality-of-experience monitoring, and abuse detection. This module explores these challenges through scenario-based design problems that exercise different aspects of building a video streaming system from the ground up.

---

## Problem 1 — High-Level Architecture of a Video Streaming Platform

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Design the high-level architecture for a video streaming platform that supports both video-on-demand (VOD) and user-generated content uploads. The design should identify the major components — upload service, transcoding pipeline, storage layer, delivery layer, and client player — and describe the end-to-end flow from a creator uploading a video to a viewer watching it.

### Scenario

**Context:** A startup is building a video-sharing platform targeting educational content creators. They expect 10,000 video uploads per day, with videos ranging from 5 minutes to 2 hours in length. The platform should support 500,000 daily active viewers, with peak concurrency of 50,000 simultaneous streams. Viewers use a mix of mobile phones on cellular networks, laptops on Wi-Fi, and smart TVs on broadband. The team needs a clear architectural blueprint before implementation begins.

**Requirements:** Identify the major components of the system and their responsibilities. Describe the upload flow (creator uploads a video) and the playback flow (viewer watches a video) step by step. Explain how the system handles different device capabilities and network conditions. State what happens when a component fails (e.g., a transcoding worker crashes mid-job).

**Expected Approach:** Introduce an upload service that receives raw video files, a transcoding pipeline that produces multiple renditions, a distributed storage layer for video segments and metadata, a CDN-backed delivery layer for serving segments to viewers, and a client player that implements adaptive bitrate streaming.

<details>
<summary>Hints</summary>

1. Separate the upload path from the playback path. Uploads are write-heavy and can tolerate higher latency; playback is read-heavy and latency-sensitive. Different components optimize for each workload.
2. Transcoding is CPU-intensive and time-consuming. Use a job queue (e.g., message queue) to decouple the upload service from transcoding workers. This allows the system to scale transcoding capacity independently and retry failed jobs.
3. Adaptive bitrate streaming (ABR) protocols like HLS or DASH split each video into short segments (e.g., 4 seconds) and provide a manifest file listing all available quality levels. The client player fetches segments one at a time, choosing the quality level based on current bandwidth.
4. A CDN caches video segments at edge locations worldwide. Popular videos are served entirely from the CDN, reducing load on the origin storage. Less popular (long-tail) content may require fetching from origin on cache miss.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Design a five-component architecture — upload service, transcoding pipeline, storage layer, delivery layer (CDN), and client player — with clear separation between the upload path and the playback path.

1. **Components:**
   - **Upload Service:** Receives raw video files from creators via chunked HTTP uploads (supporting resume on failure). Validates file format and size limits. Stores the raw file in temporary storage and enqueues a transcoding job.
   - **Transcoding Pipeline:** A pool of transcoding workers pulls jobs from a message queue. Each worker downloads the raw video, encodes it into multiple renditions (e.g., 360p, 720p, 1080p at various bitrates), segments each rendition into 4-second chunks, and uploads the segments to permanent storage. Generates a manifest file (HLS .m3u8 or DASH .mpd) listing all segments and quality levels.
   - **Storage Layer:** A distributed object store (similar to S3 or the system from Distributed File Storage Design) holds video segments, manifest files, thumbnails, and metadata. Organized by video ID for efficient retrieval.
   - **Delivery Layer (CDN):** A global CDN with edge nodes in major regions. When a viewer requests a segment, the CDN serves it from cache if available; otherwise, it fetches from the origin storage and caches it for subsequent requests.
   - **Client Player:** An application (web, mobile, or TV) that fetches the manifest file, measures available bandwidth, and requests segments at the appropriate quality level. Implements ABR logic to switch quality levels smoothly as network conditions change.

2. **Upload flow (creator uploads a 30-minute video):**
   ```
   1. Creator initiates upload via the web interface.
   2. Upload Service receives the file in chunks (resumable upload protocol).
   3. Upload Service validates the file (format, duration, size) and stores it in temporary storage.
   4. Upload Service creates a video record in the metadata database (status: "processing") and enqueues a transcoding job.
   5. A transcoding worker picks up the job, downloads the raw file, and begins encoding.
   6. Worker produces segments for each rendition (e.g., 450 segments × 5 renditions = 2,250 segment files).
   7. Worker uploads segments and manifest to permanent storage.
   8. Worker updates the video record (status: "ready") and deletes the raw file from temporary storage.
   9. Creator receives a notification that the video is ready for viewing.
   ```

3. **Playback flow (viewer watches the video):**
   ```
   1. Viewer opens the video page; the client player requests the manifest file from the CDN.
   2. CDN returns the manifest (cache hit) or fetches it from origin (cache miss).
   3. Player parses the manifest to discover available quality levels and segment URLs.
   4. Player estimates initial bandwidth and selects a starting quality (e.g., 720p).
   5. Player requests the first segment from the CDN.
   6. CDN serves the segment (cache hit) or fetches from origin and caches it.
   7. Player buffers the segment and begins playback.
   8. Player continuously measures download speed and adjusts quality level for subsequent segments (ABR).
   9. Playback continues segment by segment until the video ends or the viewer stops.
   ```

4. **Handling device diversity:**
   - The manifest file lists all available renditions. A mobile phone on 3G might select 360p; a smart TV on fiber might select 1080p.
   - The player's ABR algorithm adapts in real time — if the viewer moves from Wi-Fi to cellular, the player detects reduced bandwidth and switches to a lower quality level to avoid buffering.

5. **Failure handling:**
   - **Transcoding worker crash:** The job remains in the queue (unacknowledged). Another worker picks it up and restarts from the beginning. For very long videos, checkpointing can allow resumption from the last completed segment.
   - **CDN edge node failure:** The CDN automatically routes requests to the next nearest edge node. Viewers may experience a brief increase in latency but no interruption.
   - **Origin storage failure:** If the storage layer is designed with replication (per Distributed File Storage Design), data survives individual node failures. The CDN retries fetches from healthy replicas.

</details>

---

## Problem 2 — Video Transcoding Pipeline Design

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a scalable video transcoding pipeline that converts uploaded videos into multiple renditions (resolutions and bitrates) suitable for adaptive bitrate streaming. The pipeline must handle videos of varying lengths and complexities, prioritize jobs appropriately, and maximize throughput while keeping costs under control. Address how the pipeline scales to handle traffic spikes (e.g., a viral video upload event) and how it recovers from worker failures.

### Scenario

**Context:** The video platform from Problem 1 has grown to 50,000 uploads per day. Videos range from 1-minute clips to 4-hour lectures. The current transcoding system uses a fixed pool of 20 transcoding servers, but during peak hours (evenings and weekends), the queue backs up and creators wait 6+ hours for their videos to be ready. The team also notices that short videos (under 5 minutes) are stuck behind long videos, frustrating creators who expect quick turnaround. Additionally, 5% of transcoding jobs fail due to worker crashes or corrupted source files, requiring manual intervention. The team needs a redesigned pipeline that scales dynamically, prioritizes jobs intelligently, and handles failures gracefully.

**Requirements:** Design the job queue structure, including how jobs are prioritized (e.g., short videos first, premium creators first). Explain how the pipeline scales horizontally — when and how to add or remove transcoding workers. Describe the encoding ladder (which renditions to produce) and justify the choices. Design the failure handling mechanism, including retries, dead-letter queues, and alerting. Address how to handle extremely long videos (4+ hours) that exceed the memory or time limits of a single worker.

**Expected Approach:** Use a priority queue with multiple priority levels based on video duration and creator tier. Implement auto-scaling based on queue depth and worker utilization. Define an encoding ladder with 4–6 renditions covering mobile to 4K. Split long videos into chunks that can be transcoded in parallel and stitched together. Use a dead-letter queue for jobs that fail repeatedly.

<details>
<summary>Hints</summary>

1. A priority queue with multiple levels prevents short videos from being blocked by long ones. A simple approach: assign priority based on estimated transcoding time (shorter = higher priority) and creator tier (premium = higher priority). Workers always pick the highest-priority job available.
2. For auto-scaling, monitor two metrics: queue depth (number of pending jobs) and average worker utilization (CPU usage). Scale up when queue depth exceeds a threshold (e.g., >100 pending jobs) or utilization is >80%. Scale down when queue depth is near zero and utilization is <30%. Use cloud auto-scaling groups with a 5-minute cooldown to avoid thrashing.
3. Splitting a long video into segments for parallel transcoding is called "chunked transcoding." The source video is split at keyframe boundaries (e.g., every 30 seconds), each chunk is transcoded independently by a different worker, and the output chunks are concatenated in order. This reduces wall-clock time proportionally to the number of workers.
4. A dead-letter queue (DLQ) captures jobs that fail after a maximum number of retries (e.g., 3 attempts). Jobs in the DLQ are reviewed manually or by an automated diagnostic system. Common failure causes: corrupted source file, unsupported codec, out-of-memory on the worker.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Design a multi-priority job queue with chunked parallel transcoding, cloud-based auto-scaling, and a robust failure handling pipeline.

1. **Job queue structure:**
   - Use a message broker (e.g., RabbitMQ or SQS) with multiple priority levels:
     - **Priority 0 (highest):** Short videos (<5 min) from premium creators
     - **Priority 1:** Short videos from free-tier creators
     - **Priority 2:** Medium videos (5–30 min) from premium creators
     - **Priority 3:** Medium videos from free-tier creators
     - **Priority 4 (lowest):** Long videos (>30 min) from any creator
   - Workers always dequeue from the highest non-empty priority level.
   - Each job message contains: video ID, source file URL, creator tier, estimated duration, retry count, and timestamp.

2. **Encoding ladder (renditions to produce):**
   | Rendition | Resolution | Video Bitrate | Audio Bitrate | Target Device |
   |-----------|-----------|---------------|---------------|---------------|
   | 1 | 360p | 400 kbps | 64 kbps | Mobile on 3G |
   | 2 | 480p | 800 kbps | 96 kbps | Mobile on LTE |
   | 3 | 720p | 2.5 Mbps | 128 kbps | Laptop/tablet |
   | 4 | 1080p | 5 Mbps | 192 kbps | Desktop/TV |
   | 5 | 1440p | 10 Mbps | 192 kbps | High-end desktop |
   - Codec: H.264 for broad compatibility; optionally produce H.265 or AV1 variants for newer devices (30–50% smaller files at equivalent quality).
   - Each rendition is segmented into 4-second chunks for ABR streaming.

3. **Chunked parallel transcoding for long videos:**
   ```
   1. Coordinator receives a job for a 4-hour video.
   2. Coordinator analyzes the source file to identify keyframe positions.
   3. Coordinator splits the video into N chunks at keyframe boundaries (e.g., 480 chunks of ~30 seconds each).
   4. Coordinator enqueues N sub-jobs, one per chunk, all referencing the parent job ID.
   5. Workers pick up sub-jobs and transcode each chunk independently into all renditions.
   6. As sub-jobs complete, results are uploaded to storage with sequential naming.
   7. When all N sub-jobs are complete, a stitching job concatenates the chunks for each rendition and generates the final manifest file.
   8. Parent job is marked complete.
   ```
   - A 4-hour video split into 480 chunks, processed by 20 workers, completes in roughly 1/20th the time of sequential transcoding.

4. **Auto-scaling:**
   - **Scale-up trigger:** Queue depth > 100 pending jobs OR average worker CPU > 80% for 5 minutes.
   - **Scale-up action:** Add workers in increments of 10 (using cloud spot/preemptible instances for cost savings).
   - **Scale-down trigger:** Queue depth < 10 AND average worker CPU < 30% for 15 minutes.
   - **Scale-down action:** Drain workers gracefully (finish current job, then terminate). Remove workers in increments of 5.
   - **Ceiling:** Maximum 200 workers to cap costs. If queue depth still grows, alert the operations team.

5. **Failure handling:**
   ```
   Job fails on worker:
   1. Worker reports failure reason (OOM, crash, corrupt file, timeout).
   2. Job is re-enqueued with retry_count += 1.
   3. If retry_count < 3: job goes back to the priority queue.
   4. If retry_count >= 3: job moves to the dead-letter queue (DLQ).
   5. DLQ processor analyzes failure reasons:
      - Corrupt source file → notify creator to re-upload.
      - OOM → retry on a high-memory worker instance.
      - Unknown crash → alert engineering team.
   ```
   - For chunked transcoding, only the failed chunk is retried, not the entire video.
   - Workers send heartbeats every 30 seconds. If a worker misses 3 heartbeats, its in-progress job is re-enqueued (the worker is assumed crashed).

6. **Cost optimization:**
   - Use spot/preemptible instances for transcoding workers (60–80% cost savings). Spot interruptions are handled by the retry mechanism.
   - Transcode popular renditions (720p, 1080p) immediately; defer less common renditions (360p, 1440p) until first requested ("lazy transcoding").
   - Cache transcoding results — if a creator re-uploads the same file (same content hash), skip transcoding and reuse existing segments.

</details>

---

## Problem 3 — Adaptive Bitrate Streaming and Client-Side Quality Selection

**Difficulty:** Hard
**Estimated Time:** 50 minutes

### Problem Statement

Design the adaptive bitrate (ABR) streaming mechanism for a video platform, covering both the server-side manifest generation and the client-side quality selection algorithm. The system must allow viewers to experience smooth playback with minimal buffering, adapt quickly to changing network conditions, and avoid unnecessary quality oscillations that degrade the viewing experience. Address how the system handles the cold-start problem (selecting initial quality before any bandwidth measurement) and how it behaves during network congestion.

### Scenario

**Context:** The video platform serves 2 million concurrent viewers during peak hours. Viewer complaints fall into three categories: (1) frequent buffering on mobile networks, especially during commutes when signal strength fluctuates, (2) quality oscillation — the video rapidly switches between 720p and 360p every few seconds, creating a jarring experience, and (3) slow start — when a viewer clicks play, the video takes 5–8 seconds to begin because the player starts at 1080p and must buffer a large initial segment before playback can begin. The team needs an ABR algorithm that balances video quality, startup latency, and playback smoothness.

**Requirements:** Design the manifest file format that describes available renditions and segment URLs. Design the client-side ABR algorithm that selects the quality level for each segment based on measured bandwidth, buffer level, and playback history. Address the cold-start problem — how the player selects the initial quality before any bandwidth data is available. Define how the algorithm avoids rapid quality oscillation. Explain how the system handles sudden bandwidth drops (e.g., entering a tunnel on a train). Describe how the server can assist the client in quality selection (e.g., server-side bandwidth estimation or quality hints).

**Expected Approach:** Use a buffer-based ABR algorithm that considers both estimated bandwidth and current buffer occupancy. Start playback at a conservative quality level (e.g., 480p) to minimize startup delay. Use a hysteresis mechanism to prevent rapid oscillation — only switch quality when the bandwidth change is sustained for multiple segments. Implement an emergency drop to the lowest quality when the buffer falls below a critical threshold.

<details>
<summary>Hints</summary>

1. The manifest file (HLS .m3u8 or DASH .mpd) is the contract between server and client. It lists every available rendition with its resolution, bitrate, and codec, plus the URL pattern for each segment. The client parses this manifest once and uses it to construct segment URLs for the chosen quality level.
2. A pure bandwidth-based ABR algorithm estimates available bandwidth by measuring how long each segment takes to download. However, bandwidth estimates are noisy — a single slow segment can cause an unnecessary quality drop. Smoothing the estimate (e.g., exponential moving average over the last 5 segments) reduces noise.
3. Buffer-based ABR adds a second signal: the current buffer level (how many seconds of video are buffered ahead of the playback position). When the buffer is full (e.g., 30 seconds), the algorithm can afford to try a higher quality. When the buffer is low (e.g., <5 seconds), it should drop quality aggressively to avoid stalling.
4. Hysteresis prevents oscillation by requiring a sustained bandwidth change before switching quality. For example, only upgrade quality if the estimated bandwidth exceeds the target bitrate for 3 consecutive segments, and only downgrade if it falls below for 2 consecutive segments.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement a hybrid bandwidth-buffer ABR algorithm with conservative cold start, hysteresis-based switching, and emergency quality drops.

1. **Manifest file design (HLS variant):**
   ```
   #EXTM3U
   #EXT-X-STREAM-INF:BANDWIDTH=464000,RESOLUTION=640x360,CODECS="avc1.42e00a,mp4a.40.2"
   /video/{id}/360p/playlist.m3u8
   #EXT-X-STREAM-INF:BANDWIDTH=896000,RESOLUTION=854x480,CODECS="avc1.42e00a,mp4a.40.2"
   /video/{id}/480p/playlist.m3u8
   #EXT-X-STREAM-INF:BANDWIDTH=2628000,RESOLUTION=1280x720,CODECS="avc1.4d401f,mp4a.40.2"
   /video/{id}/720p/playlist.m3u8
   #EXT-X-STREAM-INF:BANDWIDTH=5192000,RESOLUTION=1920x1080,CODECS="avc1.640028,mp4a.40.2"
   /video/{id}/1080p/playlist.m3u8
   ```
   - Each variant playlist lists the individual segment URLs and durations for that rendition.
   - The master manifest is fetched once; variant playlists are fetched as needed when switching quality.

2. **Cold-start strategy:**
   - Start at a conservative quality level (480p, ~900 kbps) regardless of device type. This ensures the first segment downloads quickly (under 2 seconds on most connections), minimizing time-to-first-frame.
   - Optionally, use a server-side bandwidth hint: the CDN edge node measures the TCP connection's initial throughput during the manifest fetch and includes an `X-Estimated-Bandwidth` header. The player uses this hint to select a better initial quality if bandwidth is clearly high.
   - After the first 2–3 segments, the player has enough bandwidth measurements to switch to the algorithm-selected quality.

3. **Hybrid ABR algorithm:**
   ```
   function selectQuality(segment_index):
       bw_estimate = exponentialMovingAverage(last_5_segment_throughputs)
       buffer_level = player.bufferedSeconds()
       
       # Emergency mode: buffer critically low
       if buffer_level < 4 seconds:
           return LOWEST_QUALITY  # 360p — avoid stall at all costs
       
       # Buffer-based zone mapping
       if buffer_level < 10 seconds:
           # Conservative zone: pick quality well below bandwidth estimate
           target_bitrate = bw_estimate * 0.6
       elif buffer_level < 25 seconds:
           # Steady zone: pick quality matching bandwidth estimate
           target_bitrate = bw_estimate * 0.85
       else:
           # Surplus zone: buffer is healthy, try higher quality
           target_bitrate = bw_estimate * 1.0
       
       candidate = highest rendition with bitrate <= target_bitrate
       
       # Hysteresis: prevent oscillation
       if candidate > current_quality:
           # Upgrade only if bandwidth has supported this level for 3 consecutive segments
           if consecutive_segments_above(candidate.bitrate) >= 3:
               return candidate
           else:
               return current_quality
       elif candidate < current_quality:
           # Downgrade if bandwidth has been below current level for 2 consecutive segments
           if consecutive_segments_below(current_quality.bitrate) >= 2:
               return candidate
           else:
               return current_quality
       else:
           return current_quality
   ```

4. **Handling sudden bandwidth drops (tunnel scenario):**
   - When a segment download stalls or takes significantly longer than expected, the player detects the drop within one segment period (4 seconds).
   - If the buffer is still above 10 seconds, the player continues at the current quality — the buffer absorbs the temporary drop.
   - If the buffer drops below 4 seconds, the emergency mode activates: the player immediately switches to 360p and requests the smallest possible segments.
   - When bandwidth recovers, the hysteresis mechanism gradually ramps quality back up over 3+ segments, avoiding a jarring jump.

5. **Avoiding quality oscillation:**
   - The hysteresis thresholds (3 segments to upgrade, 2 to downgrade) create an asymmetric switching policy. Upgrades are cautious; downgrades are faster (to protect against buffering).
   - Additionally, the algorithm enforces a minimum dwell time: after any quality switch, the player stays at the new level for at least 3 segments before considering another switch. This prevents rapid back-and-forth.

6. **Server-assisted quality selection:**
   - **Bandwidth hints:** The CDN edge includes throughput estimates in HTTP response headers, giving the client a second data point beyond its own measurements.
   - **Quality capping:** For live events with millions of viewers, the server can cap the maximum available quality (e.g., limit to 720p) to reduce origin load. The manifest dynamically excludes higher renditions during peak load.
   - **Per-segment quality recommendations:** An advanced approach where the server analyzes each segment's visual complexity (high-motion scenes need more bitrate than static slides) and includes per-segment bitrate recommendations in the manifest. The client uses these hints to allocate bitrate more efficiently.

</details>

---

## Problem 4 — CDN Architecture and Multi-Tier Caching for Video Delivery

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design the content delivery network (CDN) architecture for a video streaming platform that serves a catalog of 10 million videos to a global audience. The CDN must minimize playback latency, maximize cache hit rates, handle the long-tail distribution of video popularity (a small fraction of videos account for most views), and efficiently serve both popular and rarely-watched content. Address how the CDN handles cache warming for new viral content and cache eviction for the vast long-tail catalog.

### Scenario

**Context:** The video platform has 10 million videos in its catalog, but viewership follows a power-law distribution: the top 1% of videos (100,000 videos) account for 80% of all views, while the remaining 99% (9.9 million videos) are watched infrequently. The platform operates 50 CDN edge locations worldwide. Each edge location has 20 TB of cache storage, for a total of 1 PB of edge cache. However, the full video catalog (all renditions, all segments) occupies 50 PB of origin storage — only 2% of the catalog can fit in the edge cache at any time. The team observes that cache hit rates vary wildly: popular videos have 99% hit rates, but long-tail videos have only 10% hit rates, causing frequent origin fetches that increase latency and origin load. Additionally, when a video goes viral (e.g., shared on social media), the sudden spike in requests overwhelms the edge cache before it can warm up, causing a "thundering herd" of requests to the origin. The team needs a CDN architecture that handles both the steady-state long-tail and the viral spike scenarios.

**Requirements:** Design a multi-tier caching hierarchy (edge, regional, origin shield) and explain the role of each tier. Define the cache eviction policy for each tier, considering the power-law popularity distribution. Design a cache warming strategy for newly uploaded or trending videos. Address the thundering herd problem — when thousands of requests for the same uncached segment arrive simultaneously at an edge node. Explain how the CDN routes requests to the optimal edge location and handles edge node failures. Analyze the trade-offs between cache storage cost and origin bandwidth cost.

**Expected Approach:** Implement a three-tier cache hierarchy: edge nodes (closest to viewers), regional mid-tier caches (aggregate traffic from multiple edges), and an origin shield (single point of contact with origin storage). Use LRU or LFU eviction at the edge, with longer TTLs at the regional tier. Implement request coalescing at each tier to handle thundering herds. Proactively warm edge caches for trending videos based on view velocity.

<details>
<summary>Hints</summary>

1. A multi-tier cache hierarchy reduces origin load by absorbing cache misses at intermediate layers. An edge miss goes to the regional tier (which may have the content cached from another edge's request), and only if the regional tier misses does the request reach the origin. This is especially effective for moderately popular content that isn't hot enough to stay in every edge cache but is requested often enough to stay in the regional cache.
2. The thundering herd problem occurs when many clients request the same uncached content simultaneously. Without mitigation, each request triggers a separate origin fetch, overwhelming the origin. Request coalescing (also called "collapsed forwarding") solves this: the first request triggers an origin fetch, and subsequent requests for the same content wait for that fetch to complete, then all are served from the newly cached copy.
3. Cache warming for viral content can be triggered by view velocity — if a video's request rate increases by 10x in the last hour, proactively push its segments to edge caches in regions where the traffic is spiking. This requires a real-time analytics pipeline that detects trending videos and a push mechanism to populate caches.
4. For the long-tail, consider a two-tier eviction policy: use LRU (Least Recently Used) for the edge tier (optimizes for recency, good for viral spikes) and LFU (Least Frequently Used) for the regional tier (optimizes for overall popularity, good for steady-state traffic).

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Design a three-tier CDN architecture with differentiated caching policies, request coalescing, and proactive cache warming for trending content.

1. **Three-tier cache hierarchy:**
   ```
   Viewer → Edge Node → Regional Mid-Tier → Origin Shield → Origin Storage
   ```
   - **Edge Nodes (50 locations, 20 TB each):** Closest to viewers, optimized for low latency. Cache the hottest content — segments from the top 1% of videos plus recently accessed long-tail segments. High churn rate; content evicted within hours if not re-requested.
   - **Regional Mid-Tier (5 locations, 200 TB each):** Aggregates traffic from 10 edge nodes each. Caches moderately popular content that doesn't fit in every edge but is requested often enough across the region. Lower churn rate; content may persist for days.
   - **Origin Shield (1 location, 500 TB):** Single point of contact with origin storage. All cache misses from regional tiers funnel through the origin shield. Caches the broadest set of content, including long-tail videos that are requested at least once per week. Protects the origin from direct traffic spikes.

2. **Cache eviction policies:**
   | Tier | Policy | Rationale |
   |------|--------|-----------|
   | Edge | LRU with frequency boost | Prioritizes recent requests (captures viral spikes) but gives a bonus to frequently accessed segments (keeps popular content sticky). |
   | Regional | LFU with aging | Prioritizes overall popularity. Aging factor gradually reduces the frequency count over time, allowing stale popular content to be evicted. |
   | Origin Shield | LFU with long TTL | Maximizes hit rate for the widest range of content. Long TTL (7 days) ensures even infrequently accessed content stays cached if space permits. |

3. **Request coalescing (thundering herd mitigation):**
   ```
   Edge node receives request for segment X:
   1. Check local cache → HIT: serve immediately.
   2. MISS: Check if a fetch for segment X is already in progress.
      a. If yes: add this request to the waiting list for segment X.
      b. If no: initiate fetch from regional tier; mark segment X as "fetching."
   3. When fetch completes: cache segment X; serve all waiting requests.
   ```
   - This ensures that even if 1,000 requests for the same segment arrive within 100ms, only one origin fetch occurs.
   - Implemented at every tier: edge coalesces requests to regional; regional coalesces requests to origin shield; origin shield coalesces requests to origin storage.

4. **Cache warming for trending videos:**
   - **Detection:** A real-time analytics pipeline tracks request rates per video. A video is flagged as "trending" if its request rate increases by 5x compared to its 24-hour average.
   - **Warming action:** When a video is flagged as trending:
     1. Identify the regions where traffic is spiking (based on request source IPs).
     2. Push the first 10 segments of each rendition (covering the first 40 seconds of playback) to edge nodes in those regions.
     3. Continue pushing subsequent segments as the video's popularity grows.
   - **Proactive warming for new uploads:** When a creator with a large following uploads a new video, pre-warm edge caches in regions where their followers are concentrated (based on historical view data).

5. **Request routing:**
   - **GeoDNS:** The platform's domain resolves to different IP addresses based on the viewer's geographic location, directing them to the nearest edge node.
   - **Anycast:** Alternatively, all edge nodes share the same IP address via BGP anycast. The network routes each request to the topologically closest edge node.
   - **Failover:** If an edge node fails, health checks detect the failure within 30 seconds. GeoDNS removes the failed node from rotation; anycast automatically routes around it. Viewers experience a brief latency increase as requests go to the next nearest edge.

6. **Cost trade-off analysis:**
   - **More edge cache storage:** Higher upfront cost, but reduces origin bandwidth (which is often the largest ongoing cost). Each additional TB of edge cache that achieves a 90% hit rate saves ~9 TB of origin egress per month.
   - **Fewer tiers:** Simpler architecture, but higher origin load. A two-tier system (edge + origin) works for smaller catalogs but struggles with the long-tail at scale.
   - **Optimal balance:** For a 50 PB catalog with power-law popularity, the three-tier architecture with 1 PB edge + 1 PB regional + 0.5 PB origin shield achieves ~95% overall hit rate, reducing origin egress by 20x compared to no CDN.

7. **Long-tail optimization:**
   - For videos with <10 views per month, caching is inefficient (cache miss rate approaches 100%). Instead, serve these directly from origin storage, which is optimized for high-capacity, low-cost storage.
   - The CDN can detect long-tail requests (based on video popularity metadata) and bypass caching entirely, reducing cache pollution and improving hit rates for popular content.

</details>

---

## Problem 5 — Live Video Streaming Architecture and Low-Latency Delivery

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design the architecture for a live video streaming platform that supports real-time broadcasts with low end-to-end latency. The system must handle live events with millions of concurrent viewers, minimize the delay between the broadcaster's camera and the viewer's screen (targeting under 10 seconds for standard live and under 3 seconds for interactive live), and gracefully handle broadcaster-side and viewer-side network issues. Address how the system differs from video-on-demand in terms of segment generation, caching, and delivery.

### Scenario

**Context:** The video platform is expanding into live streaming for gaming, sports, and interactive events. For gaming streams, viewers expect to interact with the broadcaster via chat, so latency under 5 seconds is critical — longer delays make chat interactions feel disconnected. For sports events, millions of viewers watch simultaneously, and any buffering during a goal or key play causes user complaints. The current VOD architecture uses 4-second segments, but this introduces a minimum 8–12 second latency for live streams (segment generation + CDN propagation + client buffering). The team needs a live streaming architecture that achieves sub-5-second latency for interactive streams while still supporting million-viewer scale for major events.

**Requirements:** Design the live ingest pipeline that receives the broadcaster's stream and produces segments in real time. Explain how segment duration affects latency and scalability trade-offs. Design the live-specific CDN behavior, including how edge nodes fetch segments that are still being written at the origin. Address the "live edge" problem — ensuring all viewers are watching approximately the same point in the stream. Design the client-side player behavior for live streams, including how it handles rebuffering without falling too far behind live. Describe how the system handles broadcaster disconnection and reconnection. Compare and contrast the architecture with the VOD system from Problem 1.

**Expected Approach:** Use shorter segments (1–2 seconds) or chunked transfer encoding to reduce latency. Implement "just-in-time" segment delivery where edge nodes begin serving a segment before it is fully written. Use a live manifest that updates every segment duration, pointing to the latest available segments. The client player maintains a small buffer (2–4 seconds) and prioritizes staying close to the live edge over smooth quality adaptation.

<details>
<summary>Hints</summary>

1. End-to-end live latency is the sum of: encoding latency (time to encode one segment) + upload latency (broadcaster to origin) + CDN propagation latency (origin to edge) + client buffer latency (segments buffered before playback starts). Reducing segment duration from 4 seconds to 1 second cuts the encoding and buffering components significantly, but increases the number of HTTP requests and manifest updates.
2. Chunked transfer encoding (HTTP/1.1) or HTTP/2 server push allows the CDN to begin forwarding a segment to the viewer before the origin has finished writing it. This eliminates the "wait for full segment" delay and can reduce latency by one full segment duration.
3. The "live edge" is the most recent segment available for playback. All viewers should be within 1–2 segments of the live edge. If a viewer falls behind (due to rebuffering), the player should skip forward to the live edge rather than playing through the backlog — viewers want to see what is happening now, not what happened 30 seconds ago.
4. For million-viewer events, the CDN must handle a "flash crowd" where all viewers request the same segment within a 1–2 second window. Request coalescing (from Problem 4) is essential, but the timing is tighter because each segment is only relevant for a few seconds before the next one replaces it.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Design a low-latency live streaming pipeline with short segments, chunked delivery, live-edge-aware client behavior, and CDN optimizations for simultaneous segment requests.

1. **Live ingest pipeline:**
   ```
   Broadcaster → RTMP/SRT Ingest Server → Live Transcoder → Segment Packager → Origin Storage → CDN → Viewers
   ```
   - **Ingest Server:** Receives the broadcaster's stream via RTMP (Real-Time Messaging Protocol) or SRT (Secure Reliable Transport). SRT is preferred for its built-in error correction and lower latency over unreliable networks.
   - **Live Transcoder:** Encodes the incoming stream into multiple renditions in real time. Unlike VOD transcoding, live transcoding must keep up with the real-time stream — if encoding takes longer than the segment duration, the system falls behind.
   - **Segment Packager:** Splits the transcoded stream into segments. For standard live (target <10s latency), use 2-second segments. For interactive live (target <3s latency), use 1-second segments or CMAF (Common Media Application Format) chunks with chunked transfer encoding.
   - **Origin Storage:** Stores segments temporarily (live segments are only needed for a few minutes; older segments are either discarded or archived for VOD replay).

2. **Segment duration vs. latency trade-off:**
   | Segment Duration | Min Latency | Segments/Hour | Manifest Updates/Hour | Best For |
   |-----------------|-------------|---------------|----------------------|----------|
   | 4 seconds | ~12 seconds | 900 | 900 | VOD-like live (sports replays) |
   | 2 seconds | ~6 seconds | 1,800 | 1,800 | Standard live (gaming, talk shows) |
   | 1 second | ~3 seconds | 3,600 | 3,600 | Interactive live (auctions, Q&A) |
   | CMAF chunks (200ms) | ~1 second | 18,000 | 18,000 | Ultra-low latency (bidding, gaming) |
   - Shorter segments increase HTTP request overhead and CDN load but reduce latency. The choice depends on the use case.

3. **Live-specific CDN behavior:**
   - **Just-in-time delivery:** Edge nodes begin serving a segment as soon as the first bytes arrive from the regional tier, using chunked transfer encoding. The viewer's player receives data progressively, reducing latency by up to one segment duration.
   - **Short TTLs:** Live segments have a TTL of 2× the segment duration (e.g., 4 seconds for 2-second segments). After the TTL, the segment is evicted to make room for newer segments.
   - **Manifest polling:** The client polls for an updated manifest every segment duration. The manifest always points to the latest N segments (e.g., the last 5 segments, covering 10 seconds of content). Older segments are removed from the manifest.
   - **Request coalescing:** Critical for live — all viewers request the same segment within a narrow time window. The edge node fetches each new segment once from the regional tier and serves it to all concurrent requesters.

4. **Live edge management and client behavior:**
   ```
   function livePlayerLoop():
       manifest = fetchManifest()
       live_edge_segment = manifest.last_segment
       
       # Start playback 2 segments behind live edge (4 seconds buffer for 2s segments)
       current_segment = live_edge_segment - 2
       
       while streaming:
           downloadAndPlay(current_segment)
           current_segment += 1
           
           # Check if we've fallen behind
           manifest = fetchManifest()
           live_edge = manifest.last_segment
           drift = live_edge - current_segment
           
           if drift > 4:  # More than 8 seconds behind live
               # Skip forward to live edge minus buffer
               current_segment = live_edge - 2
               log("Skipped forward to live edge")
           
           # ABR: prefer lower quality over falling behind
           if buffer_level < 2 seconds:
               switchToLowestQuality()
   ```
   - The player prioritizes staying near the live edge over maintaining high quality. If the buffer runs low, it drops quality immediately rather than risk falling behind.
   - If the player falls more than 4 segments behind the live edge, it skips forward rather than playing through the backlog.

5. **Broadcaster disconnection and reconnection:**
   - **Disconnection detection:** The ingest server detects a dropped connection within 5 seconds (no data received).
   - **Viewer experience:** The player's buffer provides 4–6 seconds of continued playback. After the buffer is exhausted, the player shows a "Broadcaster is reconnecting" overlay.
   - **Reconnection:** When the broadcaster reconnects, the ingest server resumes accepting the stream. The segment packager starts a new segment sequence. The manifest is updated to point to the new segments.
   - **Seamless reconnection:** If the broadcaster reconnects within the buffer window (e.g., <5 seconds), viewers may not notice the interruption. The player transitions smoothly from buffered content to the new live segments.
   - **Extended disconnection (>30 seconds):** The stream is marked as "offline." Viewers see a static screen or are redirected to a replay of the last few minutes.

6. **Comparison with VOD architecture:**
   | Aspect | VOD | Live |
   |--------|-----|------|
   | Segment generation | Offline (minutes to hours after upload) | Real-time (within seconds) |
   | Segment duration | 4–6 seconds (optimized for caching) | 1–2 seconds (optimized for latency) |
   | Manifest | Static (generated once) | Dynamic (updated every segment) |
   | CDN caching | Long TTL (hours/days) | Short TTL (seconds) |
   | Client buffer | Large (30+ seconds) | Small (2–6 seconds) |
   | Quality adaptation | Aggressive (maximize quality) | Conservative (prioritize live edge) |
   | Storage | Permanent | Temporary (archived for replay) |

7. **Scaling for million-viewer events:**
   - Pre-warm edge caches in all regions before the event starts (push placeholder segments or pre-event content).
   - Use dedicated ingest servers for high-profile broadcasts (avoid sharing with smaller streams).
   - Deploy additional edge capacity in regions with high expected viewership.
   - Monitor per-edge request rates in real time; if an edge approaches capacity, redirect overflow traffic to nearby edges via DNS weight adjustment.

</details>
