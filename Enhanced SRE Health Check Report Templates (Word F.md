<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" class="logo" width="120"/>

# 

---

# Enhanced SRE Health Check Report Templates (Word Format)

The following document presents refined daily and weekly health check report templates for Site Reliability Engineering teams. These templates have been structured to provide comprehensive system monitoring while maintaining executive readability and actionability.

## Daily SRE Health Check Report Template

The daily health check report serves as a critical communication tool that balances technical specificity with executive clarity. This template incorporates clear Red-Amber-Green (RAG) status indicators with precise threshold definitions.

### Email Subject Format

```
[DAILY HEALTH CHECK] System Status Report - Wednesday, March 12, 2025 - [STATUS: GREEN/AMBER/RED]
```


### Email Body Structure

```
Dear Leadership Team,

Below is the executive summary of today's health check performed at [TIME] by [ENGINEER NAME].

SYSTEM STATUS: [GREEN/AMBER/GREEN]

Executive Summary:
[2-3 sentence overview highlighting system performance, any critical issues, and immediate action items if needed]

Critical Metrics Status:
```


### Critical Metrics Table

The following table should be included with appropriate values and status indicators:


| Metric | Current Value | Threshold | Status | Trend | Weight |
| :-- | :-- | :-- | :-- | :-- | :-- |
| Availability | [99.xx%] | ≥99.9% (G), 99.0-99.9% (A), <99.0% (R) | [G/A/R] | [↑/↓/→] | Critical |
| Response Time | [xx ms] | ≤100ms (G), 101-300ms (A), >300ms (R) | [G/A/R] | [↑/↓/→] | Critical |
| Error Rate | [x.xx%] | <1% (G), 1-5% (A), >5% (R) | [G/A/R] | [↑/↓/→] | Critical |
| CPU Utilization | [xx%] | <50% (G), 50-80% (A), >80% (R) | [G/A/R] | [↑/↓/→] | High |
| Memory Usage | [xx%] | <70% (G), 70-85% (A), >85% (R) | [G/A/R] | [↑/↓/→] | High |
| Disk Space | [xx%] | <70% (G), 70-85% (A), >85% (R) | [G/A/R] | [↑/↓/→] | Medium |
| DB Connections | [xx/xx] | <80% (G), 80-90% (A), >90% (R) | [G/A/R] | [↑/↓/→] | High |

### Incident Summary

```
Incidents (Past 24 Hours):
[Brief description of any incidents that occurred, impact scope, resolution status, and mitigation actions]

Action Items:
[List of immediate actions being taken to address any issues, with owners and deadlines]

Upcoming Changes (Next 24 Hours):
[Any deployments or maintenance activities scheduled with expected impact]

Detailed system logs and metrics are available in the monitoring dashboard [LINK].

Regards,
[ENGINEER NAME]
On-Call SRE Engineer
```


## Weekly SRE Health Check Report Template

The weekly report provides a comprehensive view of system health over the past week, highlighting trends, recurring issues, and strategic recommendations.

### Email Subject Format

```
[WEEKLY HEALTH CHECK] System Performance Report - Week 10, 2025 - [STATUS: GREEN/AMBER/RED]
```


### Email Body Structure

```
Dear Leadership Team,

Please find below the weekly health check summary for March 3-9, 2025. This report has been compiled following comprehensive analysis by the SRE team.

OVERALL HEALTH STATUS: [GREEN/AMBER/RED]

Executive Summary:
[3-4 sentences summarizing the week's system performance, highlighting significant events, improvements, and overall health trends]
```


### Performance Metrics Table

| Metric | Current Value | Previous Week | Change | Threshold | Status | Notes |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| Uptime | [99.xx%] | [99.xx%] | [+/-x.xx%] | ≥99.9% | [G/A/R] | [Weekly trend] |
| Avg. Response Time | [xxx ms] | [xxx ms] | [+/-xx ms] | ≤200ms | [G/A/R] | [User impact] |
| Error Rate | [x.xx%] | [x.xx%] | [+/-x.xx%] | <1% | [G/A/R] | [Error pattern] |
| Deployment Success | [xx%] | [xx%] | [+/-xx%] | ≥95% | [G/A/R] | [CI/CD health] |
| Mean Time to Recovery | [xx h] | [xx h] | [+/-xx h] | ≤4h | [G/A/R] | [Incident recovery] |
| SLO Compliance | [xx%] | [xx%] | [+/-xx%] | ≥98% | [G/A/R] | [Critical SLOs] |
| Toil Reduction | [xx%] | [xx%] | [+/-xx%] | ↑ | [G/A/R] | [Automation gain] |

### Key Achievements Section

```
Key Achievements:
• [Achievement 1 with measurable impact]
• [Achievement 2 with measurable impact]
• [Achievement 3 with measurable impact]
```


### Incident Summary Table

| Incident ID | Description | Duration | Impact Scope | Root Cause | Resolution | Status |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| INC-xxxx | [Brief description] | [HH:MM] | [Users/services affected] | [Primary cause] | [Resolution steps] | [Open/Closed] |
| INC-xxxx | [Brief description] | [HH:MM] | [Users/services affected] | [Primary cause] | [Resolution steps] | [Open/Closed] |

### Infrastructure Status

```
Infrastructure Status:
[Overview of infrastructure health, including capacity utilization, scaling events, and performance trends]

Risk Register:
```


### Risk Register Table

| Risk Item | Severity | Likelihood | Current Status | Mitigation Plan | Owner |
| :-- | :-- | :-- | :-- | :-- | :-- |
| [Risk description] | [H/M/L] | [H/M/L] | [Status] | [Mitigation steps] | [Owner] |
| [Risk description] | [H/M/L] | [H/M/L] | [Status] | [Mitigation steps] | [Owner] |

```
Upcoming Work (Next Week):
• [Planned work item 1 with expected impact]
• [Planned work item 2 with expected impact]
• [Planned work item 3 with expected impact]

Recommendations:
Based on this week's findings, we recommend the following actions to improve system reliability:
1. [Recommendation 1 with expected impact]
2. [Recommendation 2 with expected impact]
3. [Recommendation 3 with expected impact]

Detailed metrics, logs, and incident reports are available in our monitoring system [LINK].

Regards,
[SRE MANAGER NAME]
SRE Team Manager
```


## RAG Status Calculation Framework

To standardize health status reporting across teams, implement the following threshold-based RAG status framework:

### Component-Level RAG Status

For each individual metric or component, apply these thresholds:

1. **Availability**:
    - **Green**: ≥99.9% availability
    - **Amber**: 99.0-99.9% availability
    - **Red**: <99.0% availability
2. **Response Time**:
    - **Green**: ≤100ms (or customized per service)
    - **Amber**: 101-300ms
    - **Red**: >300ms
3. **Error Rate**:
    - **Green**: <1%
    - **Amber**: 1-5%
    - **Red**: >5%
4. **Resource Utilization** (CPU, Memory, Disk):
    - **Green**: <50% (CPU), <70% (Memory/Disk)
    - **Amber**: 50-80% (CPU), 70-85% (Memory/Disk)
    - **Red**: >80% (CPU), >85% (Memory/Disk)

### Overall System RAG Status Calculation

Implement a weighted scoring system for overall health:

1. Assign weights to each component based on criticality:
    - Critical: 5 points
    - High: 3 points
    - Medium: 2 points
    - Low: 1 point
2. Calculate component scores:
    - Green: +1 point
    - Amber: 0 points
    - Red: -2 points
3. Calculate weighted total:
    - Multiply each component's score by its weight
    - Sum all weighted scores
4. Determine overall status:
    - **Green**: Total score ≥ 80% of maximum possible
    - **Amber**: Total score between 50-79% of maximum
    - **Red**: Total score < 50% of maximum

This framework provides a balanced, data-driven approach to health status determination that prioritizes critical components while reflecting overall system health.

## Implementing These Templates in Word Format

To create these reports as Word documents, you can either:

1. Copy and paste the above templates into Microsoft Word and format as needed.
2. Use the Python code available from search result with modifications to generate the documents programmatically, which is especially useful for automated reporting.

The Python code utilizes the python-docx library to create professionally formatted Word documents with proper tables, headings, and styling.

<div style="text-align: center">⁂</div>

[^1]: https://wikitech.wikimedia.org/wiki/Incident_response/Full_report_template

[^2]: https://stoneridgesoftware.com/wp-content/uploads/2016/01/Stoneridge-Software-Sample-Health-Check-Report.pdf

[^3]: https://www.template.net/business/report-templates/free-weekly-report-template/

[^4]: https://www.catchpoint.com/learn/sre-report-2025

[^5]: https://kb.icai.org/pdfs/PDFFile5b4362efeebc22.39783105.pdf

[^6]: https://docs.oracle.com/applications/biapps102/etl/GUID-C1BD3F54-E330-42F1-A391-2CE4801AD6FE.htm

[^7]: https://www.adobe.com/express/learn/blog/weekly-report

[^8]: https://www.dynatrace.com/resources/ebooks/sre-report/

[^9]: https://sre.google/workbook/postmortem-culture/

[^10]: https://www.cisco.com/c/en/us/td/docs/optical/ncs1014/configuration/guide/b-ncs1014-system-setup-guide-24-x-x/m-system-health-check.pdf

[^11]: https://www.canva.com/reports/templates/daily/

[^12]: https://www.linkedin.com/pulse/sre-playbook-step-akash-saxena-s3v3f

[^13]: https://sre.google/sre-book/incident-document/

[^14]: https://sre.google/sre-book/monitoring-distributed-systems/

[^15]: https://www.pinterest.com/pin/weekly-status-report-template-of-employee-free-report-templates--36873290690363151/

[^16]: https://github.com/bregman-arie/sre-checklist

[^17]: https://e2e.ti.com/cfs-file/__key/communityserver-discussions-components-files/196/LP38692MP_2D00_x_5F00_x-NOPB-Reliabilty-Report.docx

[^18]: https://sre.google/workbook/monitoring/

[^19]: https://visme.co/blog/daily-report-templates/

[^20]: https://www.servicenow.com/docs/bundle/yokohama-security-management/page/product/secops-analyst-workspace/task/create-report-template-sir.html

