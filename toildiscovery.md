Nice, we’re zooming into the human side first. Let’s design a one-time “toil harvest” that you can run across teams and then feed into your more systematic machinery later.

I’ll keep it tight and practical.


---

1. Overall Shape

One-time discovery = 3 passes over the same problem:

1. Structured pre-survey (individual thinking with prompts)


2. Team workshop (group consolidation + de-dup + ranking)


3. Normalization into a standard “Toil Card” per use case



Everything should be designed so it’s trivially convertible into a workflow later.


---

2. Pre-Survey: What each engineer fills

Send this to every L1/L2 + key L2/L3 folks who touch incidents.

Section A – Identity & scope

Name

Team / Service(s) you support (e.g., cds-store, cds-find, cds-distribute)

Primary role in incidents (L1/L2/L3)


Section B – Top 5 painful repetitive tasks

For each of up to 5 tasks:

1. Task name
– “Investigate CDS 5xx spike,” “Provide error logs for consumer,” “Check nightly job status,” etc.


2. When does this happen?
– Trigger: “P2/P3 incident”, “consumer email”, “daily check”, “post-deploy check”


3. How often last month?
– 1–2 / 3–5 / >5 / >10


4. Avg time spent each time?
– 0–5m / 5–15m / 15–30m / >30m


5. What do you actually DO? (short procedural list)

Example:

1. Run Splunk query X on index=am_cds


2. Filter by service=cds-store & env=prod


3. Count top error messages


4. Check Dynatrace for active problems on cds-store


5. Paste summary into ServiceNow comment





6. What tools are involved?
– Splunk / Dynatrace / internal API / UI / ServiceNow only / other


7. How much “thinking” is needed?
– Low (follow runbook blindly) / Medium (some judgment) / High (deep debugging)


8. Does a runbook exist?
– Y/N + link if Y


9. Could you hand this to a new hire with a checklist?
– Yes / Partially / No



This form forces them to give you repeatability + rough time + steps + tools.


---

3. Toil Qualification Rubric (inline in the form)

At the end of each task in the survey, ask them to answer quickly:

This task is:

[ ] Manual

[ ] Repetitive (≥ 3 times last month)

[ ] Mostly mechanical (low thinking)

[ ] Could be done using Splunk/Dynatrace/HTTP + simple rules


Toil score (gut feel): 1–5
– 1 = “once in a while, interesting”
– 5 = “soul-crushingly repetitive”


You don’t need perfect accuracy; you just need relative signals from the people doing the work.


---

4. Team Workshop: Turn survey noise into a ranked list

Run a 60–90 minute session per group (e.g., CDS support).

Step 1 – Prep (done by you/lead)

Before the meeting:

Export survey responses.

Group tasks by:

Service (cds-store, cds-find, …)

Trigger (incident vs user request vs health-check)


Identify clusters that are clearly the same thing with slightly different wording.


Bring a draft list of 10–20 candidate toil items to the workshop.

Step 2 – In the session

For each candidate cluster:

On a shared doc/board, write:

Name: “CDS 5xx auto-diag for incidents”

Trigger: “P2/P3 incident with cds-* service + 5xx in description”

Current manual steps: bullet list from surveys

Tools: Splunk / Dynatrace / API / SN

Rough stats: median frequency, time/occurrence

Toil score (team vote): 1–5


Then ask the group:

1. Is this description accurate? Anything missing from steps?


2. Is this really repetitive, or are there many edge cases?


3. What’s the “happy path” that happens in 80% of cases?


4. Is there a runbook that already describes this? Which one?



Update each candidate live.

Step 3 – Prioritize live

Once refined, have the team rank all items using 2 dimensions:

Impact (1–5): time saved × how often it happens

Automatability (1–5):
– Data is in Splunk/Dynatrace/APIs, steps are rule-based


Plot quickly in a table:

Task	Impact (1–5)	Automatability (1–5)	Priority Score (IxA)

CDS 5xx auto-diag	5	5	25
Consumer log dump req	4	5	20
Nightly job status check	3	5	15


Top-right corner = first automation batch.


---

5. Normalize into a “Toil Card” per use case

Right after the workshop (same day if possible), convert each high-priority item into a standard Toil Card (this is the artifact your workflow platform will eventually consume).

Fields:

Name

Owner team / contact

Trigger
– “New ServiceNow incident with business_service in cds-* and 5xx in short_description”

Frequency / avg time (from survey/workshop)

Tools involved
– Splunk (index=am_cds SPL template), Dynatrace, internal endpoint URLs if known

“Happy path” step list (no branches, just dominant path)

Runbook link(s)

Toil score, Impact, Automatability

Risk constraints
– Read-only only / no config changes / no restarts


You now have a clean list of 10–30 Toil Cards per domain, all sourced from actual experience, deduped and prioritized.


---

6. Minimal Ops Checklist for this one-time discovery

[ ] Create & circulate the individual survey (with 5 tasks per person).

[ ] Collect responses across all L1/L2 supporting CDS.

[ ] Pre-group responses into candidate toil clusters.

[ ] Run 60–90 min workshop per team to:

Validate steps

Identify “happy path”

Score impact & automatability


[ ] Convert top items into Toil Cards with full step lists + tools + triggers.

[ ] Park Toil Cards in a central repo (Confluence page or JIRA board) as the initial automation backlog.


That gives you a solid, one-time “brain dump of toil” that’s structured enough to drive your first serious automation wave.