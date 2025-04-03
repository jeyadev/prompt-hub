# Incident Classification Guide

This page outlines the standards and criteria for incident classification, ensuring clear understanding and consistent management of incidents impacting payment systems and consumer services.

## Incident Priority Levels

| Priority | Analogy (For easy recall) | Description | Service Level Objective (SLO) |
|----------|---------------------------|-------------|------------------------------|
| **P1** *(Critical)* | **"ATM Spits No Cash"** | Complete outage or critical functionality loss. Immediate customer/business impact. | Immediately addressed according to Severity (see severity matrix). |
| **P2** *(High)* | **"Credit Card Declined Unexpectedly"** | Moderate flaw; service degradation or partial functionality loss with moderate user/business impact. | 75% within 6 hours |
| **P3** *(Moderate)* | **"Slow Mobile Banking App"** | Partial non-critical functionality loss; minor impact with simple workarounds available. | 75% within 12 hours |
| **P4** *(Low)* | **"Incorrect Transaction Description"** | Minimal user impact; cosmetic or minor issue with limited business effect. | 75% within 240 hours |
| **P5** *(Informational)* | **"Routine Maintenance Notice"** | No immediate impact; informational or routine updates. | 75% within 240 hours |

## Detailed Definitions

### Priority 1 (P1) – Critical Incidents

Priority 1 incidents are further classified based on severity levels. Refer to the detailed Severity Matrix below for escalation and immediate response procedures.

| **Severity** | **Analogy**                              | **Explanation**                                                       | **Impact Example**                                             |
|--------------|------------------------------------------|-----------------------------------------------------------------------|-----------------------------------------------------------------|
| 🔴 **Severity 1** *(Highest)* | **"Entire Bank Offline"**                | Complete global outage or major critical services impacting all customers. Immediate escalation required. | Entire payment network offline globally for multiple hours. |
| 🔶 **Severity 2**              | **"Major Region Payments Failure"** | Critical functionality lost affecting major customer segments or multiple essential services with no immediate workaround. Significant business and brand risk. | Payment failures across an entire geographic region (e.g., Europe-wide outage). |
| 🟡 **Severity 3**              | **"Single Major Service Unavailable"** | Essential service or critical functionality unavailable; significant immediate impact, potential broader impact if unresolved. | International wire transfers or specific high-volume payment service down. |
| 🟢 **Severity 4** *(Lowest)*    | **"High-Value Transactions Delay"** | Essential services experiencing degraded performance or partial functionality loss, manageable short-term impact but heightened risk if prolonged. | Significant delays in high-value or regulatory-bound payments. |

### Priority 2 (P2) – High Incidents

The system functions but is moderately flawed. Potential risk of escalating if not resolved. Criteria include:

- Non-critical features unavailable.
- Moderately impaired technology capacity.
- Resilience with increased risk to availability.
- Initial monitoring event or degraded performance observed.
- Moderate user disruption with available workaround.
- Immediate risk to production installation milestones.
- Cyber event with limited controls, moderate risk of exploitation.

> **Action:** Must be prioritized and addressed continuously until resolved.

### Priority 3 (P3) – Moderate Incidents

Partial non-critical functionality loss with user-available workarounds:

- Minor technical/application issue, minimal user disruption.
- Issues within specific non-production or lower RTO environments.
- Moderate risk to medium-term milestones.

> **Action:** Addressed after higher-priority incidents; moderate urgency.

### Priority 4 (P4) – Low Incidents

Issues with minimal/no user impact, primarily cosmetic:

- Minor bugs not impacting functionality significantly.
- General inquiries or how-to questions.
- Requests outside established request systems.

> **Action:** Addressed in standard maintenance windows.

### Priority 5 (P5) – Informational

No immediate functional impact:

- Typically event/service desk incidents.
- Informational only, deferred action.

> **Action:** Logged, monitored, deferred until after all higher priority incidents.

## Important Notes

- **Escalation & Cadence**: Follow defined escalation paths for each priority.
- **Root Cause**: Cause codes required for Priority 1 incidents only.
- **Incident Handling**: Prioritize based on definitions, considering customer and business impact.

## Reference & Resources

- Escalation Matrix
- Categorization Matrix
- Problem Management & Root Cause Analysis
- Alert Response Guidelines

**Use this guide consistently to ensure clarity, alignment, and effective incident management practices.**

Here's a structured recommendation based on industry best practices (SRE & ITIL standards), considering ideal response times, update cadences, and other essential metrics for each incident priority. These recommendations ensure rapid mitigation, transparent communication, and continuous improvement.

---

## 📌 **Incident Response & Update Cadence Guidelines**

### 🔴 **Priority 1 (Critical)**  
**Problem Context:**  
Complete or significant outage affecting customer trust, business continuity, and potential revenue loss.

