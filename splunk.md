You want to basically “build the case” that clients are abusing `createDocument` by sending `gudi` themselves. So first let’s decide **what story we want the data to tell**, then I’ll give you Splunk queries for each piece.

---

## 1. What key data to gather for the wider forum

I’d structure the report around these questions:

1. **How common is this anti-pattern?**

   * Total `createDocument` calls vs those where a `GUDI` is present in the request.
   * Trend over time (last 7/30 days).
   * Percentage of total traffic affected.

2. **Who is doing it?**

   * Top **APPNAMEs** (or whatever you use as client ID) that send a `GUDI`.
   * Their FIDs / client IPs / gateway, so owners can be found.
   * Per-app percentage: “X% of your createDocument calls are sending GUDI.”

3. **What’s the impact on the system?**

   * HTTP status distribution for requests with client `GUDI` (200 vs 4xx/5xx).
   * Any obvious latency impact (optional, but nice to have).
   * If you can, a quick check if these collide with already existing GUDIs (maybe in another log / CM side).

4. **When & where is it happening?**

   * Time series of `client_gudi` usage.
   * Breakdown by cluster / hostname / gateway.

5. **A few concrete examples**

   * Sample events (redacted) for the worst offenders: `APPNAME`, `FID`, `clientIp`, `GUDI`, `uri`.

That’s more than enough to walk into a design / governance forum and say:

> “Here are the apps, here is the volume, here’s the risk. We need to enforce ‘server-generated only’ for GUDI.”

---

## 2. Splunk queries for each slice

I’ll assume from your screenshot:

* Index: `am_cds`
* `CdsRequestInterceptor` log line with `createDocument`
* `log` field holds the long trace with `GUDI=...`, `APPNAME=[APPNAME=...]`, `FID=...`, `httpStatus=...`
* Environment filter: `environment=prod` (tweak to your reality)
* Source example from screenshot: `source=http:AWM_PROD`

You can tweak time ranges (`earliest`, `latest`) as needed.

---

### 2.1 Total createDocument vs “client sent GUDI”

```spl
index=am_cds environment=prod source="http:AWM_PROD" "CdsRequestInterceptor" "createDocument"
| eval has_gudi = if(match(log, "GUDI="), "client_gudi", "no_client_gudi")
| stats count by has_gudi
| eventstats sum(count) as total
| eval pct = round(100 * count / total, 2)
| sort -count
```

**What you get**

* Number and % of `createDocument` calls where a `GUDI` was present in the request log.

---

### 2.2 Time trend: how often are clients sending GUDI?

```spl
index=am_cds environment=prod source="http:AWM_PROD" "CdsRequestInterceptor" "createDocument"
| eval has_gudi = if(match(log, "GUDI="), "client_gudi", "no_client_gudi")
| timechart span=1h count by has_gudi
```

Change `span` to `15m` / `1d` depending on the timeframe.

**Use this** graph in slides to show this isn’t a one-off.

---

### 2.3 Top offending clients (APPNAME/FID)

```spl
index=am_cds environment=prod source="http:AWM_PROD" "CdsRequestInterceptor" "createDocument" "GUDI="
| rex field=log "GUDI=(?<gudi>[0-9a-fA-F-]+)"
| rex field=log "APPNAME=\[(?:APPNAME=)?(?<appname>[^,\]]+)"
| rex field=log "FID=(?<fid>[A-Z0-9]+)"
| stats count dc(gudi) as unique_gudi by appname fid
| sort -count
| head 20
```

**What you get**

* Top 20 app/FID pairs sending GUDI and how many times + distinct GUDIs.

If `APPNAME=` pattern differs slightly, just adjust that `rex`.

---

### 2.4 Per-client misuse rate (% of their calls with client GUDI)

This is the “shame chart” per app, but data-driven.

```spl
index=am_cds environment=prod source="http:AWM_PROD" "CdsRequestInterceptor" "createDocument"
| rex field=log "APPNAME=\[(?:APPNAME=)?(?<appname>[^,\]]+)"
| eval has_gudi = if(match(log, "GUDI="), 1, 0)
| stats count as total_calls sum(has_gudi) as gudi_calls by appname
| eval pct_with_gudi = round(100 * gudi_calls / total_calls, 2)
| sort -gudi_calls
```

**What you get**

* For each `appname`: total calls, calls with GUDI, and percentage.

You can filter to “bad” apps only:

