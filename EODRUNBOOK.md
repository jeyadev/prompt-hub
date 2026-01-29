Sure. Humans love pretending “end of day” means the system will magically behave overnight. It won’t. So the EOD runbook’s job is to make tomorrow’s on-call **not hate today’s on-call**.

Below is a **ready-to-paste Confluence doc** for **End of Day (EOD) On-Call**.

---

## Confluence Page: End of Day (EOD) On-Call Procedure

### Page title

**On-Call End of Day (EOD) Runbook**

### Purpose

Ensure a clean, unambiguous operational handoff by:

* documenting current system risk,
* summarizing incident/ticket activity,
* transferring ownership of open work clearly,
* capturing follow-ups to reduce repeat toil.

### Scope

Applies to: **Primary On-Call Engineer** (end of business day).
Also used for **shift handoffs** (weekends, follow-the-sun, etc.).

### Inputs

* Incident queue: **[LINK: Incident Queue / ServiceNow / Jira filter]**
* Alerting dashboards (optional): **[LINK: Dynatrace / Grafana / Splunk]**
* SOD email (for reference): **[LINK: Today’s SOD email or thread]**
* On-call responsibilities (general): **[LINK: On-Call Engineers Responsibilities]**
* Incident comms channel / war-room template: **[LINK]**

---

## EOD Timebox

**Target duration:** 10–15 minutes
**Hard stop:** 25 minutes
If you’re running beyond this, you either have an active incident or a messy queue. Both require escalation, not extra typing.

---

## EOD Checklist

### 1) Confirm Handoff Target (1 min)

* ✅ Identify **next on-call owner** (person/team/timezone): **[rota link]**
* ✅ If next on-call is unavailable, confirm backup owner and post in channel.

---

### 2) Incident Queue Scrub (5–8 mins)

Open the incident list:

* **Incident Queue:** **[LINK]**

#### 2.1 Every open incident must have (mandatory)

For each open incident, confirm/update:

* **Category (Primary + Secondary)**
* **Severity**
* **Current owner** (person/team)
* **Current status** (investigating / mitigated / monitoring / waiting / blocked)
* **Next action** (explicit, not “checking”)
* **Evidence link(s)** (dashboards/log queries)
* **ETA / checkpoint** (next review time, even if rough)

#### 2.2 “No rot” rules (EOD enforcement)

* No incident should remain **New/Unassigned** at EOD.
* Any incident older than **24h** must include:

  * reason it’s still open,
  * what’s blocking it,
  * escalation status (who has been contacted).

#### 2.3 Reroutes must be clean

If you rerouted today:

* ✅ verify it landed with the right team
* ✅ ensure there’s a clear ask + evidence
* ✅ if bounced back, fix ownership ambiguity now (don’t gift-wrap confusion for tomorrow)

---

### 3) Active Incident Handling (if applicable)

If there is an **active incident** at EOD:

* ✅ confirm incident roles (IC/Ops/Comms) or assign temporarily
* ✅ update the live incident doc
* ✅ explicitly state who holds the baton after handoff

**If no active incident:** state that explicitly in handoff.

---

## Daily Metrics to Report (EOD snapshot)

Capture:

* **Opened today:** #
* **Resolved/Closed today:** #
* **Net open change since SOD:** +/-
* **Current open count:** #
* **Oldest open incident age:** duration (+ link)
* **P1/P2 count currently open:** #

Optional (recommended):

* **Reopened incidents today:** #
* **Rerouted today:** # (accepted vs bounced)
* **Top recurring category today:** (e.g., Dependency/Latency)

---

## Follow-ups and Toil Harvesting (2–4 mins)

Create follow-up items for anything that will otherwise repeat:

* ✅ runbook gap found
* ✅ noisy/non-actionable alert
* ✅ repeated manual fix
* ✅ missing dashboards/log access
* ✅ dependency instability / ownership ambiguity

**Minimum standard:** at least *one* follow-up if any recurring pattern was seen.

Follow-up tracker location:

* **[LINK: Jira board / backlog / “Reliability Improvements” epic]**

---

## Post Handoff Note (Template)

**Post in:** **[LINK: on-call channel]** and/or send email if your process uses email.

**Title:** `EOD On-Call Handoff – <DATE> | Open: <#> | P1/P2: <#> | Oldest: <DURATION>`

**Body:**

```
EOD summary (<DATE>):

Incident metrics:
- Opened today: <#>
- Closed today: <#>
- Current open count: <#> (P1/P2: <#>)
- Oldest open: <DURATION> (<INCIDENT LINK>)

Open incidents requiring attention:
1) <INCIDENT ID/LINK> | Sev <Px> | Status: <...> | Owner: <...>
   - What we know: <1–2 lines>
   - What we tried: <1–2 lines>
   - Next action: <explicit next step>
   - Evidence: <dashboard/log link>
   - Risk/ETA: <...>

2) ...

Reroutes / dependencies:
- Rerouted: <#> (links)
- Blocked waiting on: <team/person> (reason + link)

Risks for tomorrow:
- <deployments/maintenance/dependency instability/known alert noise>

Follow-ups created today:
- <Jira links> (runbook/alert/toil fixes)

Handoff owner:
- Next on-call: <NAME> (confirmed) | Backup: <NAME>
```

---

## Escalation Guidance (EOD)

Escalate before leaving if:

* P1/P2 incident is active without clear ownership
* incident is “mitigated” but not stable (risk of regression)
* queue contains unknown/untriaged items
* repeated dependency issues are emerging (multiple related tickets)

Escalation paths:

* **[LINK: escalation rota / pager]**
* **[LINK: app team on-call]**
* **[LINK: incident commander process]**

---

## Definition of Done for EOD

EOD is complete only when:

* ✅ all open incidents have owner + next action + evidence link
* ✅ no New/Unassigned remains in the queue
* ✅ handoff note posted (with metrics + open incident summary)
* ✅ follow-ups captured for recurring toil/noisy alerts
* ✅ next on-call owner confirmed

---

## Appendix

* Live incident doc template: **[LINK]**
* Incident category taxonomy: **[LINK]**
* Reliability backlog / follow-up epic: **[LINK]**
* On-call responsibilities (reference): **[LINK]**

---

This EOD runbook is the difference between “we have an on-call process” and “we have a ritual where we pretend not to drop balls.”
