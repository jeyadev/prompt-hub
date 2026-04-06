# Copilot Prompt: Generate Grafana Migration Dashboards from Prometheus Metrics & Migration KB

---

## CONTEXT

You are an SRE dashboard architect. You have two inputs:

1. **Prometheus metrics payload** — the raw output from the migrate application's `/actuator/prometheus` endpoint. This contains every metric the application emits (counters, gauges, histograms, summaries, info metrics).
2. **Migration Knowledge Base folder** — a set of documents describing the end-to-end IBM CM → Pithos migration flow, including process stages, data transformations, validation steps, error handling, and retry logic.

Your job is to produce two production-grade Grafana dashboard specifications as `.md` files, with exact panel layouts, PromQL queries, and configuration details for every widget.

---

## PHASE 1: METRIC INVENTORY & CLASSIFICATION

Before designing any dashboard, you must first build a complete inventory of every metric from the `/actuator/prometheus` payload. Do NOT skip this phase. Do NOT jump to dashboard design.

### Step 1.1 — Extract and catalog every metric

For each metric found in the Prometheus payload, record:

| Field | Description |
|-------|-------------|
| `metric_name` | Full metric name as emitted (e.g., `migration_documents_processed_total`) |
| `type` | Prometheus type: `counter`, `gauge`, `histogram`, `summary`, `info`, `untyped` |
| `labels` | All label keys present (e.g., `status`, `source_type`, `target_system`, `step`) |
| `label_cardinality` | Approximate number of distinct label value combinations (low/medium/high) |
| `unit` (if inferable) | seconds, bytes, count, ratio, etc. |
| `description` | From the `# HELP` line, or inferred from naming convention |

### Step 1.2 — Classify each metric by domain

Assign each metric to exactly one domain:

- **JVM / Runtime** — heap, GC, threads, classloading, CPU process metrics
- **HTTP / Server** — request counts, latencies, error rates for the app's own endpoints
- **Migration Core** — metrics directly related to migration business logic (document processing, batch progress, record counts)
- **Integration / External Calls** — outbound calls to IBM CM, Pithos, databases, message queues, or any external dependency
- **Spring Boot Actuator / Framework** — info metrics, build info, uptime, standard Spring metrics
- **Custom / Application** — any metric that doesn't fit the above categories

### Step 1.3 — Signal value assessment

For each metric, assign a signal value rating with explicit rationale:

| Rating | Criteria | Action |
|--------|----------|--------|
| **HIGH** | Directly answers: "Is the migration working? How far along are we? What's broken?" — These are metrics an engineer would check first during a migration batch or when paged. | **INCLUDE** in dashboards — these become primary panels |
| **MEDIUM** | Provides supporting context: resource saturation, dependency health, queue depths. Useful for diagnosis but not the first thing you check. | **INCLUDE** as secondary panels or drill-down rows |
| **LOW** | Standard JVM/framework telemetry that exists on every Spring Boot app. Not migration-specific. Useful only for deep debugging, not migration operations. | **EXCLUDE** from migration dashboards. Note exclusion reason. |

**Discard criteria — a metric is LOW if ALL of these are true:**
- It exists identically on every Spring Boot app (not migration-specific)
- It does not change behavior during migration processing (e.g., JVM info, build version)
- An engineer troubleshooting a stuck migration batch would never look at this metric first
- It provides no insight into migration throughput, latency, error rate, or progress

**Promotion criteria — override LOW to MEDIUM if:**
- The metric could explain resource exhaustion during heavy migration batches (e.g., heap pressure, thread pool saturation, connection pool exhaustion)
- The metric correlates with migration throughput (e.g., GC pause time affecting batch processing latency)

### Step 1.4 — Output the full classification table

Produce a complete markdown table with ALL metrics classified. This table becomes the source of truth for dashboard design. Format:

```markdown
| # | Metric Name | Type | Labels | Domain | Signal Value | Rationale | Dashboard Placement |
|---|-------------|------|--------|--------|-------------|-----------|-------------------|
| 1 | migration_documents_processed_total | counter | status, source_type | Migration Core | HIGH | Primary progress indicator | Status Dashboard — Progress row |
| 2 | jvm_memory_used_bytes | gauge | area, id | JVM / Runtime | MEDIUM | Heap pressure during large batches | Processes Dashboard — Resources row |
| 3 | jvm_classes_loaded_classes | gauge | — | JVM / Runtime | LOW | Static metric, no migration relevance | EXCLUDED |
```

