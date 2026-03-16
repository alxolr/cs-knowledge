# Search System Design

**Track:** Design Concepts
**Difficulty Tier:** Advanced
**Prerequisites:** [Caching Strategies](../caching-strategies/README.md), [Microservices Architecture](../microservices-architecture/README.md)

## Concept Overview

Search is one of the most fundamental features of any large-scale application. Whether it is a web search engine indexing billions of pages, an e-commerce platform helping users find products, or an internal tool surfacing documents across an organization, the underlying challenge is the same: accept a user query, match it against a massive corpus of data, rank the results by relevance, and return them in milliseconds. Building a search system at scale requires combining information retrieval theory — tokenization, inverted indexes, TF-IDF, BM25 — with distributed systems engineering — sharding, replication, caching, and query routing.

A production search system is far more than a single index lookup. The ingestion pipeline must crawl or receive documents, normalize and tokenize their content, and build or update an inverted index that maps terms to document identifiers. At query time, the system must parse the user's input, expand or correct it (autocomplete, spell correction, synonym expansion), fan the query out to multiple index shards, merge and rank the partial results, and return a unified response — all within a latency budget of 100–200 milliseconds. Features like faceted filtering, typeahead suggestions, and personalized ranking add further layers of complexity.

Scaling a search system introduces classic distributed systems trade-offs. The index must be partitioned across machines because no single node can hold billions of documents in memory. Each partition must be replicated for fault tolerance and read throughput. A query coordinator must scatter queries to all relevant partitions, gather partial results, and merge them — a pattern known as scatter-gather. Caching at multiple levels (query result cache, filter cache, segment-level cache) is essential to absorb repeated queries and reduce computational load. Designing a search system is therefore an exercise in composing indexing pipelines, distributed query execution, relevance ranking, and caching into a cohesive, low-latency architecture.

---

## Problem 1 — Full-Text Search Index Architecture

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Design the high-level architecture for a full-text search service that indexes a corpus of 10 million documents and serves keyword search queries with sub-200 ms latency. The design should cover how documents are ingested and indexed, how the inverted index is structured, and how a search query is executed against the index. Focus on the core indexing and retrieval flow for a single-node deployment.

### Scenario

**Context:** A startup is building an internal knowledge base where employees can search across 10 million documents (wiki pages, support tickets, internal memos). The MVP must return relevant results within 200 ms for single-keyword and multi-keyword queries. The team expects the entire index to fit on a single server with 64 GB of RAM. Documents are updated infrequently (a few hundred new documents per day). The team needs to decide on the index data structure, the tokenization strategy, and the query execution flow before writing code.

**Requirements:** Define how documents are processed during ingestion — tokenization, normalization (lowercasing, stemming), and stop-word removal. Describe the structure of the inverted index and how it maps terms to document postings. Explain how a multi-keyword query is executed against the inverted index, including how results from multiple terms are intersected or unioned. Choose a relevance scoring method and justify the choice.

**Expected Approach:** Build an inverted index that maps each normalized term to a sorted list of (document_id, term_frequency) postings. During ingestion, tokenize each document, normalize tokens, and append postings to the index. At query time, look up each query term in the index, intersect the posting lists for AND queries (or union for OR queries), score each candidate document using BM25, and return the top-K results sorted by score.

<details>
<summary>Hints</summary>

1. An inverted index is the core data structure for full-text search. It maps each unique term to a posting list — a sorted list of document IDs (and metadata like term frequency and positions) where that term appears. This allows O(1) term lookup and efficient list intersection.
2. Tokenization splits document text into individual terms. Normalization (lowercasing, stemming — e.g., "running" → "run") ensures that queries match documents regardless of surface form. Stop-word removal (dropping "the", "is", "a") reduces index size without hurting relevance.
3. For multi-keyword AND queries, intersect the posting lists of all query terms. Since posting lists are sorted by document ID, this can be done in O(n + m) time using a merge-intersection algorithm, where n and m are the lengths of the two lists.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build a single-node inverted index with a BM25 scoring model, fed by a tokenization and normalization pipeline.

1. **Document ingestion pipeline:**
   - Receive a document (id, title, body).
   - Tokenize the body into individual words using whitespace and punctuation splitting.
   - Normalize each token: lowercase, apply stemming (e.g., Porter stemmer), remove stop words.
   - For each unique normalized term in the document, append a posting entry `(doc_id, term_frequency, [positions])` to the term's posting list in the inverted index.
   - Store the original document in a document store (key-value by doc_id) for retrieval after search.

