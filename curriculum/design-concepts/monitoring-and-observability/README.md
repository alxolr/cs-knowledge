# Monitoring & Observability

**Track:** Design Concepts
**Difficulty Tier:** Advanced
**Prerequisites:** None

## Concept Overview

Monitoring and observability are complementary disciplines that enable engineering teams to understand the internal state of production systems from their external outputs. Monitoring is the practice of collecting, aggregating, and alerting on predefined metrics — CPU utilization, request latency, error rates — to detect known failure modes. Observability goes further: it is the property of a system that allows engineers to ask arbitrary questions about its behavior without deploying new code, using the three pillars of metrics, logs, and distributed traces.

At scale, these disciplines become critical infrastructure in their own right. A microservices architecture with hundreds of services generates millions of metric data points per minute, terabytes of log data per day, and trace spans that must be correlated across dozens of service boundaries. The observability platform must ingest this telemetry at high throughput, store it cost-effectively with appropriate retention policies, and expose it through query interfaces that let engineers diagnose incidents in minutes rather than hours. Poorly designed observability systems either drown teams in noise (too many alerts, too much data) or leave blind spots that allow silent failures to cascade.

Designing an observability platform requires balancing competing concerns: instrumentation overhead versus visibility depth, storage cost versus query flexibility, alert sensitivity versus alert fatigue, and real-time dashboards versus historical analysis. This module explores these trade-offs through scenario-based design problems that cover metrics pipelines, log aggregation, distributed tracing, alerting strategies, and unified observability architectures.

---

## Problem 1 — Metrics Collection and Time-Series Storage Pipeline

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Design a metrics collection and storage pipeline for a microservices platform that captures system-level and application-level metrics from 200 services, stores them in a time-series database, and exposes them through dashboards. The design should cover how metrics are emitted by services, how they are collected and aggregated, and how the time-series storage is organized for efficient querying.

### Scenario

**Context:** A mid-size e-commerce company runs 200 microservices on Kubernetes. Each service emits metrics such as request count, latency percentiles (p50, p95, p99), error rate, CPU usage, and memory usage. The operations team currently relies on ad-hoc SSH sessions and scattered log files to diagnose performance issues, which takes hours during incidents. They want a centralized metrics pipeline that collects metrics from all services, stores them for 90 days, and powers real-time dashboards showing service health at a glance. The platform processes approximately 50,000 requests per second across all services.

**Requirements:** Define how services expose metrics (push vs. pull model) and justify the choice. Design the collection layer that gathers metrics from all 200 services. Choose a time-series storage model and explain how data is organized for efficient queries like "show p99 latency for the checkout service over the last 24 hours." Describe how metric cardinality is managed to prevent storage explosion. Outline the dashboard layer that visualizes the collected metrics.

**Expected Approach:** Use a pull-based model where each service exposes a metrics endpoint, and a central collector scrapes endpoints at regular intervals. Store metrics in a time-series database with labels for service name, instance, and metric type. Apply downsampling for older data to manage storage costs. Use a dashboard tool that queries the time-series database and renders graphs.

<details>
<summary>Hints</summary>

1. In a pull-based model, each service exposes an HTTP endpoint (e.g., `/metrics`) that returns current metric values in a standard format. A central collector scrapes all endpoints every 15–30 seconds. This approach decouples services from the collection infrastructure — services do not need to know where to send metrics, and the collector controls the scrape rate.
2. Time-series data is naturally organized by metric name and labels (key-value pairs like `service=checkout`, `instance=pod-3`). A query like "p99 latency for checkout over 24 hours" translates to selecting the time series with labels `{metric=http_request_duration_p99, service=checkout}` and filtering the time range.
3. High-cardinality labels (e.g., user ID, request ID) create an explosion of unique time series and should be avoided in metrics. Use labels with bounded cardinality: service name, HTTP method, status code class (2xx, 4xx, 5xx), endpoint path (grouped, not raw).

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build a pull-based metrics pipeline with a central collector, a time-series database with label-based indexing, downsampling for retention management, and a dashboard frontend.

1. **Service instrumentation (pull model):**
   - Each service includes a metrics library that maintains in-memory counters, gauges, and histograms.
   - The library exposes an HTTP endpoint (`/metrics`) that returns all current metric values in a text-based format.
   - Metric types: counters (monotonically increasing, e.g., total requests), gauges (point-in-time values, e.g., active connections), histograms (distribution of values, e.g., request latency buckets).
   - Each metric includes labels: `service`, `instance`, `method`, `endpoint`, `status_code_class`.

