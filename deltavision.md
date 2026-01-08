You’re right: **“prefilled ServiceNow incident”** sounds like we’re bragging about autofill. Executives don’t fund autofill.

Use language like:

* **“Evidence-backed incident creation”**
* **“Context-rich escalation package”**
* **“Auto-generated incident dossier”**
* **“Structured escalation with diagnostics attached”**

I’d use **“Incident Dossier”** internally and **“Evidence-backed escalation”** with leadership.

Below is a Confluence-ready doc you can paste and walk your director through.

---

# SRE-Delta L1/L2 Automation

## Vision Proposal: Incident Triage Product for CDS and Document Solutions

---

## 1. Executive Summary

CDS is the API gateway powering multiple document solutions (Doc Manager, Docsuite, etc.), and a large portion of our operational load originates from these consumer-facing applications. Today, support relies heavily on manual triage: collecting tenant/category context, running Splunk queries, checking traces, and writing ticket summaries.

We propose building an **Incident Triage Product** powered by **SRE-Delta** (our workflow builder) to shift triage left, standardize first response, and reduce L1/L2 toil.

**What’s new:** we embed triage directly into the consumer apps so we can automatically capture rich context (tenant/category/action/request metadata), run deterministic diagnostics workflows, and generate an **evidence-backed escalation package** when an incident is required.

---

## 2. Problem Statement

### Current pain

* L1/L2 triage is repetitive and variable:

  * collect context from users (tenant, category, action, timestamp)
  * run Splunk queries (`index=am_cds`) and interpret patterns
  * check Dynatrace for related problems/traces
  * manually craft ServiceNow updates
* Consumer support channels (email/Teams) increase unstructured intake.
* Ticket quality varies; resolution time depends on who is on-call.

### Impact

* High operational toil (manual, repetitive, scalable with incident volume)
* Slower time-to-first-diagnostic
* Higher back-and-forth with consumers
* Less time for prevention and reliability improvements

---

## 3. Proposed Vision

### North Star

> **Every recurring L1/L2 support action should be eliminated at the source or reduced to a one-click, explainable, auditable workflow inside our support processes.**

### Product framing

We are not building “a chatbot”.
We are building an **Incident Triage Product** with a chatbot as one interaction mode.

**Core capability:**
Embedded triage in Doc Manager / Docsuite that:

1. captures context automatically
2. runs SRE-Delta diagnostic workflows
3. presents evidence-backed guidance
4. enables **evidence-backed escalation** into ServiceNow when needed

---

## 4. Key Differentiator: Context-Aware Triage

Because triage is embedded inside Doc Manager / Docsuite, the host application already knows critical context such as:

* tenant
* document category/type
* action (upload/download/search)
* timestamps
* error codes/messages
* request/correlation IDs (where available)

This context can be passed directly into workflows, reducing ambiguity and increasing diagnostic precision.

---

## 5. User Experience (Target Flow)

### Entry points (multiple “front doors”)

* Embedded triage UI in Doc Manager / Docsuite (primary)
* Teams (bot/command entry)
* Email (intake-assisted, later phase)
* ServiceNow UI action (“Run diagnostics”, later or parallel)

### Embedded triage flow

1. **Guided intent selection** (preselected based on app screen)

   * “Upload failing”
   * “Upload slow”
   * “Document not found”
   * “Access denied”
   * “Check status / logs”
2. **Run diagnostics** (one-click)

   * Executes workflow bundle (Splunk + Dynatrace + endpoint checks)
3. **Outcome**

   * Guidance/next steps (if resolvable)
   * If escalation required: generate an **Incident Dossier** (context + diagnostics + evidence) and create an escalation into ServiceNow

---

## 6. Technical Concept (High-Level)

### Architecture pattern