2. **Inverted index structure:**
   ```
   Index: HashMap<String, PostingList>

   PostingList: sorted array of Posting entries
   Posting: {
       doc_id: u64,
       term_frequency: u16,
       positions: [u32]    // positions of the term within the document
   }

   Metadata store:
   - doc_count: total number of documents
   - doc_lengths: HashMap<doc_id, u32>  // number of terms in each document
   - avg_doc_length: f64
   ```
   - Posting lists are sorted by `doc_id` to enable efficient intersection.
   - Term frequency and document length are stored for BM25 scoring.

3. **Query execution flow (AND query for "distributed caching"):**
   ```
   1. Tokenize and normalize query: ["distributed", "caching"] → ["distribut", "cach"] (after stemming)
   2. Look up posting lists: postings["distribut"] → [doc_3, doc_17, doc_42, ...], postings["cach"] → [doc_3, doc_9, doc_42, ...]
   3. Intersect posting lists (merge-intersection on sorted doc_ids): [doc_3, doc_42, ...]
   4. For each candidate document, compute BM25 score:
      score(doc, query) = Σ for each term t in query:
          IDF(t) * (tf(t, doc) * (k1 + 1)) / (tf(t, doc) + k1 * (1 - b + b * docLen / avgDocLen))
      where IDF(t) = log((N - df(t) + 0.5) / (df(t) + 0.5) + 1)
   5. Sort candidates by score descending, return top-K (e.g., K=10).
   6. Fetch full document content from the document store for the top-K doc_ids.
   ```

4. **Why BM25 over TF-IDF:**
   - BM25 includes document length normalization (parameter b) so that longer documents are not unfairly boosted simply for containing more term occurrences.
   - BM25 has a term frequency saturation curve (parameter k1) so that repeating a term 100 times does not score 100× higher than appearing once — diminishing returns reflect real relevance better.
   - BM25 is the industry standard for keyword-based relevance ranking and is used by Elasticsearch and Lucene by default.

5. **Handling updates:**
   - New documents are added to a small in-memory buffer index. Periodically (e.g., every 5 minutes), the buffer is merged into the main index.
   - Deleted documents are tracked in a deletion set. During query execution, results matching deleted doc_ids are filtered out. Periodic compaction removes deleted entries from posting lists.

</details>

---

## Problem 2 — Distributed Index Partitioning and Query Routing

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design the partitioning and query routing strategy for a search system that must index 5 billion documents across a cluster of machines. The design must address how the index is split across nodes, how queries are routed to the correct partitions, how partial results are merged into a final ranked list, and how the system handles node failures without losing index data or degrading query latency.

### Scenario

**Context:** The single-node search system from Problem 1 has outgrown its hardware. The corpus has grown to 5 billion documents, requiring approximately 2 TB of index data — far beyond what a single machine can hold in memory. The team must distribute the index across a cluster of 50 machines. Each search query must still return results within 200 ms. The system receives 10,000 queries per second at peak. The team needs to decide between document-based partitioning (each shard holds a subset of documents) and term-based partitioning (each shard holds posting lists for a subset of terms), design the query routing layer, and plan for replica placement to handle node failures.

**Requirements:** Compare document-based and term-based partitioning strategies and justify your choice. Design the query routing layer that receives a user query, fans it out to the appropriate shards, collects partial results, and merges them into a globally ranked top-K list. Explain how replicas are placed to ensure availability when a node fails. Analyze the latency implications of the scatter-gather pattern and describe how to mitigate tail latency from slow shards.

**Expected Approach:** Use document-based partitioning where each shard holds the complete inverted index for a subset of documents. A query coordinator fans the query to all shards (scatter), each shard returns its local top-K results, and the coordinator merges them (gather). Each shard is replicated to 2–3 nodes for fault tolerance. Tail latency is mitigated by sending redundant requests to replicas and using the fastest response.

<details>
<summary>Hints</summary>

1. Document-based partitioning assigns each document to exactly one shard (e.g., by hashing the document ID). Each shard maintains a complete inverted index for its documents. A query must be sent to all shards because any shard might contain relevant documents. This is simple to implement and balances load evenly.
2. Term-based partitioning assigns each term's posting list to a specific shard. A query for "distributed caching" only needs to contact the shards holding "distributed" and "caching." This reduces fan-out but creates hot spots for common terms and complicates multi-term queries that require cross-shard posting list intersection.
3. The scatter-gather pattern introduces tail latency: the coordinator must wait for the slowest shard. If one shard is slow (GC pause, disk I/O), the entire query is delayed. Mitigations include hedged requests (send the query to both the primary and a replica, use whichever responds first) and setting aggressive timeouts with partial result fallback.
4. For merging, each shard returns its local top-K results with scores. The coordinator performs a K-way merge of the sorted result lists to produce the global top-K. This works correctly because BM25 scores are computed per-document and are comparable across shards (assuming consistent IDF statistics).

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use document-based partitioning with a scatter-gather query coordinator, 2-replica placement per shard, and hedged requests for tail latency mitigation.