2. **Collection layer:**
   - A central collector discovers service endpoints via Kubernetes service discovery (querying the API server for pod IPs and ports).
   - The collector scrapes each endpoint every 15 seconds, parsing the metric output and appending data points to the time-series database.
   - With 200 services and ~5 instances each, the collector scrapes ~1,000 endpoints per cycle. At 15-second intervals, this is ~67 scrapes per second — easily handled by a single collector instance with a few replicas for redundancy.

3. **Time-series storage model:**
   - Each time series is identified by a metric name and a set of labels: `http_request_duration_p99{service="checkout", method="POST", endpoint="/api/order"}`.
   - Data points are `(timestamp, value)` pairs stored in time-ordered chunks (e.g., 2-hour blocks).
   - An inverted index maps label combinations to time series IDs for fast lookups.
   - **Query example:** `SELECT value FROM metrics WHERE name = 'http_request_duration_p99' AND service = 'checkout' AND timestamp > now() - 24h` scans at most 12 two-hour chunks.

4. **Cardinality management:**
   - Enforce a label policy: no unbounded labels (user IDs, trace IDs, UUIDs). Labels must have fewer than 100 unique values.
   - Group endpoint paths into templates (e.g., `/api/users/{id}` instead of `/api/users/12345`).
   - Monitor the total number of active time series. Alert if it exceeds a threshold (e.g., 1 million series) indicating a cardinality explosion.

5. **Downsampling for retention:**
   - Raw data (15-second resolution) retained for 7 days.
   - 1-minute aggregates (avg, min, max, count) retained for 30 days.
   - 5-minute aggregates retained for 90 days.
   - A background compaction job runs daily, aggregating older raw data into coarser resolutions and deleting the raw points.

6. **Dashboard layer:**
   - A visualization tool queries the time-series database and renders graphs, heatmaps, and tables.
   - Pre-built dashboards: service overview (request rate, error rate, latency per service), infrastructure (CPU, memory, disk per node), and per-service deep dives.
   - Users can create ad-hoc queries and pin them to custom dashboards.

</details>

---

## Problem 2 — Centralized Log Aggregation and Search

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a centralized log aggregation system that collects, indexes, and makes searchable the log output from 200 microservices generating a combined 5 TB of log data per day. The system must support full-text search across all logs, filtering by service, severity level, and time range, and must retain logs for 30 days. The design should address how logs are shipped from services to the central store, how they are parsed and indexed, and how the system handles ingestion spikes during incidents.

### Scenario

**Context:** The e-commerce platform from Problem 1 generates extensive log output. During normal operations, the 200 services produce approximately 5 TB of logs per day (roughly 2 billion log lines). During incidents, log volume can spike 5x as services emit debug-level logs and retry loops generate additional output. Engineers currently search logs by SSH-ing into individual pods and running grep, which is impractical when a request spans 8 services. They need a centralized system where they can search for a specific error message across all services within a time window, filter by severity (ERROR, WARN, INFO, DEBUG), and correlate log lines from different services that handled the same request by using a shared request ID.

**Requirements:** Design the log shipping mechanism — how logs move from service containers to the central store. Define a structured log format that all services must follow. Design the indexing strategy that enables fast full-text search and filtered queries. Explain how the system handles a 5x ingestion spike without dropping logs. Describe the retention and archival strategy for managing 150 TB of monthly log data. Address how engineers correlate logs across services for a single user request.

**Expected Approach:** Use a sidecar or node-level agent that tails log files and ships them to a message queue for buffering. Parse logs into structured JSON with fields for timestamp, service, severity, request ID, and message. Index logs in a search-optimized store with time-based indices. Use the message queue as a buffer to absorb ingestion spikes. Archive older logs to cold storage and delete indices past the retention window.

<details>
<summary>Hints</summary>

1. A node-level log agent (one per Kubernetes node, deployed as a DaemonSet) reads container log files from the node's filesystem and forwards them to a central pipeline. This avoids modifying each service and scales with the number of nodes rather than the number of services.
2. Structured logging (JSON format with consistent fields) is essential for indexing. A log line like `{"timestamp": "2024-01-15T10:23:45Z", "service": "checkout", "severity": "ERROR", "request_id": "abc-123", "message": "payment gateway timeout"}` can be indexed on every field, enabling queries like "all ERROR logs from checkout in the last hour."
3. A message queue between the log agents and the indexing layer acts as a shock absorber. During a 5x spike, the queue buffers excess logs while the indexing layer processes them at its maximum throughput. Logs are delayed but not dropped.
4. Time-based indices (one index per day) simplify retention: deleting logs older than 30 days is as simple as dropping the index for day 31. This is far more efficient than deleting individual documents.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Deploy node-level log agents that ship structured logs through a message queue to a search-optimized indexing cluster with time-based indices, cold-storage archival, and request-ID-based correlation.

