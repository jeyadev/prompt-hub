Here you go — a **clean, consolidated Confluence-ready report** followed by a **Jira task description**.
This version assumes we are extracting **PLATFORM**, **APPLICATION**, and **TRACKING_STATUS** *only when present*, and ignoring when absent.

Before we get to the report, here’s the updated extraction block you will use in Splunk:

---

# ✅ **Updated Field Extraction (handles missing fields gracefully)**

```spl
index=am_cds environment=prod source="http:AWM_PROD"
"store.model.StorageRequestAdapter - Converting StorageRequestAdapter"
| eval has_gudi = if(match(log, "GUDI="), 1, 0)
| eval has_tracking_status = if(match(log, "TRACKING_STATUS="), 1, 0)

| rex field=log "PLATFORM=(?<platform>[^,}]+)" 
| rex field=log "APPLICATION=(?<application>[^,}]+)" 
| rex field=log "TRACKING_STATUS=(?<tracking_status>[^,}]+)" 

| eval platform = coalesce(platform, "N/A")
| eval application = coalesce(application, "N/A")
| eval tracking_status = coalesce(tracking_status, "N/A")
```

Use this as your base extraction for any analysis.

---

# 🧩 **CDS Store Upload Misuse Analysis – Confluence Report**

**Title:** *CDS Store – Upload API Contract Violation Analysis (Based on StorageRequestAdapter Logs)*
**Date:** `<Insert date>`
**Prepared by:** SRE / Platform Reliability Engineering

---

# 📌 **1. Background**

The **upload workflow** in CDS Store requires that:

* **GUDI is generated internally** by CDS → clients must *not* supply it.
* **TRACKING_STATUS** and other lifecycle fields are only applicable to **update** operations, not uploads.
* **PLATFORM** and **APPLICATION** fields help identify consumer origins but are optional.

Misuse of these fields during upload calls can cause:

* Incorrect lifecycle processing
* Duplicate / orphaned document states
* Downstream reconciliation issues
* Contract violations leading to long-term technical debt

This report summarizes upload behaviour in production using the **StorageRequestAdapter** logs as the source of truth.

---

# 📊 **2. Key Metrics Summary**

| Metric                                  | Count | % of Total     | Notes                                               |
| --------------------------------------- | ----- | -------------- | --------------------------------------------------- |
| **Total Uploads**                       | **X** | 100%           | All StorageRequestAdapter upload events             |
| **Uploads WITH GUDI**                   | **Y** | **Y/X × 100%** | Clients are incorrectly supplying GUDI              |
| **Uploads WITHOUT GUDI**                | **Z** | **Z/X × 100%** | Expected, compliant behaviour                       |
| **Uploads WITH GUDI + TRACKING_STATUS** | **W** | **W/X × 100%** | Severe misuse: treating upload as a document update |

> **Interpretation:** A non-trivial portion of uploads contain client-supplied identifiers and lifecycle metadata. These behaviours violate CDS API contract expectations and may impact ingestion health, idempotency, and lifecycle accuracy.

---

# 🧠 **3. Observed Misuse Patterns**

### **3.1 Uploads containing GUDI**

Clients are generating or passing their own GUDI values.
This breaks the ingestion contract and can:

* Collide with CDS-generated identifiers
* Cause multiple documents to map incorrectly in downstream services
* Lead to unexpected retry behaviour based on GUDI reuse

---

### **3.2 Uploads containing TRACKING_STATUS**

TRACKING_STATUS belongs exclusively to update operations.
Seeing it in uploads implies that:

* Clients misunderstand the ingestion lifecycle
* They treat upload as a “full document lifecycle update” endpoint
* Metadata downstream (audit, versioning, reconciliation) becomes corrupted

---

### **3.3 Uploads containing both GUDI + TRACKING_STATUS**

This is the strongest misuse signature.

Such clients are effectively:

* Generating their own document ID
* Passing lifecycle metadata
* Treating upload as a combined “create + update” endpoint

This is a **critical architectural violation**.

---

# 🧭 **4. Consumer Identification via Optional Fields**

Where present, the following fields were extracted:

