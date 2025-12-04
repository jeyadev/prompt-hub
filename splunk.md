Got it – we’ll ignore *all* update flows and treat **only upload/createDocument** calls where a `GUDI=` is present as the anti-pattern.

Let’s lock in a clean, upload-only report set.

---

## 1) Overall: uploads with vs without GUDI

**Story:** “Out of all upload (createDocument) requests, X% were sent *with* a client GUDI, which is against design.”

```spl
index=am_cds environment=prod source="http:AWM_PROD"
"CdsRequestInterceptor" "uri=/store/api/v3/createDocument"
| eval has_gudi = if(match(log, "GUDI="), "upload_with_gudi", "upload_without_gudi")
| stats count by has_gudi
| eventstats sum(count) as total_uploads
| eval pct = round(100 * count / total_uploads, 2)
| sort -count
```

---

## 2) Time trend: how client-GUDI uploads evolve

**Story:** “This is continuous behaviour, not a one-off; here’s the trend.”

```spl
index=am_cds environment=prod source="http:AWM_PROD"
"CdsRequestInterceptor" "uri=/store/api/v3/createDocument"
| eval has_gudi = if(match(log, "GUDI="), "upload_with_gudi", "upload_without_gudi")
| timechart span=1h count by has_gudi
```

* Use `span=1d` for a longer timeframe.

---

## 3) Which clients are doing it (APPNAME view)

**Story:** “Misuse is concentrated in these apps; here is the % of *their* uploads that send a GUDI.”

```spl
index=am_cds environment=prod source="http:AWM_PROD"
"CdsRequestInterceptor" "uri=/store/api/v3/createDocument"
| rex field=log "APPNAME=\[(?:APPNAME=)?(?<appname>[^,\]]+)"
| eval upload_with_gudi = if(match(log, "GUDI="), 1, 0)
| stats count as total_uploads sum(upload_with_gudi) as gudi_uploads by appname
| eval pct_gudi_uploads = round(100 * gudi_uploads / total_uploads, 2)
| where gudi_uploads > 0
| sort -gudi_uploads
```

Use this table directly in the deck/email to app owners.

---

## 4) HTTP status distribution for *upload-with-GUDI* calls

**Story:** “Most of these anti-pattern uploads are actually succeeding (or failing) with status X, so they’re ‘working’ today and baking in a bad contract.”

```spl
index=am_cds environment=prod source="http:AWM_PROD"
"CdsRequestInterceptor" "uri=/store/api/v3/createDocument" "GUDI="
| rex field=log "httpStatus=(?<httpStatus>\d+)"
| stats count by httpStatus
| eventstats sum(count) as total
| eval pct = round(100 * count / total, 2)
| sort -count
```

---

## 5) Concrete examples of “upload with GUDI” from storage layer

For screenshots like the ones you shared (“Upload with GUDI” lines):

```spl
index=am_cds environment=prod source="http:AWM_PROD"
"store.model.StorageRequestAdapter - Converting StorageRequestAdapter"
"GUDI="
| table _time hostname cluster log
| head 50
```

And for the “clean” baseline (upload without GUDI):

```spl
index=am_cds environment=prod source="http:AWM_PROD"
"store.model.StorageRequestAdapter - Converting StorageRequestAdapter"
NOT "GUDI="
| table _time hostname cluster log
| head 10
```

These give you side-by-side samples to paste into the wider-forum doc:

* **Expected:** upload adapter log without `GUDI=`.
* **Observed anti-pattern:** upload adapter log with `GUDI=` plus the interceptor log above showing the same.

---

### How this reads in the forum

With just these 5 queries you can say:

* “In the last *N* days, **X% of uploads** to `/store/api/v3/createDocument` were sent with a **client-supplied GUDI**, even though GUDI must be generated server-side.”
* “This behaviour is driven mainly by these apps: A, B, C (table from Query 3).”
* “Here are example log lines from the upload path showing the client-supplied `GUDI` wired into our storage adapter (Queries 5).”
* “Today these calls mostly return status … (Query 4), so clients think this is a supported behaviour – we should communicate and then enforce server-only GUDI generation.”

That’s a tight, upload-only, design-aligned story.
