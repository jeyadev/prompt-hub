You’re in the **“Save As Alert”** screen. Splunk is basically asking you: *when should I panic, and who do I annoy when I panic?* Let’s wire it properly so it only yells when a bean is missing, and the alert body contains the missing list.

## 1) Use this SPL (so “no missing” = **0 results**)

This is important because then your trigger is dead simple: **Number of Results > 0**.

**Paste this as the saved search:**

```spl
index="am_cds" "Distribute Quartz Scheduler"
earliest=-35m@m latest=now
| rex "beanName\s+(?<beanName>\w+)"
| dedup beanName
| stats values(beanName) as ran
| eval run_hour=strftime(now(), "%H")
| eval expected=case(
    run_hour=="06", split("GTA1", ","),
    run_hour=="12", split("FEES,COACS,GTA2,TAXRS,PMC,ENGAGE,IPBMANDATE,EMIR,OTC,CCTSWISS,STGIPBSTMT,FMIA,SUITABLIBP", ",")
  )
| where mvcount(expected)>0
| mvexpand expected
| eval present=if(mvfind(ran, expected) >= 0, 1, 0)
| where present=0
| stats values(expected) as missing count as missing_count values(ran) as ran_list
| eval alert_message="Missing bean(s): ".mvjoin(missing,", ")." | Ran: ".mvjoin(ran_list,", ")." | Window: last 35m"
| table missing_count missing ran_list alert_message
```

This returns **one row** only when something is missing. Perfect for alerts.

---

## 2) Fill the “Settings” section

### Title

**CDS Quartz Scheduler Missing Beans (06:30 / 12:30)**

### Description (paste this)

Checks Quartz scheduler execution logs 30 mins after cron. Alerts if any expected beanName did not run in the last 35 minutes. 06:30 expects GTA1 only; 12:30 expects the noon batch list. Output includes missing beans and what ran.

Permissions: whatever your team standard is (Private vs Shared).

---

## 3) Schedule it correctly (this is where humans usually trip)

Right now your UI shows **“Run every week” + Monday 6:00**. That’s not what you want.

### Option A (best): **one alert, cron schedule**

In the “Run every …” dropdown, pick **Cron Schedule** (or “Advanced” scheduling).
Use:

* **Mon–Fri at 06:30 and 12:30**

  ```
  30 6,12 * * 1-5
  ```

If Splunk UI doesn’t allow comma hours, create **two alerts** instead (Option B).

### Option B: two alerts (boring but foolproof)

* Alert 1 schedule:

  ```
  30 6 * * 1-5
  ```
* Alert 2 schedule:

  ```
  30 12 * * 1-5
  ```

Both can reuse the same SPL (it auto-detects hour).

---

## 4) Trigger Conditions (make it scream only when needed)

### Trigger alert when

* **Number of Results**
* **is greater than**
* **0**

### Trigger

* Choose **Once** (since your search already collapses missing into one row)

### Throttle

Enable if you want to avoid repeats (recommended):

* Throttle: ✅
* Suppress for: **60 minutes** (or 30, your call)

---

## 5) Trigger Actions (this is where you “populate details”)

Click **+ Add Actions** → choose Email / Slack / whatever.

### If Email:

* ✅ Include search results
* Format: **Inline** (table) if possible
* ✅ Include: **Results** (so the row with `alert_message` appears)

**Email Subject (use tokens):**

```
[CDS Scheduler] Missing bean(s) at $result.alert_message$
```

If that token doesn’t render in subject in your Splunk version, use:

```
[CDS Scheduler] Missing beans detected
```

**Email Body (reliable, renders in most setups):**

```
Scheduler validation failed.

Time: $trigger_time$
Search: $name$

Missing beans: $result.missing$
Missing count: $result.missing_count$
Beans that ran: $result.ran_list$

Details: $result.alert_message$
```

### If Slack:

Same idea: send `$result.alert_message$` + `$result.missing$`.

---

## 6) One tiny but crucial thing

Leave the SPL’s `earliest=-35m@m latest=now` as-is **OR** set the alert time range in UI. Don’t do both differently unless you enjoy debugging phantom misses.

---

If you configure it like this, Splunk will only alert when it actually has something useful to say (rare trait in software), and the message will literally name the missing bean(s).

```
| makeresults
| eval batch="06,12"
| makemv delim="," batch
| mvexpand batch
| eval expected=case(
    batch=="06", split("GTA1", ","),
    batch=="12", split("FEES,COACS,GTA2,TAXRS,PMC,ENGAGE,IPBMANDATE,EMIR,OTC,CCTSWISS,STGIPBSTMT,FMIA,SUITABLIBP", ",")
  )
| join type=left batch [
    search index="am_cds" "Distribute Quartz Scheduler" earliest=-1d@d latest=@d
    | rex "beanName\s+(?<beanName>\w+)"
    | eval hhmm=strftime(_time, "%H:%M")
    | eval batch=case(
        hhmm>="05:55" AND hhmm<="06:30", "06",
        hhmm>="11:55" AND hhmm<="12:30", "12",
        1==1, null()
      )
    | where isnotnull(batch)
    | stats values(beanName) as ran by batch
]
| eval ran=coalesce(ran, mvappend())
| mvexpand expected
| eval present=if(mvfind(ran, expected) >= 0, 1, 0)
| where present=0
| stats values(expected) as missing count as missing_count values(ran) as ran_list by batch
| eval day=strftime(relative_time(now(),"-1d@d"), "%Y-%m-%d")
| eval alert_message="Day=".day." Batch=".batch." Missing=".mvjoin(missing,", ")." | Ran=".if(mvcount(ran_list)=0,"<none>",mvjoin(ran_list,", "))
| table day batch missing_count missing ran_list alert_message
| sort batch
```
