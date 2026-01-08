You’re close, but right now this “model” still relies on humans being consistently diligent. That’s adorable. Let’s make it *hard to screw up*.

## Solution: a fool-proof daily on-call operating model

### 1) Define the on-call day in 3 phases (SOD → Day Ops → EOD)

**SOD (Start of Day, 15–20 mins, timeboxed)**

1. **Run Health Check** (follow Confluence runbook)

   * Send *Start-of-Day Health Check* email with the runbook link.
2. **Incident Queue Triage** (single source of truth: incident list)

   * For every *new/untriaged* incident: **categorize + set owner + set next action**.
3. **Numbers snapshot** for the email

   * Incident count **yesterday**, **today**, **closed since yesterday**, and **oldest open age** (this one prevents quiet rot).

**Day Ops (light but continuous)**

1. **Queue hygiene every 2–3 hours (5 mins)**

   * No incident should sit in “New/Unassigned” beyond your SLA (example: 30 mins during business hours).
2. **Routing discipline**

   * If it’s not your team: reroute with *the right category*, *the right evidence*, and *a clear ask*.
3. **Resolve if applicable**

   * If you resolve: document the fix + update tags so it’s searchable later.

**EOD (End of Day, 10 mins)**

1. **Handoff note** (even if the next person is you, tomorrow-you is still a different person)

   * Top open incidents, current status, blockers, next actions, and risks.
2. **If anything smelled recurring**: create a follow-up task (automation, monitoring, runbook gap, ownership gap).

---

### 2) Make incident triage unambiguous (categories + required fields)

To prevent “vibes-based triage”, enforce **minimum required fields** for every incident touched by on-call:

**Required tags/fields**

* **Severity** (P1–P5) + **User impact** (none / partial / major)
* **Category (Primary)**: Availability / Latency / Errors / Data / Auth / Dependency / Deployment / Capacity / Other
* **Category (Secondary)**: service/module/component (CDS module, DB, MQ, storage, etc.)
* **Routing status**: Keep / Rerouted / Waiting on other team / Mitigated
* **Next action**: Investigate / Mitigate / Rollback / Awaiting logs / Vendor / Needs dev
* **Evidence link**: dashboard/log query/runbook step (one link minimum)

**Reroute rule (so other teams don’t hate you)**
When rerouting, always include:

* 1–2 line summary of symptom + impact
* what you checked (dashboards/logs)
* why it’s theirs (service ownership/dependency)
* what you need from them (clear ask)
* link(s)

---

### 3) Add hard guardrails (Definition of Done for the day)

This is how you stop “I was on-call” from meaning “I existed near alerts.”

**Daily DoD**

* ✅ Health check email sent with runbook link
* ✅ No incidents in “New/Unassigned” older than **X mins**
* ✅ Every open incident has: category + owner + next action + evidence link
* ✅ Any incident older than **24h** has an explicit reason + escalation/handoff note
* ✅ End-of-day handoff posted

---

### 4) Bake in escalation and “stop the bleeding” logic

You don’t want heroics, you want **predictability**.

**Time-based escalation (example)**

* P1: acknowledge < 5 mins, escalate to SRE + dev < 10 mins if not immediately obvious
* P2: acknowledge < 15 mins, escalate < 30 mins
* P3+: batch triage windows are fine, but never leave uncategorized

**If you’re stuck**

* After **30 mins of no progress** (P1/P2): escalate with what you tried + what you need next.

---

### 5) Reference the general responsibilities page, but keep your daily model separate

Your attached “On-Call Engineers Responsibilities” page is great as a **static reference** (alerts, user issues, deployments, postmortems). Keep it linked as “General Responsibilities”, but your daily model should be a **checklist + numbers + queue control**.

---

## The Start-of-Day Health Check email template (with incident stats)

Hi team,

SOD health check completed as per runbook:

* Health check runbook (Confluence): <LINK_TO_HEALTHCHECK_RUNBOOK>

Incident queue snapshot:

* Incident count (yesterday): <#>
* Incident count (today so far): <#>
* Closed since yesterday: <#>
* Oldest open incident age: <DURATION> (Incident: <ID/LINK>)

Triage actions taken:

* Categorized/updated: <#>
* Rerouted to other teams: <#> (links: <…>)
* Resolved/closed: <#> (links: <…>)

Notes / risks:

* <1–3 bullets on anything suspicious: recurring alert, noisy monitor, deployment risk, dependency instability>

General on-call responsibilities (reference):

* <LINK_TO_ONCALL_RESPONSIBILITIES_PAGE>

Thanks,
<ONCALL_NAME>

---

## Data reasoning: what makes this “fool proof”

* **Queue hygiene metrics** (uncategorized age, oldest open age) prevent silent backlog buildup.
* **Required fields** convert tribal knowledge into operational data you can trend.
* **Timeboxes + escalation timers** stop incidents from dying in one person’s DMs.

---

## Systems analysis: common failure modes and how this model blocks them

* **“Nobody owned it”** → forced owner + next action on every open incident
* **“Wrong team, bounced around”** → reroute requires evidence + clear ask
* **“On-call got busy, backlog rotted”** → uncategorized SLA + oldest-open spotlight
* **“Same issue every week”** → EOD follow-up task requirement

---

## Validation & simulation (so you know it works, not just feels nice)

* Weekly audit: sample 10 incidents → check required fields completeness rate
* Track: median time-to-categorize, % reroutes accepted without bounce-back, oldest open age trend
* Run a tabletop: “P1 hits at 11am during deploy” → verify escalation + comms + reroute behavior

---

## Ops-ready summary checklist

* SOD: runbook + email + incident snapshot + triage pass
* Day: queue hygiene every few hours + reroute discipline + escalate on timer
* EOD: handoff note + follow-up tasks for recurring gaps
* Enforce required incident fields so the system doesn’t rely on hero memory
