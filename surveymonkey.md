Alright, foundation laid. Now we extract signal without exhausting goodwill.

Below is a lean, high-signal SurveyMonkey questionnaire optimized for:

minimal fatigue (10–12 minutes max),

easy aggregation,

direct conversion into workflows later.


No fluff. No philosophy. Just usable data.


---

CDS L1/L2 Toil Discovery Survey

Purpose: Identify repetitive, automatable work based on real support experience.

> Please think about work you personally do repeatedly. We’re not looking for “ideas”, we’re looking for reality.




---

Section 1: Context

Q1. Your role
Type: Single choice

L1 (Frontline triage)

L2 (Specialist / deep dive)

L3 / SME

Other


Q2. Services you mainly support
Type: Multi-select

cds-store

cds-find

cds-distribute

Other CDS / Seal services

Other



---

Section 2: Repetitive Task #1 (Required)

> Think of one task you’ve done multiple times recently that felt repetitive.



Q3. Short name for the task
Type: Single-line text
Example: “CDS 5xx incident diagnostics”, “Consumer log extraction”

Q4. When does this task usually occur?
Type: Single choice

During incidents (P1–P3)

During lower-priority tickets

On consumer / internal user request

Routine health / verification checks

Post-deployment

Other


Q5. How often did you do this in the last month?
Type: Single choice

1–2 times

3–5 times

6–10 times

More than 10 times


Q6. On average, how long does it take each time?
Type: Single choice

< 5 minutes

5–15 minutes

15–30 minutes

30–60 minutes

> 60 minutes




Q7. Which tools are involved?
Type: Multi-select

ServiceNow

Splunk (index=am_cds)

Dynatrace

Internal HTTP APIs

Internal web UI

Confluence / Engineers Portal

Email / chat only

Other


Q8. Describe the usual steps you follow (happy path)
Type: Long text
Keep it concrete. Bullet or numbered steps preferred.

> Example:

1. Open ServiceNow incident


2. Run Splunk query on am_cds for service


3. Check top error patterns


4. Look for Dynatrace problems


5. Update SN with summary





Q9. Does a runbook exist for this task?
Type: Single choice

Yes

No

Not sure


Q10. If yes, paste the runbook link (optional)
Type: URL

Q11. How much judgment does this task require?
Type: Single choice

Low – I mostly follow a checklist

Medium – pattern-based with some thinking

High – real investigation each time


Q12. Could a new hire do this with a checklist?
Type: Single choice

Yes

Partially

No


Q13. Which of these describe this task?
Type: Multi-select

Manual

Repetitive

Mostly mechanical

Data already exists in tools (Splunk/Dynatrace/APIs)

Grows as incident volume grows


Q14. Overall, how much does this feel like toil?
Type: Rating (1–5)

1 = not really toil

3 = mixed

5 = pure toil



---

Section 3: Repetitive Task #2 (Optional)

> If you have another task that fits the above definition, answer the same questions.



Repeat Q3–Q14 as Task #2.
(Add a note: Skip if none.)


---

Section 4: Final Catch-All

Q15. Is there any other recurring work you strongly believe should be automated, even if you didn’t detail it above?
Type: Long text (optional)


---

Why this works (so you can defend it)

Forces specific tasks, not vague complaints

Captures:

frequency × time (impact),

steps (workflow blueprint),

tools (feasibility),

judgment (automation boundary)


Easy to:

cluster,

score,

convert into Toil Cards.



No one can say “we didn’t know” after this. And no one can hide behind abstractions either.

Send this. Collect it. Then we turn pain into systems.