1. **Partitioning strategy — document-based:**
   - Assign each document to a shard using `shard_id = hash(doc_id) % num_shards`. With 50 shards and 5 billion documents, each shard holds ~100 million documents (~40 GB of index data).
   - Each shard maintains a complete inverted index for its document subset. This means every shard can independently score and rank its local documents for any query.
   - **Why not term-based:** Term-based partitioning reduces query fan-out (only contact shards holding query terms) but creates severe hot spots — terms like "the" or "data" appear in billions of documents, making their shards bottlenecks. Multi-term AND queries require cross-shard intersection of posting lists, adding network round trips. Document-based partitioning avoids these issues at the cost of full fan-out.

2. **Query routing and scatter-gather:**
   ```
   function search(query, topK=10):
       normalizedQuery = tokenizeAndNormalize(query)
       
       // Scatter: send query to all shards (or their replicas)
       futures = []
       for shard in allShards:
           replica = selectReplica(shard)  // round-robin or least-loaded
           futures.append(replica.search(normalizedQuery, topK))
       
       // Gather: collect partial results with timeout
       partialResults = awaitAll(futures, timeout=150ms)
       
       // Merge: K-way merge of sorted result lists
       merged = kWayMerge(partialResults, topK)
       return merged
   ```
   - Each shard executes the query against its local index, computes BM25 scores, and returns its local top-K results.
   - The coordinator performs a K-way merge (using a min-heap of size K) across all shard responses to produce the global top-K.

3. **Replica placement:**
   - Each shard is replicated to 3 nodes (1 primary + 2 replicas) placed in different failure domains (racks or availability zones).
   - Index updates are written to the primary and asynchronously replicated to replicas. A short replication lag (seconds) is acceptable for search — slightly stale results are tolerable.
   - If a node fails, the coordinator routes queries to the surviving replicas. A replacement node is provisioned and catches up from the primary or a surviving replica.

4. **Tail latency mitigation:**
   - **Hedged requests:** For each shard, send the query to the primary and one replica simultaneously. Use whichever response arrives first. This doubles query load but dramatically reduces p99 latency.
   - **Aggressive timeouts:** If a shard does not respond within 150 ms, the coordinator returns results from the shards that did respond, with a flag indicating partial results. Most queries hit all shards, but graceful degradation is preferable to timeout errors.
   - **Shard-level caching:** Each shard caches results for recent queries. Repeated or similar queries are served from cache without hitting the index.

5. **Global IDF consistency:**
   - BM25 requires IDF (inverse document frequency) values that reflect the entire corpus, not just a single shard. If each shard computes IDF from its local document count, scores will be slightly inconsistent across shards.
   - Solution: periodically compute global term statistics (document frequency per term across all shards) and distribute them to all shards. Shards use global IDF for scoring. This is updated asynchronously (e.g., hourly) and the slight staleness is acceptable.

**Why document-based partitioning fits:** It provides uniform load distribution, simple shard assignment, and independent per-shard query execution. The full fan-out cost is manageable with 50 shards and is offset by the simplicity of the architecture and the absence of hot-spot and cross-shard intersection problems that plague term-based partitioning.

</details>

---

## Problem 3 — Relevance Ranking and Query Understanding Pipeline

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design a multi-stage relevance ranking pipeline and query understanding layer for a search system serving an e-commerce platform with 500 million product listings. The pipeline must go beyond basic BM25 keyword matching to incorporate query intent classification, spell correction, synonym expansion, and a machine-learned re-ranking stage that considers user behavior signals (click-through rate, purchase history, recency). The design must maintain sub-200 ms end-to-end latency despite the added complexity.

### Scenario

**Context:** An e-commerce platform's search system currently uses BM25 for ranking, but users frequently complain about irrelevant results. A search for "apple" returns fruit products instead of electronics. Misspelled queries like "wireles headphones" return zero results. Searching for "sneakers" misses products listed as "running shoes." The team wants to build a multi-stage ranking pipeline: (1) query understanding to correct, expand, and classify the query, (2) candidate retrieval using the inverted index, (3) a lightweight first-pass ranker (BM25 + basic features), and (4) a machine-learned re-ranker that uses behavioral signals to produce the final ranking. The system must handle 20,000 queries per second at peak while keeping p95 latency under 200 ms.

**Requirements:** Design the query understanding layer — spell correction, synonym expansion, and intent classification. Explain how the candidate retrieval stage uses the enhanced query to fetch a broad set of candidates. Design the two-stage ranking pipeline: a fast first-pass ranker that narrows candidates to a manageable set, and a slower ML re-ranker that produces the final ordering. Specify what features the ML model uses and how behavioral signals (clicks, purchases) are incorporated. Analyze the latency budget allocation across pipeline stages.