**Do not proceed to Phase 2 until every metric is classified.**

---

## PHASE 2: MIGRATION FLOW MAPPING

Read every document in the Migration Knowledge Base folder. Build a mental model of the migration flow.

### Step 2.1 — Extract the migration process stages

Identify every distinct stage/step in the migration pipeline. For each stage, record:

| Field | Description |
|-------|-------------|
| `stage_name` | Human-readable name (e.g., "Document Extraction", "Content Transformation", "Pithos Upload") |
| `stage_order` | Sequence position in the pipeline |
| `inputs` | What enters this stage (documents, metadata, content streams) |
| `outputs` | What exits this stage |
| `external_dependencies` | Systems called (IBM CM API, Pithos API, database, queue) |
| `failure_modes` | What can go wrong at this stage (timeouts, validation failures, format errors) |
| `metrics_mapped` | Which Prometheus metrics from Phase 1 correspond to this stage |

### Step 2.2 — Identify latency-critical paths

For each stage, determine:
- Is there a histogram or summary metric that captures this stage's duration?
- If yes, map the metric to the stage explicitly.
- If no, flag this as an **observability gap** — a stage with no latency visibility.

### Step 2.3 — Identify error/retry paths

Map which metrics track:
- Error counts per stage
- Retry counts per stage
- Dead-letter / skip / quarantine counts
- Validation failure counts

### Step 2.4 — Output the flow-to-metrics mapping

```markdown
| Stage | Order | External Deps | Latency Metric | Error Metric | Retry Metric | Gaps |
|-------|-------|--------------|----------------|-------------|-------------|------|
| Document Extraction | 1 | IBM CM API | cm_extract_duration_seconds | cm_extract_errors_total | cm_extract_retries_total | — |
| Content Transformation | 2 | None | transform_duration_seconds | transform_errors_total | — | No retry metric |
| Pithos Upload | 3 | Pithos API | pithos_upload_duration_seconds | pithos_upload_errors_total | pithos_upload_retries_total | — |
```

---

## PHASE 3: DASHBOARD DESIGN

Now — and only now — design the two dashboards. Every panel must trace back to a HIGH or MEDIUM signal metric from Phase 1, mapped to a migration stage from Phase 2.

---

### DASHBOARD 1: Migration Status Dashboard

**Purpose**: Executive and operational overview. Answers: "Is the migration healthy? How far along are we? Are there errors?"

**Target audience**: SRE on-call, migration project lead, engineering manager checking status.

**Design principles**:
- Top row = instant situational awareness (stat panels, gauges)
- Middle rows = trends over time (time series panels)
- Bottom rows = error/exception detail (tables, logs)
- Left-to-right flow: progress → throughput → errors → resource health
- Use Grafana variables for filtering by batch ID, source type, date range, or any high-value label dimension

#### Required sections (rows):

**Row 1 — Migration Headline Stats** (stat panels, single-value)
Design panels that answer these questions at a glance:
- Total documents migrated (cumulative counter)
- Documents migrated in current batch / time window
- Current migration rate (documents/minute or documents/hour)
- Overall success rate (% successful vs. total attempted)
- Current error rate (errors per unit time or % of attempts)
- Migration uptime / last active timestamp

For each stat panel, specify:
- Panel title
- PromQL query (exact, executable)
- Unit / format (short, percent, docs/min, etc.)
- Thresholds (green/yellow/red) with specific values and rationale
- Value mapping if applicable

**Row 2 — Migration Progress Over Time** (time series panels)
- Cumulative documents processed over time (monotonically increasing line)
- Throughput rate over time (rate of processing — should show if migration is accelerating, steady, or stalling)
- Documents pending / remaining (if a gauge or calculable metric exists)
- Batch completion markers (if batch IDs are tracked)

For each time series panel, specify:
- Panel title
- One or more PromQL queries with legend format strings
- Y-axis unit and scale
- Any useful annotations (e.g., batch start/end events)

