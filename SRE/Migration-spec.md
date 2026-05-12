# Feature Spec: Latency-Aware Circuit Breaker for Migration Batch Execution

**Jira Story**: Implement latency-aware circuit breaker in migration utility to auto-pause batches on CM performance degradation  
**Version**: 1.0  
**Status**: Ready for Implementation  
**Scope**: Migration Utility — Backend (batch execution engine) + UI (batch trigger and monitoring views)

---

## 1. Overview

### 1.1 Problem Statement

Migration batches currently run without real-time awareness of Content Manager (CM) server health. When migration and live traffic share the same CM infrastructure, a running batch can degrade latency for active user-facing transactions (find, upload, download) without any automated detection or intervention. The on-call engineer has no automated safety net — degradation must be detected and acted on manually.

### 1.2 Feature Summary

This feature adds a **circuit breaker** to the migration batch execution lifecycle. It continuously monitors CM health via Dynatrace MCP during an active batch run, compares live metrics against a pre-batch baseline, and takes graduated protective action when degradation is detected. All pauses require human decision to resume or abort. All events are logged for audit and migration scheduling intelligence.

### 1.3 Design Principles

- **Dynatrace MCP is the primary signal source.** No dependency on Splunk for circuit breaker logic.
- **Relative deviation over static thresholds.** Baselines are captured at batch start, not hardcoded.
- **Graduated response, not binary.** Throttle first, pause second.
- **Human-in-the-loop for resume/abort.** The circuit breaker stops the batch; only a human restarts it, with context provided to inform that decision.
- **Fail-closed with acknowledged override.** Observability tool unavailability blocks batch start by default. Engineers can explicitly override with acknowledgement.
- **Dual signals: latency + error rate.** Either signal breaching threshold is sufficient to trigger.

---

## 2. Functional Requirements

### 2.1 Pre-Flight Health Check

**Trigger**: Initiated automatically when an engineer triggers a new migration batch run.

**Behaviour**:
1. The system queries the Dynatrace MCP for current latency (p95) and error rate for the three key CM transactions: **find**, **upload**, **download**.
2. The system attempts to verify Splunk connectivity status (ping/lightweight check only — not used for circuit breaker logic).
3. A health indicator is displayed in the batch trigger UI showing:
   - Dynatrace connectivity: `Connected` / `Unavailable`
   - Splunk connectivity: `Connected` / `Unavailable`
   - Current CM latency baseline (p95 for find, upload, download)
   - Current CM error rate

**Blocking conditions**:
- If Dynatrace MCP is **unavailable**: batch start is **blocked**. Engineer must acknowledge via an explicit checkbox — _"I acknowledge Dynatrace is unavailable. I am taking manual responsibility for CM health monitoring during this batch run."_ — before the start action is enabled.
- If Splunk is unavailable: **non-blocking**. A warning indicator is shown. No acknowledgement required.
- If current CM latency or error rate is **already elevated above threshold** at pre-flight time: batch start is **blocked** with a warning — _"CM performance is already degraded. Starting migration may worsen live traffic impact."_ Engineer must acknowledge to override.

**Output**: The pre-flight snapshot (latency p95 and error rate per transaction type, timestamp) is stored as the **batch run baseline** and attached to the run record. This baseline is used for all relative deviation calculations during the run.

---

### 2.2 Baseline Capture

**At batch start**, the system records and persists the following per batch run:

| Field | Description |
|-------|-------------|
| `baseline_latency_find_p95` | p95 latency for find transactions at batch start |
| `baseline_latency_upload_p95` | p95 latency for upload transactions at batch start |
| `baseline_latency_download_p95` | p95 latency for download transactions at batch start |
| `baseline_error_rate` | Overall CM error rate at batch start |
| `baseline_captured_at` | Timestamp |
| `batch_id` | Reference to the migration batch run |

**Warm-up exclusion**: The circuit breaker polling loop does **not** start evaluating signals until a configurable warm-up period after batch start (default: **120 seconds**). This prevents false positives from the initial load ramp of the batch.

---

### 2.3 Continuous Health Polling

**During an active batch run**, the system polls Dynatrace MCP on a configurable interval (default: **60 seconds**).

**Per poll, the system fetches**:
- p95 latency for find, upload, and download transactions (rolling window: last 5 minutes)
- CM error rate (rolling window: last 5 minutes)

**Deviation calculation**:

For each metric:

```
relative_deviation = (current_value - baseline_value) / baseline_value * 100
```

A **latency breach** is defined as: relative deviation exceeds the configured latency deviation threshold (default: **+40%**) for any of the three transaction types.

An **error rate breach** is defined as: current error rate exceeds the configured error rate threshold (default: **+50% relative to baseline**, or absolute threshold if baseline error rate is near zero).

Either signal breaching independently is sufficient to trigger the graduated response.

---

### 2.4 Graduated Response