* **Host App (Doc Manager / Docsuite)** passes a standardized **Context Payload** to the embedded triage UI.
* Triage UI routes to the correct workflow bundle.
* SRE-Delta executes deterministic diagnostics:

  * Splunk (`index=am_cds`) query templates filtered using payload context
  * Dynatrace problem/tracing context (where relevant)
  * Endpoint checks (via allowlisted IP + IDAnywhere FID, where applicable)
* Outputs:

  * human-friendly summary
  * evidence links/snippets
  * recommended next step / runbook link
  * incident dossier for escalation

---

## 7. The Context Payload Contract (Key Foundation)

We will define a versioned contract shared across embedding applications.

### Minimum payload (v1)

* app (`doc-manager`, `docsuite`)
* environment (`prod`, `uat`)
* action (`upload`, `download`, `search`, `distribute`)
* tenant_id / tenant_name
* category / document_type
* timestamp (or time range)
* error code/message (if available)
* request_id / correlation_id (if available)

**Principle:** structured context first; chat/free-text second.

---

## 8. Workflow Strategy: Bundles and Building Blocks

To avoid workflow sprawl, we will build reusable building blocks and compose them into “workflow bundles”.

### Examples of building blocks

* Splunk: “Top error signatures (last 15m)”
* Splunk: “5xx breakdown by endpoint/tenant”
* Dynatrace: “Relevant open problems for service”
* Endpoint: “Health/readiness check”

### Example bundle

**UploadFailureBundle**

* Splunk errors filtered by tenant/category/action
* Identify dominant error signature
* Link to runbook section
* Generate summary + evidence package

This approach scales across services without one-off logic for every case.

---

## 9. Guardrails (Enterprise Safety)

### Phase 1 = Assist-only

* No disruptive actions
* No automatic state changes in ServiceNow
* Always attach evidence and explain checks performed
* Workflows accessible only to authenticated users (IDAnywhere SSO)

Any disruptive automation (restart/scale/config actions) is explicitly out-of-scope initially and would require governance and change references.

---

## 10. Success Metrics (What We Will Measure)

* **Time-to-first-diagnostic** reduction (median)
* **% of escalations created with Incident Dossier attached**
* **Reduction in L1 manual Splunk query time**
* **Deflection rate** (issues resolved without escalation where appropriate)
* Workflow usage rate by entry point (embedded vs Teams vs SN)

---

## 11. Phased Rollout (Kickstart Plan)

### Phase 1 (Pilot) – Prove value quickly

* Target 1 host app (Doc Manager) + 1–2 high-volume intents (e.g., upload failure)
* Build 3–5 workflows in assist mode
* Deliver incident dossier + evidence-backed escalation into ServiceNow
* Measure improvements and adoption

### Phase 2 – Expand coverage

* Add more intents and CDS services (cds-store/find/distribute)
* Add Teams entry point for internal users
* Standardize building blocks

### Phase 3 – Ingest email/Teams requests more broadly (optional)

* Intake normalization into guided triage
* Automated evidence gathering before escalation

---

## 12. Decisions Needed to Start

* Confirm pilot scope:

  * Which host app first (Doc Manager vs Docsuite)
  * Which 2 intents to target first
* Confirm ownership and operating model:

  * Workflow owners per service
  * Review and promotion process (assist → later phases)
* Confirm integration approach:

  * Context payload contract ownership
  * ServiceNow escalation format for dossier attachment

---

## 13. Appendix: Terminology

* **Incident Triage Product:** Embedded experience that guides initial triage and runs diagnostics workflows.
* **Incident Dossier:** Structured package containing context, diagnostics outputs, evidence, and next steps for escalation.
* **Evidence-backed escalation:** Escalation into ServiceNow where the ticket is created with the dossier attached, reducing back-and-forth.

---

If you present this to your director, the narrative is strong because it’s not “we built a bot.” It’s **we built a triage product** that reduces operational cost, increases consistency, and improves response speed.

If you want an even more executive-friendly version, strip sections 6–8 into an appendix and keep the pitch to: *problem → vision → pilot → metrics → ask*.