**Expected Approach:** Allocate the 200 ms budget across stages: ~20 ms for query understanding, ~50 ms for candidate retrieval, ~30 ms for first-pass ranking, ~80 ms for ML re-ranking, ~20 ms for network overhead. Use an edit-distance or noisy-channel model for spell correction, a synonym dictionary for expansion, and a lightweight classifier for intent. The first-pass ranker uses BM25 + category boost to select the top 1,000 candidates. The ML re-ranker (a gradient-boosted tree or small neural network) scores the 1,000 candidates using features like BM25 score, click-through rate, conversion rate, price competitiveness, and recency.

<details>
<summary>Hints</summary>

1. Spell correction can use a combination of edit distance (Levenshtein) against a dictionary of known terms and a noisy-channel model that considers both the likelihood of the correction and the prior probability of the corrected term appearing in queries. Pre-computing edit-distance candidates for common terms and caching frequent corrections keeps latency under 10 ms.
2. Synonym expansion maps query terms to related terms using a curated synonym dictionary (e.g., "sneakers" → "running shoes", "trainers") and learned embeddings (terms with similar word2vec vectors). Expanding the query increases recall but can hurt precision — limit expansion to high-confidence synonyms.
3. A two-stage ranking pipeline is standard in production search: the first stage (retrieval + lightweight ranking) is optimized for recall and speed, narrowing billions of documents to ~1,000 candidates. The second stage (ML re-ranker) is optimized for precision and relevance, scoring only the 1,000 candidates with expensive features.
4. Behavioral features like click-through rate (CTR) and conversion rate are powerful ranking signals but must be smoothed to handle new products with no history (cold-start problem). A Bayesian smoothing approach (e.g., Wilson score interval) prevents new products from being unfairly penalized.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build a four-stage pipeline — query understanding, candidate retrieval, first-pass ranking, ML re-ranking — with strict latency budgets per stage and behavioral feature integration in the re-ranker.

1. **Query understanding layer (~20 ms):**
   - **Spell correction (5–10 ms):**
     - Maintain a dictionary of all indexed terms with their corpus frequency.
     - For each query term not found in the dictionary, generate candidates within edit distance 2 using a pre-computed symmetric delete index (SymSpell algorithm).
     - Rank candidates by `P(correction) × P(correction is intended)` using a noisy-channel model. Select the top candidate.
     - Cache the top 100,000 most frequent misspellings and their corrections for O(1) lookup.
   - **Synonym expansion (3–5 ms):**
     - Maintain a curated synonym dictionary (manually reviewed) and a learned synonym model (word2vec nearest neighbors with cosine similarity > 0.8).
     - Expand each query term with up to 2 high-confidence synonyms. Add expanded terms to the query with a reduced boost factor (e.g., 0.5× weight) so original terms are preferred.
     - Example: "sneakers" → query becomes `sneakers^1.0 OR "running shoes"^0.5 OR trainers^0.5`.
   - **Intent classification (5 ms):**
     - A lightweight logistic regression or small neural network classifies the query into categories: product search, brand search, navigational (e.g., "returns policy"), informational.
     - For product searches, the classifier also predicts the most likely product category (e.g., "apple" → Electronics with 0.7 confidence, Food with 0.3). This category signal is passed to the ranking stages as a boost factor.

2. **Candidate retrieval (~50 ms):**
   - Execute the expanded query against the distributed inverted index using the scatter-gather pattern from Problem 2.
   - Retrieve the top 1,000 candidates per shard (with 50 shards, the coordinator merges up to 50,000 partial results into a global top 1,000 using BM25 scores).
   - Apply category filtering if the intent classifier has high confidence (e.g., if "apple" is classified as Electronics with >0.7 confidence, boost documents in the Electronics category during retrieval).

3. **First-pass ranking (~30 ms):**
   - Re-score the 1,000 candidates using BM25 + lightweight features:
     - BM25 text relevance score
     - Category match boost (from intent classification)
     - Document freshness (newer products get a small boost)
     - Exact title match bonus
   - This stage uses a simple linear combination of features — no ML model, just weighted scoring.
   - Output: top 200 candidates passed to the ML re-ranker.

