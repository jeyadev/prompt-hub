# [DAILY HEALTH CHECK] System Status Report - Wednesday, March 12, 2025 - [STATUS: GREEN/AMBER/RED]

---

**Dear Leadership Team,**

Please find below the daily health check summary for **March 12, 2025**. This report focuses on our two critical applications—**abc-store** and **abc-find**—and their key methods: **upload, download, and find**. The goal is to provide you with a rapid, data-driven overview of today's system performance and any noteworthy incidents.

────────────────────────────────────────────  
## Overall System Health  
- **Status:** [🟢 GREEN / 🟠 AMBER / 🔴 RED]  
- **Key Metrics:**
  - **Uptime:** [99.xx%]
  - **Average Response Time:** [xxx ms]
  - **Error Rate:** [x.xx%]
────────────────────────────────────────────  

## Executive Summary
- **Overview:** Today’s system performance was **[brief performance summary]**.
- **Highlights:** [2-3 brief sentences on system performance, significant alerts, and immediate remediation steps.]

────────────────────────────────────────────  
## Application Performance Metrics

### abc-store
| **Method**  | **Avg. Response Time** | **Error Rate** | **Throughput** | **Status** | **Trend**  |
|-------------|------------------------|----------------|----------------|------------|------------|
| Upload      | [xxx ms]               | [x.xx%]       | [xxx req/min]  | [G/A/R]    | [↑/↓/→]   |
| Download    | [xxx ms]               | [x.xx%]       | [xxx req/min]  | [G/A/R]    | [↑/↓/→]   |
| Find        | [xxx ms]               | [x.xx%]       | [xxx req/min]  | [G/A/R]    | [↑/↓/→]   |

### abc-find
| **Method**  | **Avg. Response Time** | **Error Rate** | **Throughput** | **Status** | **Trend**  |
|-------------|------------------------|----------------|----------------|------------|------------|
| Upload      | [xxx ms]               | [x.xx%]       | [xxx req/min]  | [G/A/R]    | [↑/↓/→]   |
| Download    | [xxx ms]               | [x.xx%]       | [xxx req/min]  | [G/A/R]    | [↑/↓/→]   |
| Find        | [xxx ms]               | [x.xx%]       | [xxx req/min]  | [G/A/R]    | [↑/↓/→]   |

────────────────────────────────────────────  
## Incident Summary

| **Incident ID** | **App**    | **Method** | **Description**           | **Impact**         | **Resolution**         | **MTTR**   |
|-----------------|------------|------------|---------------------------|--------------------|------------------------|------------|
| INC-004         | abc-store  | Upload     | [Brief description]       | [Impact summary]   | [Resolution summary]   | [Time]     |
| INC-005         | abc-find   | Find       | [Brief description]       | [Impact summary]   | [Resolution summary]   | [Time]     |

────────────────────────────────────────────  
## Action Items & Upcoming Changes
- **Action Items:**  
  - [Owner Name] to investigate [specific issue] – **Due:** [Time/Date].
- **Upcoming Changes:**  
  - Scheduled maintenance or deployments: [Brief description and expected impact].

────────────────────────────────────────────  
## Dependencies & Additional Notes
- **Dependency Health:**  
  - All upstream dependencies (DB, Object Store) are **[Healthy/Degraded/Offline]**.
- **Dashboard Access:**  
  - For detailed metrics and logs, please refer to our [Monitoring Dashboard](LINK).

────────────────────────────────────────────  

Thank you for your attention. If you have any questions or require further insights, please do not hesitate to reach out.

**Best regards,**  
**[ENGINEER NAME]**  
On-Call SRE Engineer





Version 2
# [HEALTH CHECK] Daily Report for [Date]

**On-Call Engineer:** [Your Name]  
**Time:** [Time]

---

## Overall RAG Summary

| **Component**                | **Status (G/A/R)** |
|------------------------------|--------------------|
| **Infrastructure**           | [G/A/R]            |
| **Application Performance**  | [G/A/R]            |
| **Consumer Health Check**    | [G/A/R]            |
| **Content Manager Health**   | [G/A/R]            |
| **DB Health**                | [G/A/R]            |

---

## Highlights & Call Outs

- **[Highlight 1]:** [Brief description of any critical incident or noteworthy issue.]
- **[Highlight 2]:** [Any additional observations or actions needed.]
- **[Highlight 3]:** [Further call outs, if applicable.]

---

## Additional Notes / Actions

- [List any further notes, action items, or recommendations.]

---

## Dashboards for Detailed Health Monitoring

- [Dynatrace Dashboard](https://www.dynatrace.com/)  
- [Splunk Dashboard](https://www.splunk.com/)

---

**End of Report**



# Daily Health Check - Quick Reference

This page explains **where** to look for the key metrics and **how** to verify the system’s status before filling out the [HEALTH CHECK] Daily Report.

---

## 1. Infrastructure
- **Dynatrace**: 
  - Check host-level CPU, memory, and disk usage across nodes.
  - Look for any spikes or abnormal trends over the past 24 hours.
- **Splunk**: 
  - Search for system-level errors (e.g., OS or container logs).
  - Filter on keywords like “OOM”, “disk full”, or “CPU throttling”.

> **Tip**: Mark **Infrastructure** Amber or Red if resource usage is consistently above thresholds (e.g., >80% CPU or memory) or if you see recurring errors in Splunk logs.

---

## 2. Application Performance
- **Dynatrace**:
  - Review response times, throughput, and error rates for each microservice (abc-store, abc-find, etc.).
  - Pay attention to any new performance degradation or slowdowns at P95/P99 latency.
- **Splunk**:
  - Check application logs for spikes in exceptions, timeouts, or specific error codes.

> **Tip**: If response times or error rates exceed your defined Amber/Red thresholds, update the RAG status accordingly.

---

## 3. Consumer Health Check
- **Automated Consumer Health Email** (Attached or Emailed):
  - Scan for any Red or Amber statuses among consumers.
- **Dynatrace** (if integrated):
  - Look at service calls from consumer endpoints to detect anomalies.

> **Tip**: If multiple consumers are reporting errors or failing health checks, mark **Consumer Health** Amber or Red.

---

## 4. Content Manager Health
- **Dynatrace**:
  - Check the Content Manager service dashboard for latency, CPU usage, and errors.
- **Splunk**:
  - Filter logs by the Content Manager service name for any error patterns (e.g., 5xx codes, repeated exceptions).

> **Tip**: A few transient errors might be normal, but sustained failures or high latency warrants an Amber or Red rating.

---

## 5. DB Health
- **Dynatrace** or **Database Monitoring Tool**:
  - Check DB CPU usage, connection pool usage, and slow queries.
  - Ensure replication (if any) is in sync.
- **Splunk**:
  - Look for DB-related error logs, especially if the application can’t connect or queries are timing out.

> **Tip**: If the database shows sustained high usage (e.g., CPU > 80%), slow queries, or high error rates, consider marking **DB Health** Amber or Red.

---

## Determining RAG Status
- **Green (G)**: 
  - Normal operating range, no or minimal alerts.
- **Amber (A)**: 
  - Metrics nearing thresholds or sporadic errors; needs attention but not immediately critical.
- **Red (R)**: 
  - Critical thresholds exceeded or major outages; immediate investigation required.

> **Note**: Always use your best judgment and consider business impact. If a single spike recovers quickly, you might still mark Green; but a sustained issue or repeated spikes often justifies Amber or Red.

---

## Quick Links
- [Dynatrace Dashboard](https://www.dynatrace.com/)  
- [Splunk Dashboard](https://www.splunk.com/)

---

**End of Instructions**