| Condition | System Action | Engineer Notification |
|-----------|--------------|----------------------|
| **1 consecutive poll breach** | Reduce batch concurrency/rate by a configurable step (default: 50% reduction) | Alert: degradation detected, batch throttled. Includes current vs. baseline metrics. |
| **2–3 consecutive poll breaches** | Auto-pause batch at next safe batch boundary. No new requests dispatched. | Alert: batch auto-paused. Includes context packet (see §2.6). Engineer decision required. |
| **Severe breach** (deviation ≥ 2× threshold on any single poll) | Auto-pause immediately. Does not wait for consecutive confirmation. | Alert: severe degradation detected, batch immediately paused. Includes context packet. |

**Safe pause boundary**: The system finishes processing the current in-flight sub-batch unit before setting the paused state. It does not interrupt mid-batch-unit processing.

**Throttle → Pause promotion**: If throttling (1-breach state) continues for N consecutive polls without recovery (default: N=3), the system escalates directly to auto-pause regardless of whether the threshold is still breached.

---

### 2.5 Quiet Period Post-Pause

After a batch is auto-paused:

1. A **minimum quiet period** begins (default: **90 seconds**). No polls are evaluated during this window.
2. Purpose: allow in-flight CM requests dispatched before pause to drain, so the next poll reflects a clean post-load signal.
3. After the quiet period, the system runs one health re-evaluation poll and appends the result to the context packet available to the engineer.
4. The batch remains paused until explicit engineer action. It does **not** auto-resume.

---

### 2.6 Engineer Notification — Context Packet

Every pause notification sent to the responsible engineer must include:

| Field | Description |
|-------|-------------|
| Batch ID | Which batch was paused |
| Pause trigger | Which signal triggered the pause (latency breach / error rate breach / severe breach / throttle escalation) |
| Baseline metrics | Latency p95 (find, upload, download) and error rate at batch start |
| Metrics at pause trigger | Values that caused the pause |
| Current metrics (post-quiet period) | Values after in-flight drain — is it recovering? |
| Breach signal timeline | Last N poll results showing the trend leading to pause |
| Direct link | Deep link to Dynatrace view filtered to the migration batch time window |
| Decision actions | Resume / Abort (with confirmation and mandatory reason logging) |

The notification channel is configurable (email, internal messaging, or webhook) and set per batch run or globally.

---

### 2.7 Resume and Abort

**Resume**:
- Requires explicit engineer action via the migration utility UI or notification action.
- Engineer must provide a **reason** (free text, mandatory, minimum 10 characters).
- System logs: engineer ID, timestamp, reason, current CM metrics at resume time.
- After resume, the throttle state is **reset** (full batch rate restored). The circuit breaker continues polling from the post-resume state.
- The baseline used for deviation calculations **remains the original pre-batch baseline** — it is not recaptured on resume.

**Abort**:
- Requires explicit engineer action.
- Mandatory reason logging (same as resume).
- System logs abort event with full context packet snapshot.
- Triggers post-flight health check (see §2.8).

---

### 2.8 Post-Flight Health Check

Triggered after: batch **completes normally**, batch is **aborted**, or batch is **manually stopped**.

**Behaviour**:
1. System queries Dynatrace MCP for current latency and error rate.
2. Compares against the batch's stored baseline.
3. Records the **latency recovery delta**: how much latency has recovered since the batch ended.
4. Logs the full post-flight record to the batch run audit log.

**Purpose**: Over multiple migration runs, this builds a dataset correlating batch execution with CM impact and recovery time. This data informs future migration scheduling decisions.

---

### 2.9 Observability Tool Unavailability Handling

| Scenario | Behaviour |
|----------|-----------|
| Dynatrace unavailable at pre-flight | Block batch start. Show warning. Require acknowledgement checkbox to override. |
| Dynatrace unavailable mid-run (poll fails) | Treat as circuit breaker uncertainty: throttle batch immediately, alert engineer, log poll failure. If Dynatrace unavailable for N consecutive polls (default: 3), auto-pause and alert. |
| Splunk unavailable at pre-flight | Show warning indicator only. Non-blocking. |
| Dynatrace poll timeout | Treat same as unavailable for that poll. Log timeout. |

All unavailability events are logged as circuit breaker audit events.

---

## 3. Non-Functional Requirements

| Requirement | Detail |
|-------------|--------|
| **Polling overhead** | Dynatrace MCP query must complete within a configurable timeout (default: 15 seconds). If exceeded, treat as unavailable. |
| **Concurrency safety** | Pause flag and throttle state must be thread-safe. Circuit breaker state must not be corruptible by concurrent poll evaluations. |
| **No polling during pause** | Polling loop suspends during the quiet period post-pause. Resumes after quiet period for the one re-evaluation poll, then suspends again until engineer action. |
| **Configuration** | All thresholds, intervals, and timeouts must be configurable without code change (application config or admin UI). |
| **Idempotency** | Duplicate pause triggers (e.g., two polls both detecting breach simultaneously) must result in exactly one pause event. |