4. **ML re-ranking (~80 ms):**
   - A gradient-boosted decision tree model (e.g., LambdaMART) scores the 200 candidates using rich features:
     - **Text features:** BM25 score, query-title cosine similarity (using pre-computed embeddings), number of query terms matched.
     - **Behavioral features:** Historical CTR for this product on similar queries (smoothed with Bayesian prior), conversion rate, add-to-cart rate, return rate (negative signal).
     - **Product features:** Price competitiveness (percentile within category), average rating, number of reviews, seller reputation score.
     - **Contextual features:** User's browsing history category affinity, device type, geographic region.
   - The model outputs a relevance score for each candidate. The top 10–20 are returned as the final search results.
   - **Cold-start handling:** New products with no behavioral data receive smoothed default scores (e.g., category-average CTR). A small exploration budget (5% of impressions) is allocated to new products to gather behavioral signals.

5. **Latency budget summary:**
   ```
   Query Understanding:    20 ms  (spell + synonyms + intent)
   Candidate Retrieval:    50 ms  (scatter-gather across shards)
   First-Pass Ranking:     30 ms  (BM25 + lightweight features on 1,000 docs)
   ML Re-Ranking:          80 ms  (GBDT on 200 docs with rich features)
   Network / Overhead:     20 ms
   ─────────────────────────────
   Total:                 200 ms  (p95 target)
   ```

6. **Feature pipeline for behavioral signals:**
   - Click and purchase events are logged to a streaming pipeline (e.g., Kafka).
   - A batch job (hourly) aggregates CTR, conversion rate, and other behavioral metrics per (query_category, product_id) pair.
   - Aggregated features are stored in a low-latency feature store (Redis or a dedicated serving layer) and fetched during the ML re-ranking stage.
   - Real-time click signals (within the current session) can be incorporated via a lightweight online feature that tracks "user clicked product X in this session" to boost related products.

**Why a multi-stage pipeline:** Scoring all 500 million products with an expensive ML model is computationally infeasible within 200 ms. The funnel approach — 500M → 1,000 (retrieval) → 200 (first-pass) → 10 (re-ranker) — progressively narrows candidates while increasing scoring sophistication at each stage. Each stage is optimized for its role: retrieval for recall, first-pass for speed, re-ranker for precision.

</details>

---

## Problem 4 — Real-Time Index Updates and Near-Real-Time Search

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design an indexing pipeline that supports near-real-time search — when a document is created or updated, it should become searchable within 5 seconds — for a search system handling 50,000 document writes per second. The design must reconcile the tension between the immutable, segment-based architecture of inverted indexes (which favors batch rebuilds) and the product requirement for near-instant searchability. It must also handle document deletions, schema changes, and index corruption recovery without downtime.

### Scenario

**Context:** A social media platform uses a search system to let users find posts, profiles, and hashtags. Users expect that a new post is searchable almost immediately — if someone posts breaking news, other users searching for that topic should find it within seconds, not minutes. The platform generates 50,000 new or updated documents per second at peak. The current system rebuilds the index in batch every 15 minutes, which means new content has up to a 15-minute search delay. The team wants to move to a near-real-time architecture where new documents are searchable within 5 seconds of creation. However, the inverted index uses an immutable segment-based design (similar to Lucene) where segments are written once and never modified. The team must design a pipeline that bridges the gap between high-throughput writes and the immutable segment model.

**Requirements:** Design the document ingestion pipeline from write event to searchable index entry. Explain how the system uses a combination of an in-memory buffer and immutable segments to achieve near-real-time searchability. Describe the segment lifecycle — creation, merging, and compaction — and how it affects search performance. Handle document updates and deletions in an append-only segment model. Design the recovery mechanism for index corruption or node failure. Ensure that the indexing pipeline does not degrade query performance during high write throughput.

**Expected Approach:** Use a write-ahead log (WAL) for durability, an in-memory buffer (refresh buffer) that is searchable and flushed to a new immutable segment every 1–5 seconds, and a background segment merge process that compacts small segments into larger ones. Deletions are handled via a deletion bitmap — deleted documents are marked but not physically removed until the next segment merge. Recovery replays the WAL to rebuild the in-memory buffer after a crash.

<details>
<summary>Hints</summary>

1. The refresh buffer is a small, mutable in-memory index that holds recently ingested documents. It is searchable — queries check both the refresh buffer and the immutable segments. Every 1–5 seconds, the buffer is "refreshed" (flushed) into a new immutable segment, and a new empty buffer takes its place. This is how Elasticsearch achieves near-real-time search.
2. Immutable segments are never modified after creation. To delete a document, the system writes the document ID to a deletion bitmap associated with the segment. During query execution, results matching the deletion bitmap are filtered out. During segment merges, deleted documents are physically removed.
3. Segment merging is essential to prevent the number of segments from growing unboundedly (which would slow queries, since each query must check every segment). A tiered merge policy groups segments by size and merges small segments together when enough accumulate. This is similar to LSM-tree compaction.
4. A write-ahead log (WAL) records every document write before it enters the refresh buffer. If the node crashes, the WAL is replayed on restart to reconstruct the buffer. Once a segment is flushed to disk, the corresponding WAL entries can be truncated.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement a WAL-backed refresh buffer with periodic flush to immutable segments, a tiered merge policy for compaction, deletion bitmaps for logical deletes, and WAL replay for crash recovery.