**Row 3 — Error and Failure Analysis** (time series + table)
- Error count over time, broken down by error type / stage / label
- Error rate as a percentage of total processing
- Top errors table (grouped by error category, sorted by count descending)
- Retry volume over time (distinguish retries from terminal failures)

For each panel, specify:
- PromQL queries with label breakdowns using `by (label_name)`
- Table transformations (if using table panel with instant query)
- Alert thresholds if applicable

**Row 4 — Dependency Health** (time series or stat panels)
- IBM CM API availability / error rate (outbound call success rate)
- Pithos API availability / error rate
- Database connection pool utilization
- Queue depth / consumer lag (if applicable)
- HTTP client latency to each external dependency (p50, p95, p99)

**Row 5 — Resource Utilization** (time series — only MEDIUM-rated JVM/runtime metrics)
- Heap memory used vs. committed vs. max
- GC pause time and frequency
- Thread pool utilization (active threads vs. max)
- CPU usage (process CPU)
- Connection pool usage (active vs. idle vs. max)

Only include resource panels that are MEDIUM-rated (i.e., they correlate with migration performance). Do NOT include every JVM metric.

#### Dashboard-level configuration:
- **Variables**: Define Grafana template variables for every high-cardinality label that enables useful filtering. Specify variable name, label, query, and whether it's multi-select.
- **Time range**: Default to last 6 hours, with quick options for 1h, 6h, 24h, 7d.
- **Refresh**: Auto-refresh every 30 seconds.
- **Annotations**: If any metric or log marks batch boundaries or migration events, configure as annotations.

---

### DASHBOARD 2: Migration Processes Dashboard (Latency & Stage Analysis)

**Purpose**: Engineering deep-dive. Answers: "Where is the latency? Which stage is the bottleneck? Where do we lose time?"

**Target audience**: SRE debugging slow batches, developer optimizing migration code.

**Design principles**:
- Organize panels by migration stage (from Phase 2 flow mapping)
- Each stage gets its own row with: latency distribution, throughput, errors, and dependency calls
- Heatmaps for latency distribution (histogram metrics)
- p50/p95/p99 overlays to show tail latency
- Enable comparison: "is Stage X slower today than yesterday?"
- Use Grafana variables for filtering by batch, time window, document type

#### Required sections (rows):

**Row 0 — End-to-End Pipeline Latency** (top-level view before stage drill-down)
- Total document processing latency (end-to-end, all stages combined)
- p50, p95, p99 as separate lines or stat panels
- Latency heatmap (time on X-axis, latency buckets on Y-axis, color = count)
- Stage-by-stage latency breakdown as stacked bar or stacked area (if metrics allow attribution)

**Row N — One row per migration stage** (repeat for each stage identified in Phase 2)

For each stage, create panels:

1. **Stage Latency Distribution** (heatmap panel)
   - PromQL: Use `histogram_quantile()` on the stage's histogram metric
   - Display as heatmap with time on X, latency buckets on Y
   - Color scheme: standard (green-yellow-red)
   - Specify bucket configuration

2. **Stage Percentile Trends** (time series panel)
   - p50, p95, p99 as three lines
   - PromQL: `histogram_quantile(0.5, rate(metric_bucket[5m]))` etc.
   - Y-axis: seconds or milliseconds
   - Legend format: `p50 — {stage_name}`, `p95 — {stage_name}`, etc.

3. **Stage Throughput** (time series panel)
   - Rate of items processed through this stage per unit time
   - PromQL: `rate(metric_total[5m])`
   - Compare with previous stage's throughput to identify where items are queuing

4. **Stage Error Rate** (time series panel)
   - Errors at this stage over time
   - Broken down by error type if label exists
   - PromQL with `rate()` and `by (error_type)` or equivalent label

5. **Stage Dependency Latency** (time series — only if stage calls an external system)
   - Outbound call latency to the external dependency
   - p50/p95/p99 of the external call
   - Distinguish between: time spent waiting for dependency vs. time spent in application logic (if metrics allow)

**Final Row — Observability Gaps & Recommendations**
Based on Phase 2 gap analysis, include a text panel listing:
- Migration stages with no latency metric (blind spots)
- Stages with no error metric
- Stages with no retry visibility
- Recommendations for additional instrumentation