---

## 4. Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `cb.warmup_seconds` | 120 | Seconds after batch start before circuit breaker begins evaluating |
| `cb.poll_interval_seconds` | 60 | Polling frequency during active run |
| `cb.latency_deviation_threshold_pct` | 40 | Relative % increase over baseline to constitute a latency breach |
| `cb.error_rate_deviation_threshold_pct` | 50 | Relative % increase over baseline to constitute an error rate breach |
| `cb.severe_breach_multiplier` | 2.0 | Multiplier on threshold to classify a breach as severe (immediate pause) |
| `cb.consecutive_breaches_for_pause` | 2 | Consecutive breach count (excluding severe) to trigger auto-pause |
| `cb.throttle_escalation_polls` | 3 | Consecutive throttled polls without recovery before escalating to pause |
| `cb.throttle_rate_reduction_pct` | 50 | Batch rate reduction applied on first breach |
| `cb.quiet_period_seconds` | 90 | Post-pause quiet period before re-evaluation poll |
| `cb.dt_poll_timeout_seconds` | 15 | Dynatrace MCP query timeout |
| `cb.dt_unavailable_pause_threshold` | 3 | Consecutive failed polls before auto-pause on DT unavailability |

---

## 5. Data and Audit Log

Every circuit breaker event must be written as a structured log entry. Minimum fields per event:

| Field | Description |
|-------|-------------|
| `event_type` | `PREFLIGHT_CHECK`, `BASELINE_CAPTURED`, `POLL_RESULT`, `THROTTLE_TRIGGERED`, `PAUSE_TRIGGERED`, `PAUSE_ACKNOWLEDGED`, `RESUME_DECISION`, `ABORT_DECISION`, `POSTFLIGHT_CHECK`, `DT_UNAVAILABLE` |
| `batch_id` | Migration batch run identifier |
| `timestamp` | ISO-8601 |
| `triggered_by` | `SYSTEM` or engineer ID |
| `latency_find_p95` | Current value at event time |
| `latency_upload_p95` | Current value at event time |
| `latency_download_p95` | Current value at event time |
| `error_rate` | Current value at event time |
| `deviation_latency_pct` | Relative deviation from baseline at event time |
| `deviation_error_rate_pct` | Relative deviation from baseline at event time |
| `reason` | Free text (mandatory for RESUME_DECISION and ABORT_DECISION) |
| `dt_available` | Boolean — was Dynatrace reachable at this event |

These logs should be queryable by batch_id and event_type. Persistence: same store as existing migration utility run history.

---

## 6. UI Requirements

> Framework is TBD. These requirements are framework-agnostic.

### 6.1 Batch Trigger Screen — Pre-Flight Panel

- Health indicator widget showing Dynatrace and Splunk connectivity status with visual status icons.
- Current CM metrics panel: latency p95 (find, upload, download) and error rate, fetched at trigger time.
- If Dynatrace unavailable: disable the batch start action. Render acknowledgement checkbox — _"I acknowledge Dynatrace is unavailable and accept manual monitoring responsibility."_ Start action re-enables only after checkbox is checked.
- If CM already degraded at pre-flight: warning banner. Require acknowledgement checkbox to override. Start action blocked until acknowledged.

### 6.2 Active Batch Run — Circuit Breaker Status Panel

- Live status indicator: `Healthy` / `Throttled` / `Paused` / `Severe Breach`.
- Current metrics vs. baseline delta shown continuously.
- Last poll timestamp.
- Circuit breaker event timeline (scrollable, most recent first).

### 6.3 Pause Decision Screen

Displayed to engineer on receiving a pause notification or navigating to a paused batch:

- Context packet displayed in full (as defined in §2.6).
- Two actions: **Resume** (with mandatory reason text input) and **Abort** (with mandatory reason text input).
- No action proceeds without a non-empty reason field.

---

## 7. Out of Scope (v1)

- Auto-resume on recovery (human decision required in v1; may be considered in v2).
- Structural changes to how batches are split or sized.
- Integration with CM server-side controls (e.g., throttling at CM layer — not in scope for migration utility).
- Historical trend analysis UI across multiple batch runs (audit log is captured; analysis tooling is future scope).
- Adjusting deviation thresholds mid-run without restart.

---

## 8. Open Questions for Implementation Team

1. What is the Dynatrace MCP query interface available in this environment? Confirm supported query types for p95 latency and error rate per transaction name.
2. What constitutes a "safe batch boundary" for pause — is there a natural unit (e.g., end of a batch chunk/page) the executor already tracks?
3. What is the existing batch run persistence model? (To confirm where baseline and audit log fields are stored.)
4. What notification channels are available and configured for on-call engineers?
5. Is there an existing admin configuration mechanism in the utility, or does configuration need to be introduced?

---

*Spec version 1.0 — authored for GitHub Copilot-guided implementation. All threshold values are defaults and must be validated against observed CM latency patterns before first production migration run.*
 
