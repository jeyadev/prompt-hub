Good. A Confluence page is the closest thing humans have to “shared memory,” so let’s make it actually usable.

Below is a **ready-to-paste Confluence doc** for **Start of Day (SOD) On-Call**. It’s designed to be fool-proof: timeboxed, checklist-driven, and forces the right artifacts (email + triage hygiene + metrics).

---

## Confluence Page: Start of Day (SOD) On-Call Procedure

### Page title

**On-Call Start of Day (SOD) Runbook**

### Purpose

Run a consistent **daily health + incident queue control** routine so:

* we catch service degradations early,
* incidents don’t rot in the queue,
* the team has a shared daily snapshot.

### Scope

Applies to: **Primary On-Call Engineer** (business days; can be adapted for weekend shifts).

### Inputs

* Health check runbook: **[LINK: Health Check Runbook]**
* Incident list/dashboard: **[LINK: Incident Queue / ServiceNow / Jira filter]**
* On-call responsibilities (general): **[LINK: On-Call Engineers Responsibilities]**
* Key dashboards/logs (optional): **[LINKS: Dynatrace dashboards, Splunk searches, etc.]**

---

## SOD Timebox

**Target duration:** 15–20 minutes
**Hard stop:** 30 minutes (if it’s taking longer, you likely have a real issue and should escalate)

---

## SOD Checklist

### 1) Confirm On-Call Ownership (2 mins)

* ✅ Verify you are the **Primary On-Call** for today
* ✅ Confirm backup contact / secondary on-call: **[name/rotation link]**
* ✅ Check for known planned risk: deployments, maintenance, partner changes

**If primary is OOO:** announce backup owner in the on-call channel immediately.

---

### 2) Run the Health Check (5–10 mins)

Follow the documented steps here:

* **Health Check Runbook:** **[LINK]**

**Minimum expected output**

* ✅ Health check completed
* ✅ Any anomalies noted with evidence link(s)
* ✅ If anomalies are user-impacting or unclear: declare an incident / escalate (see Escalation section)

---

### 3) Incident Queue Triage (5–10 mins)

Open the incident queue:

* **Incident Queue:** **[LINK]**

#### 3.1 Triage rules (mandatory)

For **every new/untriaged incident**, ensure:

* **Category (Primary):** Availability / Latency / Errors / Data / Auth / Dependency / Deployment / Capacity / Other
* **Category (Secondary):** service/module/component
* **Severity:** P1–P5 (or your org’s scheme)
* **Owner:** person/team
* **Next action:** investigate / mitigate / rollback / waiting / needs dev / vendor
* **Evidence link:** dashboard / log query / runbook step

#### 3.2 Reroute rules (if incident belongs to another team)

When rerouting, include:

* 1–2 line symptom + impact
* what you checked
* why it’s theirs (dependency/ownership)
* clear ask (what you need)
* evidence link(s)

#### 3.3 “No rot” rule

* No incident should remain **New/Unassigned** beyond **[X mins]** during business hours.
* Any incident older than **24h** must have a documented reason + explicit next step.

---

## Daily Metrics to Report (for visibility)

Capture:

* **Incident count yesterday:** #
* **Incident count today (so far):** #
* **Closed since yesterday:** #
* **Oldest open incident age:** duration (+ incident link)

Optional (recommended):

* **New incidents since yesterday:** #
* **Rerouted today:** #
* **Reopened incidents:** #

---

## Send the SOD Health Check Email (Template)

**Subject:** `SOD On-Call Health Check – <DATE> | Incidents: Yday <#> | Today <#> | Closed since yday <#>`

**Body:**

```
Hi team,

SOD health check completed as per runbook:
- Runbook (Confluence): <LINK_TO_HEALTHCHECK_RUNBOOK>

Incident queue snapshot:
- Incident count (yesterday): <#>
- Incident count (today so far): <#>
- Closed since yesterday: <#>
- Oldest open incident age: <DURATION> (Incident: <ID/LINK>)

Triage actions taken:
- Categorized/updated: <#>
- Rerouted to other teams: <#> (links: <…>)
- Resolved/closed: <#> (links: <…>)

Notes / risks:
- <1–3 bullets on anomalies, recurring alerts, planned risks>

General on-call responsibilities:
- <LINK_TO_ONCALL_RESPONSIBILITIES_PAGE>

Thanks,
<ONCALL_NAME>
```

---

## Escalation Guidance (SOD)

Escalate immediately if:

* Health check shows **user-impacting degradation**
* There is **uncertainty** on scope/impact after a short investigation
* Multiple related incidents appear (possible systemic issue)
* A P1/P2 incident is present and not actively owned

Escalation paths:

* On-call channel: **[LINK]**
* SRE escalation: **[LINK / Pager / rota]**
* App team escalation: **[LINK / group / rota]**

---

## Definition of Done for SOD

SOD is complete only when:

* ✅ Health check done + runbook linked
* ✅ Incident queue triaged (no New/Unassigned rot)
* ✅ Incident stats captured
* ✅ SOD email sent

---

## Appendix

* Common incident categories mapping: **[LINK]**
* Known noisy alerts list + owner: **[LINK]**
* Incident commander / comms process: **[LINK]** (optional but recommended)

---

If you want this to be *actually* fool-proof, add one more thing at the top: **“If you can’t finish SOD in 30 mins, you likely have an active incident. Stop reporting and start incident management.”** That one sentence saves entire teams from pretending everything is normal while production is melting.