1. **Structured log format (enforced across all services):**
   ```json
   {
     "timestamp": "2024-01-15T10:23:45.123Z",
     "service": "checkout",
     "instance": "checkout-pod-7a3b",
     "severity": "ERROR",
     "request_id": "req-abc-123-def",
     "trace_id": "trace-789-xyz",
     "message": "Payment gateway timeout after 5000ms",
     "metadata": {
       "gateway": "stripe",
       "order_id": "ord-456",
       "retry_attempt": 2
     }
   }
   ```
   - `request_id` is propagated across all services handling the same user request, enabling cross-service correlation.
   - `trace_id` links to the distributed tracing system (Problem 3) for deeper investigation.

2. **Log shipping architecture:**
   - A node-level agent runs as a DaemonSet on each Kubernetes node (e.g., 50 nodes).
   - The agent tails container log files from `/var/log/containers/`, parses each line into the structured format, and batches log entries.
   - Batches are sent to a message queue topic (`logs-raw`) every 5 seconds or when the batch reaches 1 MB.
   - **Throughput:** 5 TB/day = ~58 MB/s steady state. With 50 agents, each handles ~1.2 MB/s — well within a single agent's capacity.

3. **Buffering and spike handling:**
   - The message queue topic `logs-raw` is configured with multiple partitions (e.g., 20) for parallel consumption and a retention of 24 hours.
   - During a 5x spike (290 MB/s), the queue absorbs the excess. The indexing layer consumes at its maximum rate (e.g., 150 MB/s), and the backlog drains once the spike subsides.
   - **Backpressure:** If the queue approaches its storage limit, the log agents switch to sampling mode — forwarding all ERROR and WARN logs but sampling only 10% of INFO and DEBUG logs.

4. **Indexing strategy:**
   - A search cluster ingests logs from the message queue and indexes them into time-based indices: one index per day (e.g., `logs-2024-01-15`).
   - Each log entry is indexed on: `timestamp`, `service`, `severity`, `request_id`, `trace_id`, and full-text on `message`.
   - **Query example:** "All ERROR logs from checkout service in the last hour" translates to a filtered query on today's index with `service=checkout AND severity=ERROR AND timestamp > now()-1h`.
   - **Cross-service correlation:** Searching by `request_id=req-abc-123-def` returns all log lines from every service that handled that request, ordered by timestamp.

5. **Retention and archival:**
   - Hot storage (search cluster): last 7 days of indices, fully indexed and queryable in real time.
   - Warm storage: days 8–30, indices are moved to cheaper storage nodes with fewer replicas. Queries are slower but still possible.
   - Cold archive: after 30 days, indices are snapshotted to object storage (compressed). They can be restored for investigation but are not queryable in real time.
   - **Deletion:** Indices older than 30 days in warm storage are deleted. Cold archives are retained for 1 year for compliance.
   - **Storage math:** 5 TB/day × 30 days = 150 TB in warm+hot storage. With compression (~5:1 for JSON logs), the search cluster stores ~30 TB of compressed index data.

6. **Operational considerations:**
   - Monitor ingestion lag (time between log emission and indexing). Alert if lag exceeds 5 minutes.
   - Monitor queue depth. Alert if the backlog exceeds 1 hour of data.
   - Provide a search UI where engineers enter queries, filter by service/severity/time, and view results with syntax highlighting and context (surrounding log lines).

</details>

---

## Problem 3 — Distributed Tracing Across Microservices

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design a distributed tracing system that captures the end-to-end journey of a request as it flows through a chain of microservices, enabling engineers to identify which service in the chain is causing latency or errors. The system must propagate trace context across service boundaries, collect trace spans from all services, assemble them into a complete trace, and provide a query interface for finding and visualizing traces. The design must handle the volume of traces generated by a platform processing 50,000 requests per second without imposing significant overhead on the services themselves.

### Scenario

**Context:** The e-commerce platform processes 50,000 requests per second. A single user request to load a product page touches 8 services: API gateway → product service → inventory service → pricing service → recommendation service → review service → image service → API gateway (response assembly). When the product page is slow (>2 seconds), engineers cannot determine which of the 8 services is the bottleneck. Metrics show that the overall p99 latency is high, but per-service metrics look normal because the slowness is caused by a specific combination of services under certain conditions (e.g., pricing service is slow only when it calls the promotion engine for items with complex discount rules). The team needs distributed tracing to pinpoint the exact service and operation causing the delay.