1. **Document ingestion pipeline:**
   ```
   Document write event (from Kafka or API)
       → Write to WAL (append-only log on local disk or distributed log)
       → Insert into refresh buffer (in-memory mutable inverted index)
       → Acknowledge write to producer
   
   Every 1 second (refresh interval):
       → Freeze current refresh buffer
       → Convert frozen buffer to a new immutable segment (written to disk)
       → Create a new empty refresh buffer
       → Truncate WAL entries covered by the new segment
   
   Document is searchable as soon as it enters the refresh buffer (~milliseconds after write).
   ```

2. **Refresh buffer design:**
   - The refresh buffer is a small in-memory inverted index (hash map of term → posting list) that supports concurrent reads and writes.
   - Writes append to posting lists. Reads iterate posting lists just like the immutable segments.
   - At refresh time, the buffer is frozen (no more writes accepted), serialized to an immutable segment file on disk, and replaced with a new empty buffer. This operation takes ~50–100 ms for a buffer holding 50,000 documents (1 second of writes at peak).
   - During the freeze-and-flush, a second buffer accepts incoming writes so that ingestion is never blocked.

3. **Immutable segment structure:**
   ```
   Segment file layout:
   +---------------------------+
   | Segment header            |  (version, doc count, creation timestamp)
   +---------------------------+
   | Term dictionary           |  (sorted list of terms with offsets into posting lists)
   +---------------------------+
   | Posting lists             |  (compressed arrays of doc_id, term_freq, positions)
   +---------------------------+
   | Stored fields             |  (original document content for result display)
   +---------------------------+
   | Deletion bitmap           |  (bitset, initially all zeros)
   +---------------------------+
   ```
   - Segments are immutable after creation — no in-place modifications.
   - Posting lists are compressed using variable-byte encoding or PFOR-delta to reduce disk and memory footprint.

4. **Handling updates and deletions:**
   - **Update:** An update is treated as a delete of the old version followed by an insert of the new version. The old document's ID is added to the deletion bitmap of the segment containing it. The new version is inserted into the refresh buffer.
   - **Delete:** The document's ID is added to the deletion bitmap of the segment containing it. During query execution, the query engine checks the deletion bitmap and excludes matching documents from results.
   - **Physical removal:** Deleted documents are physically removed during segment merges. The merge process reads all documents from the source segments, skips those marked in deletion bitmaps, and writes surviving documents to a new merged segment.

5. **Segment merge policy (tiered):**
   - Segments are grouped into tiers by size: small (<10 MB), medium (10–100 MB), large (100 MB–1 GB), very large (>1 GB).
   - When a tier accumulates more than a threshold number of segments (e.g., 10 small segments), they are merged into a single segment in the next tier.
   - Merging runs in background threads and does not block queries or indexing. The old segments are swapped out atomically once the merged segment is ready.
   - **Merge throttling:** During peak write throughput, merge I/O is throttled to prevent it from competing with query serving for disk bandwidth.

6. **Query execution across segments:**
   ```
   function search(query):
       results = []
       // Search the active refresh buffer
       results.append(refreshBuffer.search(query))
       // Search all immutable segments
       for segment in activeSegments:
           segResults = segment.search(query)
           segResults = filterDeleted(segResults, segment.deletionBitmap)
           results.append(segResults)
       // Merge and rank all results
       return mergeAndRank(results, topK=10)
   ```
   - The query engine searches the refresh buffer and all active segments, filters out deleted documents, and merges results. More segments mean more work per query, which is why segment merging is critical.

7. **Crash recovery via WAL replay:**
   - On node startup, the system checks for a WAL file.
   - If the WAL contains entries not covered by any on-disk segment (identified by sequence numbers), those entries are replayed into a new refresh buffer.
   - Once replay is complete, the node resumes normal operation. No data is lost because every write is persisted to the WAL before being acknowledged.
   - Segment files on disk are immutable and self-contained — they do not need recovery. Only the in-memory refresh buffer (which is volatile) needs WAL-based reconstruction.

8. **Isolation between indexing and querying:**
   - Indexing (buffer writes, segment flushes, merges) and querying run on separate thread pools.
   - Segment flushes and merges use sequential I/O, while queries use random I/O (seeking into posting lists). Separating these onto different I/O schedulers or disk volumes prevents write amplification from degrading query latency.