#### Dashboard-level configuration:
- **Variables**: Same as Dashboard 1, plus stage-specific filters if applicable
- **Time range**: Default to last 1 hour (tighter window for latency debugging)
- **Refresh**: Auto-refresh every 10 seconds
- **Linked dashboards**: Add a link back to Dashboard 1 (Status) for context switching

---

## PHASE 4: OUTPUT FORMAT

Produce exactly two files:

### File 1: `migration-status-dashboard.md`

Structure:
```
# Migration Status Dashboard — IBM CM → Pithos

## Dashboard Metadata
- Dashboard title
- UID (suggest a slug)
- Tags
- Default time range
- Refresh interval
- Variables (full specification)

## Metric Classification Summary
(Condensed table from Phase 1 — only HIGH and MEDIUM metrics used in this dashboard)

## Row 1: Headline Stats
### Panel 1.1: [Title]
- **Type**: Stat
- **Query**: `exact_promql_here`
- **Unit**: ...
- **Thresholds**: Green < X, Yellow < Y, Red >= Y
- **Rationale**: Why this panel matters

### Panel 1.2: [Title]
...

## Row 2: Progress Over Time
### Panel 2.1: [Title]
- **Type**: Time Series
- **Queries**:
  - A: `promql` | Legend: `{{label}}`
  - B: `promql` | Legend: `{{label}}`
- **Y-Axis**: Unit, min, max
- **Rationale**: ...

(Continue for every panel in every row)

## Observability Gaps
- List any questions this dashboard CANNOT answer with available metrics
```

### File 2: `migration-processes-dashboard.md`

Structure:
```
# Migration Processes Dashboard — Latency & Stage Analysis

## Dashboard Metadata
(Same structure as Dashboard 1)

## Migration Flow Summary
(Condensed flow-to-metrics mapping from Phase 2)

## Row 0: End-to-End Pipeline Latency
### Panel 0.1: [Title]
...

## Row 1: [Stage Name]
### Panel 1.1: Stage Latency Heatmap
- **Type**: Heatmap
- **Query**: `promql`
- **Bucket configuration**: ...
- **Color scheme**: ...

### Panel 1.2: Stage Percentile Trends
...

(Continue for every stage)

## Observability Gaps & Instrumentation Recommendations
- Stages without latency metrics
- Stages without error metrics
- Recommended metric additions with suggested metric names and types
```

---

## CRITICAL RULES

1. **Every PromQL query must be syntactically valid and executable.** Do not use placeholder metric names. Use the exact metric names from the `/actuator/prometheus` payload.
2. **Every panel must have an explicit rationale** — why is this panel on the dashboard? What question does it answer?
3. **Do not include LOW-rated metrics** in either dashboard. If a JVM metric didn't earn MEDIUM or higher, it does not get a panel.
4. **Do not create generic Grafana dashboards.** Every panel must be specific to this migration application's actual metrics. If a metric doesn't exist in the payload, don't invent a panel for it — flag it as a gap.
5. **Histogram queries must use the correct bucket metric name** (typically `_bucket` suffix). Verify the actual suffix from the Prometheus payload before writing `histogram_quantile()` queries.
6. **Label names in queries must match the actual labels** in the Prometheus payload. Do not guess label names.
7. **Thresholds must have rationale**, not arbitrary numbers. If you don't know the right threshold, say so and suggest a baseline approach (e.g., "set initial threshold at p95 of first week's data, then tune").
8. **Variable definitions must include the PromQL query** that populates them (e.g., `label_values(migration_documents_processed_total, source_type)`).
9. **Flag every observability gap** — if a migration stage from the KB has no corresponding metric, call it out explicitly with a recommended metric name and type.
10. **Panel sizing hints**: Suggest Grafana grid positions (w, h, x, y) or at minimum specify "full width", "half width", "quarter width" for each panel.

---

## INPUT FILES

### Input 1: Prometheus Metrics
The content below is the raw output from the migrate application's `/actuator/prometheus` endpoint:

```
<PASTE THE FULL /actuator/prometheus OUTPUT HERE>
```

### Input 2: Migration Knowledge Base
The following files describe the migration flow. Read all of them before starting Phase 2:

```
<PASTE OR REFERENCE THE KB FOLDER CONTENTS HERE>
```