**Requirements:** Define the trace data model — what a trace, span, and span context contain. Design the context propagation mechanism that carries trace information across HTTP calls, message queue messages, and asynchronous tasks. Describe how spans are collected from services and assembled into complete traces. Address the sampling strategy — tracing every request at 50,000 RPS would generate enormous data, so the system must sample intelligently. Design the query interface that lets engineers find traces by service, operation, duration, or error status. Explain how the system handles clock skew between services running on different machines.

**Expected Approach:** Each request is assigned a unique trace ID at the entry point. As the request flows through services, each service creates a span (with start time, end time, service name, operation name, and status) and propagates the trace ID and parent span ID to downstream calls via HTTP headers. Spans are collected asynchronously by a trace agent and sent to a central trace store. Sampling is applied at the entry point — e.g., trace 1% of requests, plus 100% of requests that result in errors or exceed a latency threshold. The trace store indexes traces by trace ID, service, and duration for querying.

<details>
<summary>Hints</summary>

1. A trace represents the entire journey of a request. It is composed of spans, where each span represents a single operation within a single service (e.g., "pricing-service: calculateDiscount"). Spans form a tree: the root span is created at the entry point, and each downstream call creates a child span linked to its parent via a parent span ID.
2. Context propagation uses HTTP headers (e.g., `traceparent: 00-{trace_id}-{parent_span_id}-{flags}`) for synchronous calls. For asynchronous communication (message queues), the trace context is embedded in the message metadata. This ensures the trace is not broken when a request crosses service boundaries.
3. Head-based sampling (decide at the entry point whether to trace this request) is simple but misses interesting requests that only become slow or erroneous downstream. Tail-based sampling (collect all spans, then decide after the trace is complete whether to keep it) captures all anomalous traces but requires buffering all span data temporarily.
4. Clock skew between machines can cause a child span to appear to start before its parent. Mitigate this by using NTP synchronization across all nodes and by adjusting span timestamps during trace assembly using the parent-child relationship as a constraint (a child cannot start before its parent).

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement a span-based tracing system with HTTP header context propagation, a tail-based sampling strategy for capturing anomalous traces, and a trace store with indexed queries and timeline visualization.

1. **Trace data model:**
   ```
   Trace:
     trace_id: 128-bit globally unique ID (generated at the entry point)
     spans: list of Span objects forming a tree

   Span:
     span_id: 64-bit unique ID (unique within the trace)
     parent_span_id: 64-bit ID of the parent span (null for root span)
     trace_id: 128-bit ID linking this span to its trace
     service_name: string (e.g., "pricing-service")
     operation_name: string (e.g., "calculateDiscount")
     start_time: microsecond-precision timestamp
     end_time: microsecond-precision timestamp
     status: enum (OK, ERROR)
     tags: key-value pairs (e.g., http.method=GET, http.status_code=200)
     logs: timestamped event annotations (e.g., "cache miss at 10:23:45.123")
   ```

2. **Context propagation:**
   - **HTTP calls:** The calling service injects trace context into HTTP headers using the W3C Trace Context format: `traceparent: 00-{trace_id}-{span_id}-{flags}`. The receiving service extracts the header, creates a new child span with the received `span_id` as its `parent_span_id`, and propagates the same `trace_id` to further downstream calls.
   - **Message queues:** The producer embeds trace context in message metadata (e.g., message headers or attributes). The consumer extracts it and continues the trace.
   - **Asynchronous tasks:** When a service enqueues a background job, it attaches the trace context to the job payload. The worker creates a child span linked to the original trace.
   - **Instrumentation:** Services use a tracing library that automatically instruments HTTP clients, HTTP servers, and message queue producers/consumers, minimizing manual code changes.

3. **Span collection pipeline:**
   - Each service's tracing library buffers spans in memory and flushes them to a local trace agent (sidecar or node-level daemon) every 1 second or when the buffer reaches 100 spans.
   - The trace agent batches spans and sends them to a central trace collector via gRPC.
   - The trace collector writes spans to a trace store (a database optimized for trace queries).
   - **Overhead budget:** The tracing library adds <1 ms of latency per span and uses <5 MB of memory per service instance. Network overhead is ~100 bytes per span × 50,000 spans/second = ~5 MB/s across the cluster (before sampling).