**Why this architecture achieves near-real-time search:** Documents enter the searchable refresh buffer within milliseconds of ingestion. The 1-second refresh interval converts the buffer to a durable segment, ensuring persistence. The WAL guarantees no data loss on crash. The tiered merge policy keeps segment count bounded, maintaining query performance. This is the same architecture used by Elasticsearch and Apache Lucene.

</details>

---

## Problem 5 — Typeahead Search and Query Autocomplete at Scale

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a typeahead search and query autocomplete system for a search engine serving 500 million daily active users. As a user types each character, the system must return the top 5–10 suggested query completions within 50 ms. The suggestions must reflect real-time popularity (trending queries should surface immediately), support personalization (a user who frequently searches for programming topics should see different suggestions than a user who searches for cooking), and handle multiple languages. The system must sustain 100,000 autocomplete requests per second at peak without degrading the main search path.

### Scenario

**Context:** A global search engine wants to add a typeahead feature to its search bar. When a user types "how to b", the system should instantly suggest completions like "how to bake bread", "how to build a website", "how to buy stocks" — ranked by a combination of global popularity and the user's personal search history. The suggestions must update in real time: if a major event causes "how to buy masks" to trend, it should appear in suggestions within minutes. The system serves users in 20+ languages, and suggestions must be language-appropriate. The autocomplete service must be completely independent of the main search infrastructure so that a spike in autocomplete traffic does not affect search query latency. The team needs to design the data structures, the ranking algorithm, the real-time update mechanism, and the serving architecture.

**Requirements:** Design the data structure that stores and retrieves prefix-based completions efficiently. Explain how suggestions are ranked using a combination of global popularity, trending signals, and personalization. Design the real-time update pipeline that incorporates new query data into the suggestion index within minutes. Describe how the system handles multi-language support and prevents offensive or inappropriate suggestions. Specify the serving architecture that achieves 50 ms p99 latency at 100,000 requests per second. Ensure the autocomplete system is isolated from the main search infrastructure.

**Expected Approach:** Use a trie (prefix tree) or a sorted prefix index as the core data structure for fast prefix lookups. Rank suggestions using a weighted combination of historical query frequency, recent trending score (exponentially decayed), and a personalization boost from the user's search history. Update the trie in near-real-time by streaming query logs through a processing pipeline that aggregates counts and pushes updates to the serving layer. Serve from an in-memory data structure replicated across multiple data centers, with a CDN or edge cache for the most common prefixes.

<details>
<summary>Hints</summary>

1. A trie (prefix tree) allows O(L) lookup for a prefix of length L, but storing all completions at each node is memory-intensive. A more practical approach is a trie where each node stores only the top-K completions (pre-computed), so a prefix lookup returns suggestions in O(L) time without traversing the subtree.
2. For ranking, combine three signals: (a) historical frequency — how often this query has been searched over the past 30 days, (b) trending score — a time-decayed count of searches in the last hour, weighted heavily to surface breaking trends, and (c) personalization — a boost for queries matching the user's recent search categories or explicit interests.
3. Real-time updates can use a two-layer architecture: a base trie rebuilt from batch data every few hours, and a small overlay trie updated in real time from streaming query logs. At query time, results from both tries are merged. Periodically, the overlay is folded into the base trie.
4. Multi-language support requires separate tries per language (or a shared trie with language-tagged entries). Language detection can use the user's locale setting or the script of the typed characters (Latin, CJK, Cyrillic, etc.).

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build a two-layer trie architecture (base + real-time overlay) with a multi-signal ranking function, per-language indexes, a streaming update pipeline, and an edge-cached serving layer isolated from main search.

1. **Core data structure — prefix trie with pre-computed top-K:**
   - Each node in the trie corresponds to a character in a prefix. The path from root to a node spells out the prefix.
   - Each node stores a pre-computed list of the top 10 suggestions for that prefix, along with their ranking scores.
   - Lookup for prefix "how to b" traverses 8 nodes and returns the pre-computed top-10 list at the final node — O(L) time, O(1) for the result retrieval.
   - Memory optimization: only store nodes for prefixes that have been queried at least N times (e.g., N=5). Rare prefixes fall back to a slower path that traverses the subtree.
   ```
   Trie node structure:
   {
       children: HashMap<char, TrieNode>,
       topSuggestions: [(query: String, score: f64); 10],
       isTerminal: bool  // marks a complete query string
   }
   ```

