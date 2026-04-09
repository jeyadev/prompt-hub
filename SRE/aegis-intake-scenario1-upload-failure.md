# AEGIS Demo Scenario Intake — Document Upload Failure

> **For**: Aayushi (CDS Developer)
> **From**: Jey (SRE)
> **Purpose**: AEGIS needs real CDS production details to generate realistic incident runbooks for the demo. I've pre-filled my best guesses — **correct, replace, or expand anything that's wrong or incomplete**. Even partial answers help. Flag anything you're unsure about with `[UNSURE]`.
>
> **Time needed**: ~20-30 min (or we can do a quick call and I'll fill it in)

---

## 1. Detection — How do we know this is happening?

| Question | Pre-filled (correct/replace) |
|----------|------------------------------|
| Which CDS service(s) are involved? (exact names) | `[FILL: exact service name(s) that handle document upload]` |
| What does the user/caller experience? | `[FILL: e.g., HTTP 500? timeout? specific error message returned to caller?]` |
| What Splunk query would confirm this is happening? | `[FILL: exact SPL or close approximation, e.g., index=cds_prod sourcetype=... error_code=...]` |
| What Splunk index/sourcetype is relevant? | `[FILL: index name, sourcetype]` |
| Key Splunk fields/values that indicate the failure | `[FILL: e.g., status=500, error_code=DOC_UPLOAD_FAILED, exception_class=...]` |
| What Dynatrace/AppDynamics signal would show this? | `[FILL: service flow failure? exception count spike? specific service name in Dynatrace?]` |
| Is there a Grafana dashboard or Prometheus metric? | `[FILL: metric name if any, e.g., cds_upload_error_total]` |
| What does "normal" look like? (baseline) | `[FILL: e.g., ~X uploads/hour, success rate >99.X%, typical response time Xms]` |

---

## 2. Common Root Causes — What typically causes upload failures?

*Top causes you've actually seen in production. Rank by how often they occur.*

| # | Root Cause | How you confirm it (what to check) | Rough frequency |
|---|-----------|--------------------------------------|-----------------|
| 1 | `[FILL: e.g., downstream storage service unavailable]` | `[FILL: how do you verify this?]` | `[FILL: e.g., ~40%]` |
| 2 | `[FILL: e.g., payload validation failure / malformed document]` | `[FILL]` | `[FILL]` |
| 3 | `[FILL: e.g., auth/token expiry for service-to-service call]` | `[FILL]` | `[FILL]` |
| 4 | `[FILL: e.g., queue/message broker connectivity issue]` | `[FILL]` | `[FILL]` |
| 5 | `[FILL: e.g., resource exhaustion — thread pool / connection pool]` | `[FILL]` | `[FILL]` |

---

## 3. Validation Steps — The actual L1/L2 triage sequence

*Walk me through what a competent on-call engineer does today, step by step. Order matters.*

| Step | What to check | How to check it (exact tool / query / screen) | "Bad" result looks like | "Good" result looks like |
|------|--------------|------------------------------------------------|------------------------|-------------------------|
| 1 | Confirm failure is real and ongoing (not a blip) | `[FILL: Splunk query with time window, e.g., last 15 min error count]` | `[FILL: e.g., >X errors in 15 min]` | `[FILL: e.g., <Y errors, already recovering]` |
| 2 | Check scope — which consumers are affected? | `[FILL: how to identify affected callers — Splunk field? API key? client ID?]` | `[FILL: e.g., multiple consumers affected]` | `[FILL: e.g., single consumer, likely caller-side issue]` |
| 3 | Check upstream dependency health | `[FILL: which dependency? how to check — Dynatrace service flow? health endpoint? Splunk?]` | `[FILL: dependency down/degraded]` | `[FILL: dependency healthy]` |
| 4 | Check for recent deployments/changes | `[FILL: where do you look — deployment dashboard? change log? specific tool?]` | `[FILL: deployment correlates with failure onset]` | `[FILL: no recent changes]` |
| 5 | Check resource health (CPU, memory, connections) | `[FILL: Dynatrace host view? Grafana dashboard? specific metric?]` | `[FILL: resource exhaustion visible]` | `[FILL: resources normal]` |
| 6 | Check specific error details in logs | `[FILL: Splunk query for stack traces / error details]` | `[FILL: specific exception pattern]` | `[FILL: N/A]` |
| 7 | `[ADD MORE STEPS IF NEEDED]` | | | |

---

## 4. Remediation — What fixes it?

*For each root cause in Section 2:*

| Root Cause | Immediate Fix (L1/L2 can do) | Needs Escalation (L3/dev) | Rollback / Safeguard |
|-----------|------------------------------|--------------------------|---------------------|
| `[Root cause 1]` | `[FILL: e.g., restart service, clear queue, switch to fallback]` | `[FILL: when to escalate]` | `[FILL: rollback option if any]` |
| `[Root cause 2]` | `[FILL]` | `[FILL]` | `[FILL]` |
| `[Root cause 3]` | `[FILL]` | `[FILL]` | `[FILL]` |
| `[Root cause 4]` | `[FILL]` | `[FILL]` | `[FILL]` |
| `[Root cause 5]` | `[FILL]` | `[FILL]` | `[FILL]` |

---

## 5. Blast Radius & Dependencies

| Question | Your Answer |
|----------|------------|
| Which downstream consumers are affected when uploads fail? | `[FILL: specific consumer app names or teams]` |
| Upstream dependencies that could cause this | `[FILL: e.g., storage layer, auth service, message queue — exact names]` |
| Does this cascade to other CDS services? | `[FILL: which ones and how?]` |
| Is there a circuit breaker or fallback in place? | `[FILL: yes/no, if yes — how does it behave?]` |
| Is there a dead letter queue or retry mechanism? | `[FILL: what happens to failed uploads — lost? retried? queued?]` |

---

## 6. Artifacts & References

| Item | Link / Location |
|------|----------------|
| Confluence page(s) covering the upload flow | `[FILL: URL]` |
| Existing runbook for upload failures (if any) | `[FILL: URL or "none"]` |
| Splunk saved searches related to upload monitoring | `[FILL: search name or URL]` |
| Dynatrace service flow showing upload path | `[FILL: URL or description]` |
| Past incident tickets for upload failures | `[FILL: ServiceNow IDs or "will look up"]` |
| Architecture diagram showing upload flow | `[FILL: URL or "none"]` |

---

## 7. Demo Realism — Sample Data

*This makes or breaks the demo. AEGIS will display these in the UI.*

### Sample Splunk log lines for upload failure (2-3 lines)
```
[FILL: paste or write realistic log lines, e.g.,
2026-04-09T14:23:45.123Z ERROR [upload-service] DocumentUploadHandler - Failed to upload document docId=ABC123 clientId=AWM-PORTFOLIO error="Connection refused to storage backend" requestId=req-789xyz
]
```

### Sample exception / stack trace (abbreviated is fine)
```
[FILL: paste or write a realistic abbreviated stack trace, e.g.,
com.jpmc.cds.upload.StorageException: Connection refused
    at com.jpmc.cds.upload.StorageClient.put(StorageClient.java:142)
    at com.jpmc.cds.upload.DocumentUploadHandler.handle(DocumentUploadHandler.java:87)
    ...
]
```

### CDS-specific jargon or field names AEGIS should use
```
[FILL: any CDS-specific terminology that would make the demo look authentic,
e.g., specific field names, service identifiers, queue names, error codes]
```

---

> **Questions? Gaps?** Flag anything with `[UNSURE]` or `[NEED TO CHECK]` and I'll follow up. Partial is fine — something is always better than nothing.