4. **Tail-based sampling strategy:**
   - **Phase 1 — Collect all spans:** All spans from all services are sent to the trace collector regardless of sampling decisions. Spans are buffered in the collector for a short window (e.g., 30 seconds) to allow the complete trace to arrive.
   - **Phase 2 — Evaluate the complete trace:** Once all spans for a trace have arrived (detected by a timeout or by receiving the root span's completion), the collector evaluates sampling rules:
     - **Always keep:** Traces with any ERROR span, traces with total duration > 2 seconds (slow requests), traces explicitly flagged by a developer (debug flag in the trace context).
     - **Sample at 1%:** Normal, healthy traces (for baseline analysis).
     - **Drop:** The remaining 99% of normal traces.
   - **Storage savings:** At 50,000 RPS, 1% sampling keeps 500 traces/second. Error and slow traces add perhaps 50–200 traces/second. Total: ~700 traces/second stored, versus 50,000 without sampling.

5. **Trace store and query interface:**
   - The trace store indexes traces by: `trace_id` (primary lookup), `service_name`, `operation_name`, `duration`, `status`, and `timestamp`.
   - **Query examples:**
     - "Find all traces where pricing-service took >500 ms in the last hour" → filter by `service_name=pricing-service AND duration>500ms AND timestamp>now()-1h`.
     - "Show the full trace for trace ID abc-123" → retrieve all spans with `trace_id=abc-123` and render as a timeline.
   - **Visualization:** A trace timeline view shows spans as horizontal bars on a time axis, nested by parent-child relationships. Engineers can see at a glance which service and operation consumed the most time.

6. **Handling clock skew:**
   - All machines run NTP for clock synchronization (typical skew <10 ms).
   - During trace assembly, if a child span's start time is before its parent's start time (impossible in reality), the trace store adjusts the child's start time to equal the parent's start time and logs a clock-skew warning.
   - Span durations (end - start) are computed locally on each machine and are not affected by cross-machine clock skew, so per-span duration measurements remain accurate.

</details>

---

## Problem 4 — Alerting Strategy and Noise Reduction

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design an alerting system for a microservices platform that notifies on-call engineers of genuine production issues while minimizing false positives and alert fatigue. The system must define alert rules based on metrics and logs, route alerts to the appropriate team, suppress duplicate and transient alerts, and escalate unacknowledged alerts. The design should address how to set meaningful thresholds in a dynamic environment where traffic patterns vary by time of day and day of week.

### Scenario

**Context:** The e-commerce platform has deployed the metrics pipeline (Problem 1) and log aggregation system (Problem 2). The operations team has configured 500 alert rules, but the on-call engineer receives an average of 120 alerts per day, of which only 15 require human action. The remaining 105 are false positives caused by: (1) static thresholds that trigger during normal traffic fluctuations (e.g., "error rate > 1%" fires every day at 3 AM when traffic is low and a handful of errors cause a high percentage), (2) transient spikes that resolve within 30 seconds but still fire alerts, (3) cascading alerts where a single root cause (e.g., a database slowdown) triggers alerts in 20 downstream services simultaneously, and (4) alerts for non-critical issues that do not require immediate attention. The team is experiencing alert fatigue — engineers have started ignoring alerts, and a real outage was missed because the alert was buried among noise.

**Requirements:** Design an alert rule definition language that supports both static thresholds and dynamic baselines (anomaly detection). Define how alerts are suppressed for transient spikes (requiring a condition to persist for a minimum duration before firing). Design an alert grouping and deduplication mechanism that collapses cascading alerts from a single root cause into one notification. Define alert severity levels and routing rules that direct critical alerts to on-call engineers and low-severity alerts to a dashboard or ticket system. Design an escalation policy for unacknowledged alerts. Analyze how the proposed design would reduce the 120 daily alerts to a manageable number.

**Expected Approach:** Replace static thresholds with dynamic baselines that account for time-of-day and day-of-week traffic patterns. Require alerts to persist for a minimum duration (e.g., 5 minutes) before firing. Group alerts by a shared label (e.g., the root-cause service or the affected cluster) and send one notification per group. Define severity levels (critical, warning, info) with different routing: critical pages the on-call engineer, warning creates a ticket, info goes to a dashboard. Escalate unacknowledged critical alerts to the secondary on-call after 15 minutes.

<details>
<summary>Hints</summary>

1. Dynamic baselines compare the current metric value against a predicted value based on historical patterns. For example, if the checkout service normally has a 0.5% error rate at 3 AM on weekdays, an alert fires only if the error rate exceeds 2× the predicted value (1.0%), not a fixed 1% threshold. This eliminates false positives from normal low-traffic fluctuations.
2. A "for" duration clause in alert rules (e.g., `error_rate > 2x_baseline FOR 5m`) requires the condition to be true for 5 consecutive minutes before the alert fires. This filters out transient spikes that resolve on their own.
3. Alert grouping by a shared label (e.g., `cluster=us-east-1` or `root_cause=database-primary`) collapses 20 cascading alerts into one notification. The notification includes a summary: "20 services affected — root cause: database-primary latency spike."
4. Severity-based routing ensures that only genuinely urgent issues page the on-call engineer. Warning-level alerts create tickets for next-business-day resolution. Info-level alerts are logged to a dashboard for trend analysis but generate no notification.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build a multi-layer alerting pipeline with dynamic baselines, persistence-based suppression, label-based grouping, severity routing, and time-based escalation.

1. **Alert rule definition language:**
   ```
   alert: CheckoutHighErrorRate
   expr: error_rate{service="checkout"} > 2 * baseline(error_rate{service="checkout"}, "1w")
   for: 5m
   severity: critical
   labels:
     team: checkout-team
     category: availability
   annotations:
     summary: "Checkout error rate is {{ $value }}%, 2x above the weekly baseline"
     runbook: "https://wiki.internal/runbooks/checkout-errors"
   ```
   - `baseline(metric, "1w")` computes the expected value based on the same time-of-day and day-of-week from the past week (e.g., average of the last 4 Tuesdays at 3 PM).
   - `for: 5m` requires the condition to hold for 5 consecutive evaluation cycles (evaluated every 1 minute) before the alert fires.
   - Static thresholds are still available for absolute limits (e.g., `disk_usage > 90%`) where dynamic baselines do not apply.

2. **Dynamic baseline computation:**
   - For each metric used in alert rules, the system computes a rolling baseline using the same hour and day-of-week from the past 4 weeks.
   - Baseline = median of the 4 historical values. Upper bound = baseline × multiplier (e.g., 2×).
   - Baselines are recomputed hourly and cached. This accounts for seasonal patterns (weekday vs. weekend, business hours vs. off-hours).
   - **Example:** Checkout error rate at Tuesday 3 PM over the past 4 weeks: 0.4%, 0.5%, 0.3%, 0.6%. Baseline = 0.45%. Alert threshold = 0.9%. A current value of 0.7% does not fire; 1.2% does.

3. **Transient spike suppression:**
   - The `for` clause ensures that a metric must exceed the threshold for the specified duration before an alert fires.
   - Evaluation happens every 1 minute. A `for: 5m` rule requires 5 consecutive evaluations above the threshold.
   - If the metric dips below the threshold during the window, the counter resets.
   - **Impact:** Eliminates alerts from spikes lasting <5 minutes (e.g., a brief network hiccup causing a 30-second error spike).

4. **Alert grouping and deduplication:**
   - When multiple alerts fire within a short window (e.g., 2 minutes), the system groups them by a configurable key:
     - **By team:** All alerts with `team=checkout-team` are grouped into one notification.
     - **By category:** All alerts with `category=availability` are grouped.
     - **By root cause (inferred):** If a database alert fires and 15 downstream service alerts fire within 2 minutes, the system groups them under the database alert as the likely root cause.
   - The notification includes: the group key, the number of alerts in the group, the highest severity, and a list of affected services.
   - **Deduplication:** If the same alert (same rule, same labels) fires again while a previous instance is still active (unresolved), the system does not send a new notification — it updates the existing alert's timestamp and value.

5. **Severity levels and routing:**
   | Severity | Criteria | Routing | Response Time |
   |----------|----------|---------|---------------|
   | Critical | Service down, data loss risk, >5% error rate | Page on-call engineer (phone + SMS) | 5 minutes |
   | Warning | Degraded performance, elevated errors, capacity approaching limits | Create ticket, send Slack message | Next business day |
   | Info | Anomalous but non-impactful trends | Dashboard entry, weekly report | No immediate action |

   - Routing rules map `severity` + `team` labels to notification channels.
   - Critical alerts outside business hours page the on-call engineer. During business hours, they also post to the team's Slack channel.

6. **Escalation policy:**
   ```
   Level 1: On-call engineer (immediate notification)
   Level 2: Secondary on-call (if Level 1 does not acknowledge within 15 minutes)
   Level 3: Engineering manager (if Level 2 does not acknowledge within 30 minutes)
   Level 4: VP of Engineering (if Level 3 does not acknowledge within 60 minutes)
   ```
   - Acknowledgment means the engineer clicks "ACK" in the alerting tool, indicating they are investigating.
   - Resolved means the alert condition is no longer true for 5 consecutive minutes, or the engineer manually marks it resolved.

7. **Projected noise reduction:**
   - Dynamic baselines eliminate ~40 false positives/day (low-traffic time-of-day triggers).
   - `for` duration suppression eliminates ~30 transient spike alerts/day.
   - Alert grouping collapses ~25 cascading alerts/day into ~5 grouped notifications.
   - Severity routing diverts ~10 non-critical alerts/day from paging to tickets/dashboard.
   - **Result:** From 120 alerts/day to approximately 15–20 actionable notifications/day, matching the actual number of issues requiring human attention.

</details>

---

## Problem 5 — Unified Observability Platform with Correlation Engine

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a unified observability platform that integrates metrics, logs, and distributed traces into a single system with a correlation engine that automatically links related signals across the three pillars. When an engineer investigates an incident, the platform should allow them to start from a metric anomaly, pivot to the relevant traces that contributed to the anomaly, and drill down into the specific log lines from those traces — all without manually copying IDs between tools. The design must address how the three data types are stored, how correlation links are established, and how the query interface enables seamless cross-pillar navigation.

### Scenario

**Context:** The e-commerce platform now has separate systems for metrics (Problem 1), logs (Problem 2), and traces (Problem 3). During a recent incident, the on-call engineer noticed a latency spike on the checkout service dashboard (metrics), then switched to the tracing tool to find slow traces, copied a trace ID, pasted it into the log search tool to find related logs, and finally identified the root cause — a slow database query logged by the inventory service. This investigation took 45 minutes, most of which was spent context-switching between three different tools and manually correlating data. The engineering leadership wants a unified platform where the engineer can click on the latency spike in the metrics dashboard, see the traces that contributed to it, click on a specific trace, and see the logs from each service in that trace — all in one interface. The goal is to reduce mean time to diagnosis from 45 minutes to under 10 minutes.

**Requirements:** Design the data model that links metrics, traces, and logs through shared identifiers (trace ID, service name, time range). Define how exemplars (pointers from a metric data point to a specific trace) are captured and stored. Design the correlation engine that, given a metric anomaly, retrieves the relevant traces and logs. Describe the unified query interface that supports cross-pillar navigation. Address the storage architecture — should all three data types live in one store or remain in separate stores with a federation layer? Analyze the trade-offs. Explain how the platform handles the different retention periods and query patterns of metrics (90 days, aggregate queries), logs (30 days, full-text search), and traces (7 days, trace-ID lookup).

**Expected Approach:** Keep separate specialized stores for each data type (time-series DB for metrics, search cluster for logs, trace store for traces) but add a correlation layer that links them via shared identifiers. Attach exemplars to metric data points — when a histogram bucket is incremented, record the trace ID of the request that caused the increment. The correlation engine uses time windows and shared labels (service name, instance) to find related signals across stores. The unified UI provides a single pane of glass with drill-down capabilities.

<details>
<summary>Hints</summary>

1. Exemplars are the key bridge between metrics and traces. When a service records a latency measurement (e.g., a histogram observation of 1500 ms), it also records the trace ID of that specific request as an exemplar attached to the metric data point. Later, when an engineer sees a latency spike in the dashboard, they can click on the spike and see the trace IDs of the actual requests that caused it.
2. The correlation between traces and logs is established through the `trace_id` field. If all log lines include the `trace_id` of the request being processed, a query for `trace_id=abc-123` in the log store returns all log lines from all services that participated in that trace.
3. A federation layer (a query router that dispatches sub-queries to each specialized store and merges results) avoids the complexity and cost of migrating all data into a single store. Each store remains optimized for its data type, and the federation layer handles cross-pillar queries.
4. Different retention periods mean that an engineer investigating a 3-day-old incident can correlate metrics and logs (both retained) but may not find traces (7-day retention). The platform should clearly indicate when a data type is unavailable for the requested time range.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Build a federated observability platform with specialized stores for each pillar, an exemplar-based metrics-to-traces bridge, trace-ID-based traces-to-logs linking, and a unified query interface with cross-pillar navigation.

1. **Storage architecture (federated model):**
   ```
   ┌─────────────────────────────────────────────────┐
   │              Unified Query Interface              │
   ├─────────────────────────────────────────────────┤
   │              Correlation Engine                   │
   ├───────────────┬───────────────┬─────────────────┤
   │  Time-Series  │  Search       │  Trace          │
   │  DB (Metrics) │  Cluster(Logs)│  Store (Traces) │
   │  90-day ret.  │  30-day ret.  │  7-day ret.     │
   └───────────────┴───────────────┴─────────────────┘
   ```
   - Each store is optimized for its data type: columnar compression for metrics, inverted indices for logs, span-tree indexing for traces.
   - The correlation engine sits above all three stores and routes cross-pillar queries.
   - **Why not a single store:** Metrics require high-throughput numeric aggregation, logs require full-text search, and traces require graph-like span assembly. No single storage engine excels at all three. A federated approach lets each store use the optimal data structure.

2. **Exemplars (metrics → traces bridge):**
   - When a service records a metric observation (e.g., request latency = 1500 ms), the metrics library attaches the current `trace_id` as an exemplar:
     ```
     metric: http_request_duration_seconds
     labels: {service="checkout", method="POST", endpoint="/api/order"}
     value: 1.5
     exemplar: {trace_id="trace-abc-123", timestamp="2024-01-15T10:23:45Z"}
     ```
   - Exemplars are stored alongside the metric data point in the time-series database. Not every data point needs an exemplar — one exemplar per histogram bucket per scrape interval is sufficient.
   - **Usage:** When an engineer sees a p99 latency spike at 10:23 AM, they click on the spike. The dashboard retrieves exemplars from that time window and displays a list of trace IDs with their latencies. The engineer clicks a trace ID to view the full trace.

3. **Trace-to-log correlation:**
   - All services include `trace_id` in every log line (established in Problem 2).
   - When viewing a trace, the platform queries the log store for `trace_id=trace-abc-123` and displays the log lines alongside the trace spans.
   - Logs are grouped by service and ordered by timestamp, showing what each service logged during its span.

4. **Correlation engine query flow (incident investigation):**
   ```
   Step 1: Engineer sees latency spike on checkout dashboard (metrics query)
           → "p99 latency for checkout exceeded 2s from 10:20 to 10:35"

   Step 2: Engineer clicks the spike → correlation engine retrieves exemplars
           → Returns 10 trace IDs with latencies: trace-abc (2.1s), trace-def (2.5s), ...

   Step 3: Engineer clicks trace-abc → trace store returns the full span tree
           → Root span: API gateway (2.1s)
             → checkout-service (1.9s)
               → inventory-service (1.7s)
                 → database query (1.6s)  ← bottleneck identified

   Step 4: Engineer clicks the inventory-service span → log store query
           → trace_id=trace-abc, service=inventory-service
           → Log: "Slow query: SELECT * FROM inventory WHERE sku IN (...) — 1.6s, full table scan"

   Step 5: Root cause identified: missing index on inventory table for SKU lookups.
   Total investigation time: ~8 minutes.
   ```

5. **Unified query interface:**
   - **Metrics view:** Dashboards with time-series graphs. Clicking a data point shows exemplars (links to traces).
   - **Trace view:** Trace timeline with spans. Clicking a span shows its logs and the metric values for that service at that time.
   - **Log view:** Searchable log stream. Clicking a `trace_id` in a log line opens the trace view. Clicking a `service` name opens the metrics dashboard for that service.
   - **Cross-pillar search:** A universal search bar accepts queries like `service=checkout AND latency>2s AND severity=ERROR` and returns results from all three stores, merged by time.

6. **Handling different retention periods:**
   - The UI displays a retention indicator for each pillar:
     ```
     Investigating: January 10, 2024 (5 days ago)
     ✅ Metrics: available (90-day retention)
     ✅ Logs: available (30-day retention)
     ✅ Traces: available (7-day retention)
     ```
   - For investigations beyond 7 days, traces are unavailable. The platform falls back to metrics + logs correlation using `service`, `timestamp`, and `request_id` (if present in logs) instead of `trace_id`.
   - For investigations beyond 30 days, only metrics are available. The platform shows metric graphs and suggests that the engineer check cold-storage log archives if deeper investigation is needed.

7. **Performance considerations:**
   - Cross-pillar queries are parallelized: the correlation engine sends sub-queries to all three stores simultaneously and merges results as they arrive.
   - Exemplar lookups are fast (indexed by metric name + time range in the time-series DB).
   - Trace-to-log correlation is fast (indexed by `trace_id` in the search cluster).
   - The correlation engine caches recent cross-pillar query results (e.g., "exemplars for checkout latency spike at 10:23") for 5 minutes to avoid redundant queries when multiple engineers investigate the same incident.

</details>
