Cool, then we treat **any** `GUDI=` in `StorageRequestAdapter` as misuse, no special casing.

Here’s a cleaned-up, final set of **upload-only, adapter-only** queries you can actually paste into Splunk for your report.

---

## 1) Overall volume – uploads with vs without GUDI

```spl
index=am_cds environment=prod source="http:AWM_PROD"
"store.model.StorageRequestAdapter - Converting StorageRequestAdapter"
| eval has_gudi = if(match(log, "GUDI="), "upload_with_gudi", "upload_without_gudi")
| stats count by has_gudi
| eventstats sum(count) as total_uploads
| eval pct = round(100 * count / total_uploads, 2)
| sort -count
```

**Use:** one table in your deck –
“X of Y uploads (Z%) arrive with a client-supplied GUDI, which violates design (GUDI is server-generated).”

---

## 2) Time trend – is this stable or growing?

```spl
index=am_cds environment=prod source="http:AWM_PROD"
"store.model.StorageRequestAdapter - Converting StorageRequestAdapter"
| eval has_gudi = if(match(log, "GUDI="), "upload_with_gudi", "upload_without_gudi")
| timechart span=1h count by has_gudi
```

Change `span=1d` for a 30-day view.

---

## 3) Who’s doing it – group by some client identifier in attributes

Pick the attribute that best represents “client” in your `attributes={...}` block.
Example below uses `ACCOUNT_NUMBER`; swap to `SOURCE_SYSTEM`, `CHANNEL`, etc. if that’s better.

```spl
index=am_cds environment=prod source="http:AWM_PROD"
"store.model.StorageRequestAdapter - Converting StorageRequestAdapter"
| rex field=log "ACCOUNT_NUMBER=(?<account_number>[^,}]+)"
| eval upload_with_gudi = if(match(log, "GUDI="), 1, 0)
| stats count as total_uploads sum(upload_with_gudi) as gudi_uploads by account_number
| eval pct_gudi_uploads = round(100 * gudi_uploads / total_uploads, 2)
| where gudi_uploads > 0
| sort -gudi_uploads
```

**Use:**
Per-client misuse table: “For these accounts/systems, N% of uploads send GUDI.”

---

## 4) Infra distribution – which clusters/hosts see GUDI-uploads

```spl
index=am_cds environment=prod source="http:AWM_PROD"
"store.model.StorageRequestAdapter - Converting StorageRequestAdapter" "GUDI="
| stats count by cluster hostname
| sort -count
```

Shows where the pattern is most visible.

---

## 5) Sample events – clean vs dirty uploads

**Expected (no GUDI):**

```spl
index=am_cds environment=prod source="http:AWM_PROD"
"store.model.StorageRequestAdapter - Converting StorageRequestAdapter"
NOT "GUDI="
| table _time hostname cluster log
| head 5
```

**Anti-pattern (with GUDI):**

```spl
index=am_cds environment=prod source="http:AWM_PROD"
"store.model.StorageRequestAdapter - Converting StorageRequestAdapter" "GUDI="
| table _time hostname cluster log
| head 10
```

Drop 1–2 redacted lines of each into your mail / Confluence page so people can “see” the issue immediately.

---

### TL;DR for the forum

* Metric: `% of StorageRequestAdapter uploads that contain `GUDI=` in attributes`.
* Dimensions: time, client identifier (ACCOUNT_NUMBER / SOURCE_SYSTEM), cluster/host.
* Evidence: side-by-side log samples from the adapter.

That’s a tight, StorageRequestAdapter-only story with no `createDocument` or test-data distractions.
