# Incident Classification & Response Guide

This document outlines the standards, criteria, and recommended response cadences for incident management, ensuring clarity, consistency, and efficiency in handling payment systems incidents.

## Incident Priority Levels

| Priority | Analogy (For easy recall) | Description | Initial Response | Update Frequency | MTTR Target |
|----------|---------------------------|-------------|------------------|------------------|-------------|
| **P1** *(Critical)* | **"ATM Spits No Cash"** | Complete outage or critical functionality loss. Immediate customer/business impact. | Within **15 mins** | Every **30 mins** | Within **4 hrs** |
| **P2** *(High)* | **"Credit Card Declined Unexpectedly"** | Moderate flaw; service degradation or partial functionality loss with moderate user/business impact. | Within **30 mins** | Every **2 hrs** | Within **8 hrs** |
| **P3** *(Moderate)* | **"Slow Mobile Banking App"** | Partial non-critical functionality loss; minor impact with simple workarounds available. | Within **1 hr** | Every **4 hrs** | Within **24 hrs** |
| **P4** *(Low)* | **"Incorrect Transaction Description"** | Minimal user impact; cosmetic or minor issue with limited business effect. | Within **4 hrs** | Every **24 hrs** or as status changes | Within scheduled maintenance window (up to **7 days**) |
| **P5** *(Informational)* | **"Routine Maintenance Notice"** | No immediate impact; informational or routine updates. | Within **1 business day** | As status changes | As capacity allows |

## Detailed Priority Definitions

### Priority 1 (P1) – Critical Incidents

Priority 1 incidents are further classified into severity levels:

| Severity | Analogy | Explanation | Example |
|----------|---------|-------------|---------|
| 🔴 **Severity 1** *(Highest)* | **"Entire Bank Offline"** | Complete global outage affecting all customers. Immediate global escalation required. | Global payment network down. |
| 🔶 **Severity 2** | **"Major Region Payments Failure"** | Major segment/service outage, significant customer impact. | Payment outages across an entire region. |
| 🟡 **Severity 3** | **"Single Major Service Unavailable"** | Critical service unavailable; significant immediate impact. | International wire transfer outage. |
| 🟢 **Severity 4** *(Lowest)* | **"High-Value Transactions Delay"** | Essential service degradation; manageable short-term impact. | Delay in regulatory-bound high-value payments. |

### Priority 2 (P2) – High Incidents

- Non-critical features unavailable.
- Moderately impaired technology.
- Increased risk to availability.
- Initial monitoring events or degraded performance.
- Moderate user disruption with available workaround.
- Immediate risk to production milestones.
- Moderate-risk cyber events.

### Priority 3 (P3) – Moderate Incidents

- Minor technical/application issues.
- User-available workarounds.
- Issues in non-production or lower-RTO environments.
- Moderate risk to medium-term milestones.

### Priority 4 (P4) – Low Incidents

- Cosmetic or minor functionality issues.
- General inquiries or questions.
- Requests outside standard channels.

### Priority 5 (P5) – Informational

- Informational logs/events.
- No immediate impact, deferred action.

## Incident Response Guidelines

### Initial Response & Update Cadence

- **P1**: Respond within 15 mins, update every 30 mins.
- **P2**: Respond within 30 mins, update every 2 hrs.
- **P3**: Respond within 1 hr, update every 4 hrs.
- **P4**: Respond within 4 hrs, update every 24 hrs or status change.
- **P5**: Respond within 1 business day, updates as status changes.

### MTTR Targets

- **P1**: Within 4 hrs
- **P2**: Within 8 hrs
- **P3**: Within 24 hrs
- **P4**: Scheduled maintenance window (up to 7 days)
- **P5**: Capacity-dependent

## Additional Metrics for Continuous Improvement

- **Mean Time to Acknowledge (MTTA)**: Efficiency in initial recognition.
- **Mean Time to Detect (MTTD)**: Effectiveness of monitoring systems.
- **False Positives Rate**: Alert accuracy; high false positives lead to alert fatigue.
- **Incident Reoccurrence Rate**: Effectiveness of corrective actions.
- **Customer Impact Score**: Numeric representation of customer impact for prioritization.

## Important Notes

- **Escalation & Cadence**: Adhere to defined escalation paths.
- **Root Cause Analysis**: Mandatory for P1 incidents.
- **Prioritization**: Always prioritize based on business and customer impact.

## Reference & Resources

- Escalation Matrix
- Categorization Matrix
- Problem Management & Root Cause Analysis
- Alert Response Guidelines

Consistent adherence to this guide ensures clarity, operational efficiency, rapid resolution, and continuous process improvement.

