L1/L2 Support Automation Framework

Foundation Vision, Principles, and Implementation Strategy (SRE + ITIL + Design Thinking)


---

1. Why This Exists

Our workflow builder can already read from endpoints, scrape webpages, and query Splunk logs. The real limiter is not tooling. It’s how consistently we can translate recurring L1/L2 work into high-quality workflows that are safe, trusted, and widely adopted.

This document formalizes the vision and mental model for building an enterprise-grade L1/L2 Support Automation capability that is:

SRE-aligned: reduces toil and improves reliability outcomes

ITIL-aligned: consistent, auditable, and safe within incident/change governance

Human-centered: reduces cognitive load and earns trust under real incident pressure



---

2. Our North Star

> Every recurring L1/L2 action should either be eliminated at the source or reduced to a one-click, explainable, auditable workflow inside ServiceNow.



This is not “automation for automation’s sake”. It is a system to progressively externalize institutional support knowledge into deterministic workflows.


---

3. Problem Statement

L1/L2 support time is consistently spent on work that is:

Repetitive diagnostics (logs, dashboards, basic checks)

Data retrieval for consumers/stakeholders

Verification/health checks

Communication/status updates


This work scales linearly with incident volume and does not build long-term reliability. It creates avoidable operational drag and reduces capacity for prevention, hardening, and engineering improvements.


---

4. Definition: What We Automate vs What We Don’t

In-Scope (initially)

We target work that is predominantly toil:

1. Diagnostic toil

Fetch logs (Splunk), summarize error patterns

Check tracing/problems (Dynatrace)

Basic endpoint checks (read-only)



2. Information requests

“Is X failing?”, “Can you share logs?”, “What’s the status?”



3. Verification work

Post-deploy checks, routine validations



4. Communication toil

Standard summaries and templated updates into ServiceNow/Teams/email




Out-of-Scope (initially)

We avoid anything requiring high judgment or disruptive actions:

Deep root cause analysis and investigative engineering work

Automated remediation involving disruptive actions (restart/scale/config changes)

Any action requiring change-management approval unless explicitly gated



---

5. Guiding Principles (SRE + ITIL + Design Thinking)

5.1 SRE Principles (Reliability Economics)

Toil reduction is the goal, not number of workflows.

Prefer eliminating the need for work over automating it.

Start with workflows that reduce repeated manual diagnostics and improve incident response consistency.

Keep workflows small, composable, and measurable.


5.2 ITIL Principles (Service Operations Discipline)

ServiceNow is the system of record.

Automation must be consistent, traceable, auditable.

Respect incident/request/problem separation and governance.

No disruptive automation without appropriate change/incident references and explicit guardrails.


5.3 Design Thinking Principles (Human Reality)

L1/L2 engineers are the primary users; incidents are high-stress contexts.

Automation must reduce cognitive load, not add noise.

Outputs must be:

Explainable (“what was checked and why”)

Actionable (“what to do next”)

Credible (show evidence, not just conclusions)


Trust is earned through transparency and safe failure modes.



---

6. Non-Negotiable Guardrails (Phase 1)

Automation starts as Assist Mode only

Workflows can:

Gather data

Summarize findings

Link to runbooks

Add ServiceNow comments/notes

Populate metadata fields


Workflows cannot (initially):

Change incident priority, state, or assignment automatically

Execute disruptive actions

Close incidents automatically


Every workflow must:

Leave an auditable trail in ServiceNow (what it ran, what it found)

Link to the supporting runbook/documentation

Fail safely and visibly (errors are reported, not hidden)



---

7. What “Good Automation” Looks Like (Quality Bar)

A workflow is considered “good” if it meets most of the following:

Saves ≥ 5–10 minutes per typical execution

Correct/valuable in ≥ 80% of common cases (happy path)

Produces a human-readable summary:

What it checked

What it found

What it suggests next


Includes evidence:

Splunk query references/results

Dynatrace problem/traces references

Endpoint responses (redacted/safe)


Does not create noise or ambiguous recommendations


If a workflow does not meet this bar, it should be refined or not deployed.


---

8. Implementation Strategy (Enterprise SRE Mental Model)

Phase 0 — Align on Constraints & Success Measures

Publish this vision + guardrails

Align stakeholders on “assist-only” start

Define success metrics:

Toil hours reduced

Time-to-first-diagnostic improvement

Workflow adoption rate

Incident handling consistency



Phase 1 — Codify Repeatable L1/L2 Work (Read-only)

Focus: diagnostics + enrichment + communication templates.

Convert recurring tasks into Toil Cards (standard format)

Build workflows that:

Run standard Splunk patterns (e.g., index=am_cds)

Pull Dynatrace problems/traces context

Perform safe endpoint checks

Post structured output into ServiceNow



Phase 2 — Standardize Building Blocks

Create reusable blocks:

“Top error signatures last 15m”

“5xx spike breakdown”

“Dynatrace problem context”

“Health endpoint validation”


Workflows become compositions of blocks rather than one-off logic.

Phase 3 — Controlled Semi-Automation (Future)

Only after trust and maturity:

Add optional human approvals (UI action/button)

Add gated actions with references:

Change # or P1/P2 incident #


Introduce progressive promotion:

Assist → Semi-auto → (Selective) Auto-resolve for low-risk tasks




---

9. Workflow Governance (Treat Workflows Like Products)

Every workflow must have:

Owner (team/person accountable)

Versioning (changes tracked)

Runbook linkage (source of truth for intent)

Risk classification

Read-only / low risk / disruptive (later)


Retirement criteria

If obsolete, inaccurate, or unused



This prevents workflow sprawl and ensures long-term maintainability.


---

10. Decision Checklist (Used During Workflow Review)

Before approving a workflow, ask:

1. Is this truly toil (repetitive, mechanical) vs actual investigation?


2. Does this work exist due to a product gap we should fix instead?


3. Can we define an 80% happy path without complex branching?


4. If it’s wrong, does it fail safely and visibly?


5. Would we trust it at 3 a.m. during a live incident?


6. Does it improve reliability outcomes or reduce operational load measurably?



If it fails multiple checks, it should not be deployed as-is.


---

11. Summary

This framework formalizes how we build L1/L2 support automation without creating:

unsafe auto-remediation

noisy “automation spam”

opaque black-box behavior

governance/security friction


We will start with assistive, auditable, human-centered automation, focus on high-yield toil, and progressively mature the system based on evidence and adoption.


---

12. Appendix: Terms

Toil: Manual, repetitive, automatable, tactical work that does not provide enduring value and scales linearly with service growth.

Assist Mode: Automation gathers and summarizes evidence, updates tickets, and recommends next steps without taking disruptive actions.

Toil Card: Standard template describing a recurring task (trigger, steps, tools, inputs/outputs) that can be converted into a workflow.


---