| Metric                    | Recommended Standard                            | Reasoning                                                    |
|---------------------------|-------------------------------------------------|--------------------------------------------------------------|
| **Initial Response Time** | Within **15 minutes**                           | Immediate response crucial to limit severe business impact. |
| **Subsequent Updates**    | Every **30 minutes** until resolution           | Frequent communication to all stakeholders essential for trust and rapid decision-making. |
| **MTTR Target**           | Ideally within **4 hours**                      | Rapid service restoration essential to minimize impact.     |
| **Post-Incident Report**  | Within **24 hours** post-resolution             | Timely PIR critical to capturing accurate learnings and root cause. |

---

### 🔶 **Priority 2 (High)**  
**Problem Context:**  
Partial outage or significant degradation with moderate user impact, potential escalation risk if unresolved.

| Metric                    | Recommended Standard                            | Reasoning                                                    |
|---------------------------|-------------------------------------------------|--------------------------------------------------------------|
| **Initial Response Time** | Within **30 minutes**                           | Fast response prevents potential escalation and restores partial functionalities promptly. |
| **Subsequent Updates**    | Every **2 hours** until resolution              | Balances urgency with operational practicality; ensures consistent stakeholder awareness. |
| **MTTR Target**           | Ideally within **8 hours**                      | Prompt resolution mitigates escalating user/business disruption. |
| **Post-Incident Report**  | Within **48 hours** post-resolution             | Prompt root cause analysis helps mitigate future risks.      |

---

### 🟡 **Priority 3 (Moderate)**  
**Problem Context:**  
Limited impact on non-critical functionality, workaround available, minimal user disruption.

| Metric                    | Recommended Standard                            | Reasoning                                                    |
|---------------------------|-------------------------------------------------|--------------------------------------------------------------|
| **Initial Response Time** | Within **1 hour**                               | Prompt acknowledgment reduces user anxiety; confirms awareness. |
| **Subsequent Updates**    | Every **4 hours** until resolution              | Less frequent but regular updates ensure visibility without resource strain. |
| **MTTR Target**           | Ideally within **24 hours**                     | Reasonable restoration timeframe balances business priority. |
| **Post-Incident Report**  | Within **72 hours** post-resolution             | Ensures corrective actions are documented efficiently.       |

---

### 🟢 **Priority 4 (Low)**  
**Problem Context:**  
Minimal or cosmetic issues, low customer/business impact, addressed in routine maintenance cycles.

| Metric                    | Recommended Standard                            | Reasoning                                                    |
|---------------------------|-------------------------------------------------|--------------------------------------------------------------|
| **Initial Response Time** | Within **4 hours**                              | Basic acknowledgment suffices, emphasizing operational efficiency. |
| **Subsequent Updates**    | Every **24 hours** or as status changes         | Minimal disruption; updates as necessary for transparency.   |
| **MTTR Target**           | Within scheduled maintenance window, or up to **7 days** | Low urgency allows planned resource allocation.              |
| **Post-Incident Report**  | Optional; standard monthly or quarterly review  | Addressed collectively in periodic operational reviews.      |

---

### ⚪ **Priority 5 (Informational)**  
**Problem Context:**  
Informational logs, events, no immediate functional impact.

| Metric                    | Recommended Standard                            | Reasoning                                                    |
|---------------------------|-------------------------------------------------|--------------------------------------------------------------|
| **Initial Response Time** | Within **1 business day**                       | Routine acknowledgment sufficient.                           |
| **Subsequent Updates**    | Not required unless status changes              | Minimal/no active management required.                       |
| **MTTR Target**           | No specific MTTR; handled as capacity allows    | Lowest priority; allocated as per operational convenience.   |
| **Post-Incident Report**  | Not required                                    | Typically, these do not warrant additional documentation.    |

---

## 📊 **Additional Metrics to Consider:**

Apart from standard response and update metrics, here are other vital metrics to further enhance your incident management processes:

- **Mean Time to Acknowledge (MTTA)**:
  - *Why:* Measures your team's efficiency in initial recognition of incidents. Helpful in identifying process bottlenecks at the alerting or monitoring level.

- **Mean Time to Detect (MTTD)**:
  - *Why:* Helps evaluate monitoring effectiveness, ensuring alerts are timely to minimize service impact.

- **Percentage of False Positives in Alerts**:
  - *Why:* Assesses the accuracy and relevance of alerts. High false positives can lead to alert fatigue and lowered responsiveness.

- **Incident Reoccurrence Rate**:
  - *Why:* Indicates effectiveness of Problem Management and RCA processes. Frequent reoccurrences imply gaps in preventive actions.

- **Customer Impact Score** (internal metric):
  - *Why:* Assigning a numeric customer impact score per incident enhances visibility into actual business impact and aids prioritization decisions.

---

## 📝 **Recommended Practices for Implementation:**

- Clearly define and communicate these metrics in your Confluence documentation.
- Integrate these metrics into dashboards (e.g., Grafana or internal systems) for real-time visibility.
- Regularly review incident response performance against these guidelines during weekly or monthly SRE team retrospectives.

By adopting these clear and reasoned standards, you'll significantly improve your incident management efficiency, customer trust, and continuous improvement capability.
