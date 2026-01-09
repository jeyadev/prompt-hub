Here’s a ready-to-paste JIRA task for Aayushi with clear acceptance criteria and a step breakdown that guides her from manual steps to an automated workflow under SRE-Delta.


---

Title: Create SRE-Delta Workflow for “Upload Failed” Diagnostic

Description:
Build an initial SRE-Delta workflow that automatically runs when a user reports an upload failure. This workflow should codify the manual troubleshooting steps typically performed by L1/L2 engineers into an automated sequence using SRE-Delta capabilities (Splunk queries, endpoint checks, evidence summary). The workflow will produce a diagnostic summary that can be used in triage, evidence-backed escalation, or self-serve guidance.


---

Assignee: Aayushi
Reporter: Your Name
Priority: Medium → High (foundation for CDS triage product)
Labels: sre-delta, workflow, cds, upload-failure


---

Tasks / Sub-Tasks

1) Define the Manual Troubleshooting Runbook

Document the “happy path” manual steps engineers take when investigating an upload failure.

Identify which Splunk queries are typically run (e.g., errors in index=am_cds filtered by upload action + tenant).

Identify relevant endpoint health checks (status of upload API → 200/timeout/503 patterns).

Identify downstream dependencies (e.g., object store, metadata service).

Identify any Dynatrace problem selectors commonly used.


Clarify what constitutes an upload failure (error codes/messages, timeouts, 4xx/5xx patterns).

Document all tools involved: Splunk, ServiceNow, internal APIs, runbooks.


Acceptance Criteria:

A detailed runbook document with steps numbered, expected outputs at each step, sample queries, and expected normal vs abnormal signatures.



---

2) Design the Workflow Logic

Based on the manual runbook, sketch the workflow logic:

Required inputs (tenant, category, time range)

Branching conditions (e.g., error present vs no error, downstream unavailable)

Data sources (Splunk query templates, endpoint URLs, Dynatrace)

Output schema (summary text, key values, evidence artifacts)


Produce a workflow specification (pseudocode or visual flow) that represents the step sequence.


Acceptance Criteria:

A workflow design document that maps manual steps → workflow blocks.

Clear definition of inputs, outputs, decision points.



---

3) Implement the Workflow in SRE-Delta

Create a new workflow in the SRE-Delta builder, named something like:
cds_upload_failure_diag_v1

Add steps to:

Extract Inputs: tenant, action timestamp, category (from payload or defaults)

Run Splunk Queries: Errors in index=am_cds for upload failures

Run Endpoint Checks: Hit the upload API with a health check path

Optional: Call Dynatrace to gather problem context for the relevant service(s)

Aggregate Results: Summarize top error signatures, response codes, and dependencies status

Produce Diagnostic Summary: A human-readable block of text with evidence links and key values



Acceptance Criteria:

Workflow is defined with all techniques above.

Variables are parameterized (tenant, category, time range).



---

4) Test & Validate the Workflow

Run the workflow manually against known upload failure cases with real context.

Validate that:

Splunk queries return expected data (no syntax errors)

Endpoint checks behave as expected

Summary output is human-readable and actionable


Log any unexpected behaviors and adjust workflows accordingly.


Acceptance Criteria:

Test results documented

Screenshots or logs saved in Confluence/JIRA

A measurable pass/fail matrix for expected patterns



---

5) Create a ServiceNow Assist Template

Create a standard diagnostic summary template for ServiceNow comments or escalation packages.

Template should include:

Summary of findings (top errors, status codes, dependencies)

Evidence elements (Splunk search links, sample results)

Suggested next steps (retry, escalate, ticket creation, runbook section links)



Acceptance Criteria:

A reusable template in Confluence

Test integration if possible (post output into an SN ticket)



---

6) Documentation & Hand-Off

Document the workflow logic, inputs/outputs, known limitations, and how to maintain/extend the workflow.

Update the SRE-Delta workflows index/catalog with this artifact.

Conduct a short knowledge transfer demo with the team.


Acceptance Criteria:

Confluence page updated

Demo session scheduled & recorded

Colleague acknowledged understanding



---

Definition of Done

Workflow exists in SRE-Delta and runs without errors

Manual test cases pass (outputs align with expected diagnostic signatures)

Confluence documentation is complete and linked

Assist template available

Team demo completed



---

Notes

Aim for assist-only mode initially (don’t perform disruptive actions).

Keep Splunk queries efficient (document limits/time ranges).

We’ll iterate to add severity classification & escalation routing later.



---

If you want, I can also generate a Confluence page template for this task’s documentation or a workflow specification document that Aayushi can fill in as she goes.