```spl
... | where pct_with_gudi > 0
```

---

### 2.5 HTTP status distribution for requests with client GUDI

```spl
index=am_cds environment=prod source="http:AWM_PROD" "CdsRequestInterceptor" "createDocument" "GUDI="
| rex field=log "httpStatus=(?<httpStatus>\d+)"
| stats count by httpStatus
| eventstats sum(count) as total
| eval pct = round(100 * count / total, 2)
| sort -count
```

**Why this matters**

* If it’s mostly 200s, you can say “this misuse appears to ‘work’, which is dangerous because it bakes in a wrong contract.”
* If you see 4xx/5xx clusters, you can tie it to operational noise or user impact.

---

### 2.6 Latency comparison (optional but nice)

If you have duration fields like `duration=2805` in the log:

```spl
index=am_cds environment=prod source="http:AWM_PROD" "CdsRequestInterceptor" "createDocument"
| rex field=log "duration=(?<duration_ms>\d+)"
| eval duration_ms = tonumber(duration_ms)
| eval has_gudi = if(match(log,"GUDI="),"client_gudi","no_client_gudi")
| stats avg(duration_ms) p90(duration_ms) p99(duration_ms) by has_gudi
```

This gives you a quick view of whether “client GUDI” calls behave differently performance-wise.

---

### 2.7 Breakdown by gateway / cluster / hostname

#### By gateway

```spl
index=am_cds environment=prod source="http:AWM_PROD" "CdsRequestInterceptor" "createDocument" "GUDI="
| rex field=log "gateway=(?<gateway>[^, ]+)"
| stats count by gateway
| sort -count
```

#### By cluster

You already have a `cluster` field in the screenshot:

```spl
index=am_cds environment=prod source="http:AWM_PROD" "CdsRequestInterceptor" "createDocument" "GUDI="
| stats count by cluster
| sort -count
```

---

### 2.8 Concrete sample events for a few worst offenders

Once you know the top appname:

```spl
index=am_cds environment=prod source="http:AWM_PROD" "CdsRequestInterceptor" "createDocument" "GUDI="
| rex field=log "APPNAME=\[(?:APPNAME=)?(?<appname>[^,\]]+)"
| search appname="BSRWORKFLO"  /* replace with real offender */
| table _time appname cluster hostname clientIp log
| head 20
```

This gives you rows you can redact and paste into a doc / deck.

---

## 3. How this fits your SRE brain

### Data reasoning

* You’re not just saying “this is wrong by design doc”; you’re showing:

  * Scale of the misuse (`has_gudi` vs not).
  * Concentration in a few apps (Pareto pattern).
  * Potential system / user impact (status & latency).
* That’s enough evidence to:

  * Raise a design-governance issue.
  * Propose a **server-side guardrail** (ignore client `GUDI`, log & return 4xx, or strip it before CM).

### Systems analysis

Uncontrolled client-supplied IDs leads to:

* Possible collisions with server-generated GUDIs.
* Clients accidentally assuming **idempotency** or overwriting logic you own.
* Tighter coupling: if you ever change your GUDI format, these clients will break.

You can also flag that **fixing this later** (when more clients depend on it) is way more expensive than addressing it now.

### Validation & simulation

Before reporting, sanity-check:

* Run the “total vs has_gudi” query for **multiple time ranges** (last 24h, 7d, 30d).
* Validate parsing:

  * Spot-check a few events to confirm `APPNAME`, `FID`, `httpStatus`, `GUDI` extractions look right.
* If you have a lower environment, confirm that *your own* test clients are **not** sending GUDI accidentally.

### Ops-ready summary (for your deck / email)

Something like:

* X total `createDocument` calls in last 7 days.
* Y (Z%) calls had a **client-supplied GUDI**, which is against the contract (server generates GUDI).
* Misuse is concentrated in N apps:

  * Top 3 apps: A, B, C (together ~M% of misuse).
* Requests with client GUDI: status distribution mostly 200s → misuse “works” today.
* Risks: ID collision, broken invariants, harder future changes to identifier scheme.
* Proposal:

  * Communicate guideline: **clients must not send GUDI on create**.
  * Plan an enforcement path: log-only → warning → soft-fail → hard-fail, with dates.
  * Provide sample queries to app teams so they can self-check their traffic.

That gives you a clean, data-driven story: “Here is the pattern, here is the risk, here is how we fix it.”
