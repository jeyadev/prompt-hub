You’re right — my last turn drifted. The root problem is that `spath` is flaky with keys that contain a colon (like `cds:PLATFORM`). It’ll silently fail even when the JSON is fine. The clean + reliable fix is to **normalize newlines** and stick to **`rex`** for the four fields.

Here’s a tight, working set you can drop in. (No `spath`.)

# 1) Base extractor (re-use everywhere)

```spl
index="am_cds" "Dispatching event to MQ for codeEvent" $time_token$
| eval raw_clean = replace(_raw, "[\r\n]+", " ")          /* make it single-line */
| rex field=raw_clean "payload\s+(?<json_payload>\{.*\})" /* isolate the JSON blob */
| rex field=json_payload max_match=1 "\"cds:PLATFORM\"\s*:\s*\{[^}]*\"value\"\s*:\s*\"(?<platform>[^\"]+)\""
| rex field=json_payload max_match=1 "\"cds:RequestStatus\"\s*:\s*\{[^}]*\"value\"\s*:\s*\"(?<request_status>[^\"]+)\""
| rex field=json_payload max_match=1 "\"cds:APPLICATION\"\s*:\s*\{[^}]*\"value\"\s*:\s*\"(?<application>[^\"]+)\""
| rex field=json_payload max_match=1 "\"cds:RequestStatusDescription\"\s*:\s*\{[^}]*\"value\"\s*:\s*\"(?<request_status_description>[^\"]+)\""
```

> Tip: save the above as a **base search** or macro; then append whatever `stats/timechart/table` you need.

# 2) Sanity check (prove extraction works)

```spl
... (base extractor) ...
| table _time platform request_status application request_status_description
```

# 3) Platform × status counts (includes FAILED)

```spl
... (base extractor) ...
| eval platform=coalesce(platform,"UNKNOWN")
| stats count by platform request_status
| sort platform request_status
```

# 4) Status counts only (quick view)

```spl
... (base extractor) ...
| stats count by request_status
| sort -count
```

# 5) Error rate single value (FAILED as errors)

```spl
... (base extractor) ...
| eval is_failed=if(request_status="FAILED",1,0)
| eventstats count as total, sum(is_failed) as failed
| eval error_rate=round(failed/total*100,2)
| stats values(error_rate) as error_rate
```

# 6) Exceptions by app / platform (useful triage)

```spl
... (base extractor) ...
| search request_status="FAILED"
| stats count by application platform request_status_description
| sort -count
```

# 7) “Why your earlier search showed no rows”

* You used `where isnull(platform)`. With the newline-normalize + `rex` above, **platform is present even for FAILED events**, so that filter legitimately returns 0 rows.
* The blanks you saw before were because `json_payload` didn’t capture across line breaks; then `platform` was null and your `stats by platform, request_status` dropped those events. The newline normalization + `rex` fixes that.

If you want, I can bundle these into a macro (`am_cds_mq_extract()`) and show the exact SimpleXML/JSON panel definitions.
