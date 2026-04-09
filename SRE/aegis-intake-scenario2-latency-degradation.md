# AEGIS Demo Scenario Intake — Upload Latency Degradation

> **For**: Aayushi (CDS Developer)
> **From**: Jey (SRE)
> **Purpose**: AEGIS needs real CDS production details to generate realistic incident runbooks for the demo. I've pre-filled my best guesses — **correct, replace, or expand anything that's wrong or incomplete**. Even partial answers help. Flag anything you're unsure about with `[UNSURE]`.
>
> **Time needed**: ~20-30 min (or we can do a quick call and I'll fill it in)

---

## 1. Detection — How do we know latency has degraded?

| Question | Pre-filled (correct/replace) |
|----------|------------------------------|
| Which CDS service(s) are involved? (exact names) | `[FILL: exact service name(s) in the upload path]` |
| What does the user/caller experience? | `[FILL: e.g., uploads taking 10x longer? timeouts after X seconds? partial success with retries?]` |
| What's the normal upload latency? (P50, P95, P99) | `[FILL: e.g., P50=200ms, P95=800ms, P99=1.5s — or whatever the real numbers are]` |
| At what latency does it become a problem? (threshold) | `[FILL: e.g., P95 > 2s? P99 > 5s? caller timeout at Xs?]` |
| What Splunk query shows latency degradation? | `[FILL: exact SPL, e.g., index=cds_prod sourcetype=... \| stats avg(response_time) p95(response_time) by service]` |
| What Splunk index/sourcetype is relevant? | `[FILL: index name, sourcetype]` |
| Key Splunk fields for latency | `[FILL: e.g., response_time_ms, duration, elapsed — exact field name]` |
| Dynatrace/AppDynamics signal for latency | `[FILL: response time alert? service flow slowdown? specific service name?]` |
| Grafana dashboard or Prometheus metric | `[FILL: e.g., cds_upload_duration_seconds histogram, dashboard URL]` |
| Does latency degrade gradually or cliff-edge? | `[FILL: e.g., slow creep over hours vs. sudden jump]` |

---

## 2. Common Root Causes — What typically causes latency to increase?

*Top causes you've actually seen in production. Rank by frequency.*

| # | Root Cause | How you confirm it | Rough frequency |
|---|-----------|-------------------|-----------------|
| 1 | `[FILL: e.g., storage backend slow — disk I/O, GC pauses, replication lag]` | `[FILL: how to check]` | `[FILL: e.g., ~35%]` |
| 2 | `[FILL: e.g., connection pool exhaustion — waiting for available connections]` | `[FILL]` | `[FILL]` |
| 3 | `[FILL: e.g., noisy neighbor — one consumer sending oversized documents or burst traffic]` | `[FILL]` | `[FILL]` |
| 4 | `[FILL: e.g., database query degradation — missing index, table growth, lock contention]` | `[FILL]` | `[FILL]` |
| 5 | `[FILL: e.g., JVM/container resource pressure — GC overhead, memory pressure, CPU throttling]` | `[FILL]` | `[FILL]` |

---

## 3. Validation Steps — The actual L1/L2 triage sequence

*Walk me through what a competent on-call engineer does today when latency is reported. Order matters — what gets checked first?*

| Step | What to check | How to check it (exact tool / query / screen) | "Bad" result looks like | "Good" result looks like |
|------|--------------|------------------------------------------------|------------------------|-------------------------|
| 1 | Confirm latency is elevated and not a false alarm | `[FILL: Splunk query or Grafana panel — how to validate the claim?]` | `[FILL: e.g., P95 > Xms sustained for >Y min]` | `[FILL: e.g., brief spike, already recovered]` |
| 2 | Determine scope — all consumers or specific ones? | `[FILL: how to break down latency by consumer/caller?]` | `[FILL: all consumers affected = systemic]` | `[FILL: single consumer = likely caller-side or payload issue]` |
| 3 | Check for traffic anomalies | `[FILL: how to see request volume by consumer? Splunk query?]` | `[FILL: e.g., one consumer 5x normal volume]` | `[FILL: traffic within normal range]` |
| 4 | Check upstream/downstream dependency latency | `[FILL: which dependencies? Dynatrace service flow? Splunk?]` | `[FILL: dependency responding slowly]` | `[FILL: dependencies healthy — problem is internal]` |
| 5 | Check resource utilization (CPU, memory, GC, threads) | `[FILL: Dynatrace host view? Grafana? specific metrics?]` | `[FILL: e.g., GC pause >Xms, CPU >Y%, thread pool saturated]` | `[FILL: resources normal]` |
| 6 | Check connection pool / queue depth | `[FILL: where are these metrics? what tool?]` | `[FILL: pool exhausted, queue backing up]` | `[FILL: pool healthy, queue draining normally]` |
| 7 | Check for recent deployments or config changes | `[FILL: deployment dashboard, change log, specific tool]` | `[FILL: change correlates with latency onset]` | `[FILL: no recent changes]` |
| 8 | Check for payload anomalies (document size, type) | `[FILL: Splunk field for document size? how to identify outliers?]` | `[FILL: e.g., documents 10x normal size]` | `[FILL: payload sizes normal]` |
| 9 | `[ADD MORE STEPS IF NEEDED]` | | | |

---

## 4. Remediation — What fixes it?

*For each root cause in Section 2:*

| Root Cause | Immediate Fix (L1/L2 can do) | Needs Escalation (L3/dev) | Rollback / Safeguard |
|-----------|------------------------------|--------------------------|---------------------|
| `[Root cause 1]` | `[FILL: e.g., restart pods, scale up replicas, clear connection pool]` | `[FILL: when to escalate]` | `[FILL]` |
| `[Root cause 2]` | `[FILL]` | `[FILL]` | `[FILL]` |
| `[Root cause 3]` | `[FILL: e.g., rate-limit the noisy consumer, contact consumer team]` | `[FILL]` | `[FILL]` |
| `[Root cause 4]` | `[FILL]` | `[FILL]` | `[FILL]` |
| `[Root cause 5]` | `[FILL]` | `[FILL]` | `[FILL]` |

---

## 5. Blast Radius & Dependencies

| Question | Your Answer |
|----------|------------|
| Which downstream consumers notice latency first? | `[FILL: who has the tightest timeout? who complains first?]` |
| Do any consumers have SLAs on upload response time? | `[FILL: specific SLA numbers if known]` |
| Upstream dependencies in the upload latency path | `[FILL: storage, database, auth, queue — exact names and which ones contribute to latency]` |
| Does upload latency affect other CDS operations? | `[FILL: e.g., does slow upload block retrieval? affect queue processing?]` |
| Is there backpressure / queue buildup when latency spikes? | `[FILL: what happens to queued requests? do they pile up and make it worse?]` |
| At what point does latency become an outage? | `[FILL: e.g., caller timeouts at Xs = functional failure even though service is "up"]` |

---

## 6. Artifacts & References

| Item | Link / Location |
|------|----------------|
| Confluence page(s) covering the upload flow / architecture | `[FILL: URL]` |
| Existing runbook for latency issues (if any) | `[FILL: URL or "none"]` |
| Splunk saved searches for latency monitoring | `[FILL: search name or URL]` |
| Grafana dashboard for upload latency | `[FILL: URL]` |
| Dynatrace service flow showing upload path | `[FILL: URL or description]` |
| Past incident tickets for latency degradation | `[FILL: ServiceNow IDs or "will look up"]` |
| Capacity/sizing documentation | `[FILL: URL or "none"]` |

---

## 7. Demo Realism — Sample Data

*This makes or breaks the demo. AEGIS will display these in the UI.*

### Sample Splunk log lines showing latency degradation (2-3 lines)
```
[FILL: paste or write realistic log lines showing slow responses, e.g.,
2026-04-09T14:23:45.123Z WARN [upload-service] DocumentUploadHandler - Slow upload detected docId=ABC123 clientId=AWM-PORTFOLIO duration_ms=4523 threshold_ms=2000 requestId=req-789xyz
2026-04-09T14:23:47.456Z WARN [upload-service] StorageClient - Storage backend slow response operation=PUT duration_ms=3891 endpoint=storage-primary requestId=req-789xyz
]
```

### What does "normal" vs "degraded" look like side by side?
```
[FILL: e.g.,
Normal:  avg=180ms, P95=650ms, P99=1.2s, error_rate=0.1%
Degraded: avg=2.1s, P95=5.8s, P99=12s, error_rate=3.2% (timeouts)
]
```

### Sample Dynatrace/metric data showing the degradation pattern
```
[FILL: e.g., what does the Dynatrace service flow look like when latency is bad?
Which hop in the service flow shows the bottleneck?]
```

### CDS-specific jargon or field names AEGIS should use
```
[FILL: any CDS-specific terminology that would make the demo look authentic,
e.g., specific field names, service identifiers, metric names, thresholds]
```

---

## 8. Latency-Specific: Degradation vs. Failure Boundary

*This scenario is trickier than outright failure because it's a spectrum. Help me understand the thresholds.*

| Question | Your Answer |
|----------|------------|
| At what latency does the team get alerted? | `[FILL: alert threshold — P95 > X for Y minutes?]` |
| At what latency do consumers start timing out? | `[FILL: consumer-side timeout settings if known]` |
| At what latency does this become a SEV-1 vs SEV-2? | `[FILL: severity criteria]` |
| Does latency degradation self-resolve? (e.g., GC pause ends, traffic subsides) | `[FILL: how often does it fix itself vs. need intervention?]` |
| Is there auto-scaling or does someone manually scale? | `[FILL: auto-scale on Gaia? manual pod scaling? neither?]` |

---

> **Questions? Gaps?** Flag anything with `[UNSURE]` or `[NEED TO CHECK]` and I'll follow up. Partial is fine — something is always better than nothing.
