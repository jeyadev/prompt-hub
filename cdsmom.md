Hi all,

Sharing the Minutes of Meeting and agreed actions from today’s discussion on reinstating the CDS SOD health check cadence and piloting incident-queue hygiene.

**Discussion summary**

1. Walked through the proposed **SOD email update** to include an **incident snapshot**: open incidents + incident counts (previous business day vs current day) + queue actions taken (categorized / assigned / resolved / rerouted). The supporting **SOD runbook** will be shared as part of this MoM.
2. Concern raised: CDS being **bombarded with misdirected incidents**. Agreed that the new snapshot will capture **rerouted ticket counts** to quantify misrouting and enable data-backed escalation to L1/L2 / relevant owning teams.
3. Concern raised: **data requests** that can be handled by **L2 ASD** without routing into CDS. Agreed we need **data** over the next week to identify and correct this pattern.

**Agreements / Decisions**
4) **1-week monitoring/audit** of the CDS incident queue to identify patterns (misrouting, data-request noise, ownership gaps). In parallel, we will clean up the current backlog of ~10 open incidents by ensuring each has clear notes, a CDS owner where applicable, and is either **resolved or rerouted**.
5) Operational approach (interim): While the proposed model assigns SOD triage to the **on-call engineer**, **Ashish** will **own daily monitoring and triage of the CDS incident queue** for now, until the team is comfortable adopting this as a standard on-call responsibility within the rota.
6) Next touchpoint: **9th Feb** to review findings from the 1-week audit and decide whether to **formally incorporate the incident snapshot into the SOD email** going forward.
7) Reinforcement: The team will **restart the SOD health check email**, which has been discontinued for a period, and use the current template during the pilot week.

**Action items (this week)**

* Ashish: Monitor CDS incident queue daily; ensure new/untriaged items are assigned, resolved, or rerouted with notes and evidence.
* CDS team: Update the existing ~10 open incidents with relevant notes, assign to a CDS engineer where applicable, and resolve/reroute as appropriate.
* [Your Name]: Share the SOD runbook link in this thread; compile a simple summary of queue patterns observed (misrouted incidents, data requests, reroute counts) for review on 9 Feb.

Thanks,
[Your Name]