2. **Ranking function:**
   ```
   score(query, user) = w1 * historicalFrequency(query)
                      + w2 * trendingScore(query)
                      + w3 * personalizationBoost(query, user)
   
   where:
     historicalFrequency = log(1 + count_30d(query))
     trendingScore = Σ (count in recent time bucket) * decay^(bucket_age)
     personalizationBoost = categoryAffinity(user, query_category) * recencyBoost(user, query)
   ```
   - **Historical frequency:** Log-scaled 30-day query count. Prevents extremely popular queries from dominating all prefixes.
   - **Trending score:** Exponentially decayed count from the last 1–4 hours. A query that spiked in the last 10 minutes gets a high trending score, which decays over hours. This surfaces breaking news and viral topics.
   - **Personalization boost:** If the user frequently searches in a category (e.g., "programming"), queries in that category get a boost. Computed from the user's recent search history (last 30 days), stored in a user profile service.
   - Weights (w1, w2, w3) are tuned via A/B testing. Typical starting values: w1=0.5, w2=0.3, w3=0.2.

3. **Two-layer trie architecture:**
   - **Base trie:** Rebuilt every 4–6 hours from a batch aggregation of the last 30 days of query logs. Contains historical frequency scores for all qualifying prefixes. Stored as a memory-mapped file for fast loading.
   - **Real-time overlay trie:** A small in-memory trie updated every 30–60 seconds from a streaming pipeline (Kafka → aggregator → overlay update). Contains only trending queries from the last few hours.
   - **Query-time merge:** When a user types a prefix, the system looks up both the base trie and the overlay trie, merges the two result lists, re-ranks using the combined scoring function, and returns the top 10.
   - **Periodic fold-in:** Every 4–6 hours, the overlay data is incorporated into a new base trie, and the overlay is reset. This prevents the overlay from growing unboundedly.

4. **Real-time update pipeline:**
   ```
   User searches "how to bake sourdough"
       → Query log event published to Kafka topic
       → Streaming aggregator (Flink/Spark Streaming) groups by query, counts per 1-minute window
       → Aggregated counts pushed to overlay trie service via RPC
       → Overlay trie updates affected nodes' topSuggestions lists
       → Next autocomplete request for "how to b" includes the updated suggestion
   
   End-to-end latency: ~30–60 seconds from search to suggestion appearance.
   ```

5. **Multi-language support:**
   - Maintain separate trie instances per language (English, Spanish, Japanese, etc.).
   - Route autocomplete requests to the appropriate language trie based on: (a) user's locale setting, (b) script detection of typed characters (CJK characters → Chinese/Japanese/Korean trie, Cyrillic → Russian trie).
   - For languages with complex tokenization (Chinese, Japanese), use character-level n-gram prefixes rather than word-level prefixes, since these languages do not use spaces between words.
   - Each language trie is independently sized and replicated based on the user population for that language.

6. **Offensive content filtering:**
   - Maintain a blocklist of offensive queries and patterns (regex-based).
   - During trie construction (both base and overlay), filter out any query matching the blocklist.
   - A human review pipeline periodically audits the top suggestions for each popular prefix and flags inappropriate entries for removal.
   - Rate-of-change monitoring: if a query's trending score spikes unusually fast, it is held for automated content safety review before being added to the overlay trie.

7. **Serving architecture:**
   - The autocomplete service runs on a dedicated cluster, completely isolated from the main search infrastructure. It has its own load balancers, servers, and monitoring.
   - Each serving node holds the complete trie for one or more languages in memory. With compression, the English trie (covering 100 million unique queries) fits in ~8–12 GB of RAM.
   - Requests are routed to the nearest data center via DNS-based geographic routing. Within a data center, a load balancer distributes requests across serving nodes.
   - **Edge caching:** The top 10,000 most common prefixes (e.g., "how", "what", "best") and their suggestions are cached at the CDN edge with a 60-second TTL. This absorbs ~40% of autocomplete traffic without hitting the serving nodes.
   - **Latency breakdown:**
     ```
     CDN/edge cache hit:     5–10 ms  (40% of requests)
     Serving node lookup:   15–25 ms  (trie traversal + merge + ranking)
     Network (user → DC):   10–30 ms  (depends on geography)
     Total p99:             < 50 ms
     ```

8. **Capacity planning:**
   - 100,000 requests/second ÷ 40% cache hit rate = 60,000 requests/second to serving nodes.
   - Each serving node handles ~5,000 requests/second (in-memory trie lookup is fast).
   - 60,000 / 5,000 = 12 serving nodes per data center, with 3 data centers = 36 nodes total.
   - Each node: 16 GB RAM (trie + overhead), 4 CPU cores. Total cluster: ~576 GB RAM, 144 cores — modest infrastructure for the scale.

**Why a two-layer trie with edge caching fits:** The base trie provides stable, high-quality suggestions from historical data. The real-time overlay surfaces trending queries within minutes. Edge caching absorbs the most repetitive traffic. The isolated serving architecture ensures that autocomplete spikes (e.g., during a major event) do not affect main search latency. The pre-computed top-K at each trie node ensures constant-time suggestion retrieval regardless of the corpus size.

</details>