| Field               | Meaning                                    | Use                                   |
| ------------------- | ------------------------------------------ | ------------------------------------- |
| **PLATFORM**        | Source platform (e.g., LUX, PAM)           | High-level consumer grouping          |
| **APPLICATION**     | Application sending the upload             | Allows direct routing to owning teams |
| **TRACKING_STATUS** | Lifecycle state consumers incorrectly send | Helps identify severity of misuse     |

Not all consumers populate these fields, but they provide strong signals when present.

Example extracted values:

* PLATFORM = LUX
* APPLICATION = PAM
* TRACKING_STATUS = Complete

---

# 📈 **5. Behaviour By Consumer Group**

(Replace with actual Splunk output once available)

### Example format:

| PLATFORM | APPLICATION | Uploads With GUDI | Uploads With GUDI + TRACKING_STATUS | Comments                                    |
| -------- | ----------- | ----------------- | ----------------------------------- | ------------------------------------------- |
| PAM      | PAM         | 4,102             | 2,150                               | Significant misuse from PAM batch ingestion |
| LUX      | N/A         | 1,900             | 320                                 | Lifecycle metadata included with uploads    |
| N/A      | N/A         | 850               | 0                                   | Cannot identify consumer system             |

This section helps the product team target conversations with the correct owners.

---

# 🧪 **6. Sample Misuse Event (Redacted)**

Below is an anonymized real example showing both GUDI and TRACKING_STATUS in an upload:

```
store.model.StorageRequestAdapter - Converting StorageRequestAdapter [
 attributes={
   PID=DRH043N,
   GUDI=c8e55798-cf29-4ae3-9884-d3af53b3a3ee,          <-- Should NOT be present
   ACCOUNT_NUMBER=7269397,
   PLATFORM=LUX,
   APPLICATION=PAM,
   TRACKING_STATUS=Complete,                          <-- Should NOT be present
   PHYSICAL_STATUS=Original,
   DOC_CODE=045,
   DOC_DATE=2024-12-05
 }
]
```

This example is suitable for use in communication to consumer teams.

---

# 🚨 **7. Impact Assessment**

### ❗ Contract Friction

Consumers using upload as an update endpoint violate designed separation of concerns.

### ❗ Reconciliation Issues

Incorrect TRACKING_STATUS during upload can cause:

* premature state transitions
* mismatches in CMS / downstream processors

### ❗ Audit Integrity

Lifecycle transitions are logged twice or in incorrect sequence when passed during upload.

### ❗ Retrying & Idempotency Issues

Client-generated GUDI values can cause:

* overwrite-like behaviour
* replayed ingestion incorrectly updating older documents

---

# 📬 **8. Recommended Product Team Actions**

1. **Communicate misuse findings** to affected consumer teams.
2. **Share correct ingestion contract**, explicitly stating:

   * GUDI is assigned by CDS
   * TRACKING_STATUS is not valid during upload
3. **Review documentation** to ensure upload/update separation is clear.
4. **Introduce schema validation** (recommended):

   * Reject uploads containing GUDI
   * Reject uploads containing TRACKING_STATUS
5. **Set up monitoring dashboards** to track misuse monthly.

---

# 🧱 **9. Follow-Up & Tracking**

* Owner: Product Engineering
* Supporting: SRE, Architecture
* Deliverable: Consumer remediation plan
* Timeline: `<Insert>`

---

# 📝 **Jira Task Description (Short, Ready-To-Paste)**

**Title:** Investigate and Remediate Upload API Contract Violations (GUDI & TRACKING_STATUS Misuse)

**Description:**
Analysis of CDS Store’s StorageRequestAdapter logs shows that multiple consumer systems are incorrectly sending **GUDI** and **TRACKING_STATUS** within upload requests. These fields are not valid for ingestion and represent API contract violations. Misuse may lead to lifecycle corruption, audit inconsistencies, and ingestion retries.

This task involves:

* Reviewing misuse patterns
* Identifying impacted consumer applications (via PLATFORM/APPLICATION fields)
* Communicating corrective actions
* Updating contract documentation
* Defining next steps for enforcing validation at CDS Store level

**Acceptance Criteria:**

* Complete list of violating consumers delivered
* Communication sent to all affected teams
* Proposal for validation enforcement drafted
* Dashboard queries stored for future audit

---

If you want, I can also generate:

✔ A Confluence page template
✔ A PowerPoint summary
✔ A Splunk dashboard JSON for automated reporting
