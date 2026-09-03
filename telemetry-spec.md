# OpsKit Telemetry — Specification v1

**Status:** ready to build
**Audience:** the workspace agent implementing this, and the reviewers who will challenge the numbers it produces
**Owner:** SRE, AWM Shared Services

---

## 1. What this measures and what it must never do

**Purpose.** Determine whether the kit changes two things: how fast engineers reach a reviewable change without shipping more defects, and whether incidents get diagnosed from evidence, faster, and stop recurring.

**Non-goals — these are hard constraints on the implementation, not preferences:**

- **No individual attribution, ever.** Not disabled by policy — made impossible by construction. See §9. If a reviewer asks "can we see engineer X's numbers," the correct answer must be "the system cannot produce that."
- **No code, prompt text, log lines, ticket bodies, or query results leave the laptop.** Only counts, durations, hashes, and categorical enums. See §9.
- **Invocation counts are not a headline metric.** They are diagnostic only. A KR built on usage volume will be gamed within one quarter and tells you nothing about value.
- **No silent collection.** Any engineer can run `opskit telemetry inspect` and see the exact payload queued for upload, in full.

---

## 2. Measurement design — build this first

Everything downstream is worthless without a comparison group. The implementation must support the design below from day one, because cohort assignment has to be recorded at the moment of every event, not reconstructed later.

### 2.1 Stepped-wedge rollout

Squads receive the kit in randomized waves. Each squad contributes a pre-period (control) and a post-period (treatment). At any given week, some squads have it and some don't, giving both within-squad and between-squad comparison.

```
week      1  2  3  4  5  6  7  8  9 10 11 12
wave A    ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██ ██
wave B    ·· ·· ·· ██ ██ ██ ██ ██ ██ ██ ██ ██
wave C    ·· ·· ·· ·· ·· ·· ██ ██ ██ ██ ██ ██
wave D    ·· ·· ·· ·· ·· ·· ·· ·· ·· ██ ██ ██
          ── control ──  ── treatment ──
```

**Requirements on the implementation:**

- Randomize at **squad** level, not engineer level. Engineers on the same squad talk, pair, and review each other's PRs; engineer-level assignment leaks treatment into control and biases the effect toward zero.
- Wave assignment is deterministic: `wave = waves[ HMAC(assignment_salt, squad_id) mod 4 ]`. Recorded once, emitted in the `cohort.assigned` event, and stamped on every subsequent event as `cohort.wave` and `cohort.phase` (`pre` | `post`).
- **Minimum 4 weeks of pre-period per squad.** A squad that installs the kit and starts emitting on day one produces no usable baseline.
- Record `cohort.installed_at` and `cohort.wave_scheduled_at` separately. Squads that install early are a known contamination source and must be identifiable at analysis time.

### 2.2 Baseline sources that do not depend on the kit

The pre-period needs data the kit isn't collecting yet. Pull it from systems of record, not from the kit:

| Metric family | Source | Extraction |
|---|---|---|
| PR cycle time, size, review iterations | Bitbucket API | batch job, 90 days retrospective |
| Rework / hotfix rate | Bitbucket + CD pipeline | batch job |
| Incident timings, CI, category, resolution | ServiceNow | batch job, 180 days retrospective |
| Incident recurrence | ServiceNow, grouped by CI + failure signature | batch job |

**This backfill job is part of the deliverable.** Without it there is no pre-period and the stepped wedge collapses into a pre/post comparison with no control.

---

## 3. The trace model

Treat every unit of work as a trace. This is the primitive that makes the joins possible; without it you have disconnected counters and no way to attribute anything.

```
SDLC trace                          Ops trace
──────────────────────────          ──────────────────────────
trace_id = work_id                  trace_id = INC number
  (ticket key, or normalized          (from ServiceNow)
   branch name if none)

  ├─ span: skill.spec_draft         ├─ span: skill.triage
  ├─ span: skill.scaffold           ├─ span: skill.splunk_query
  ├─ span: skill.test_gen           │    └─ evidence: SPL-a41f
  ├─ span: skill.review_prep        ├─ span: skill.dt_entity_lookup
  │                                 │    └─ evidence: DT-9c02
  ├─ event: git.pr_opened           ├─ span: skill.hypothesis
  └─ event: git.pr_merged           ├─ event: snow.state_change
                                    └─ event: snow.resolved
```

**Correlation rules the implementation must enforce:**

- `work_id` for SDLC: ticket key parsed from the branch name if present, else `sha256(repo + normalized_branch)[:16]`. Normalization strips personal prefixes (`jn/`, `feature/`) so the same work is one trace across renames.
- `work_id` for ops: the INC number, read from whatever context the engineer is working in — the kit already knows the ServiceNow queue from the manifest.
- A session with no resolvable `work_id` still emits events, tagged `work_id: "unattributed"`. Do not drop them; the unattributed share is itself a signal about how the kit is being used.
- Every span carries `parent_span_id` when invoked from within another skill. Nested invocation depth is a quality signal.
- **Evidence artifacts get stable IDs.** When a skill produces a Splunk result, a Dynatrace entity snapshot, or a trace link, it mints `evidence_id = sha256(kind + query_hash + timestamp)[:12]` and records it. This is the mechanism that makes §7.4 possible.

---

## 4. Event schema v1

NDJSON, one event per line. Every line carries the envelope; `payload` varies by `event_type`.

```json
{
  "schema_version": "1.0.0",
  "event_id": "01JQ8F2K3M4N5P6Q7R8S9T0V",
  "event_type": "skill.completed",
  "emitted_at": "2026-09-03T09:41:22.418Z",
  "engineer_pseudonym": "eng_7f3a91c4d2e08b56",
  "squad_id": "awm-shared-svcs-04",
  "cohort": { "wave": "B", "phase": "post", "installed_at": "2026-07-14" },
  "kit_version": "1.4.2",
  "trace_id": "AWMPOS-4821",
  "span_id": "sp_9c02a41f",
  "parent_span_id": null,
  "work_kind": "sdlc",
  "payload": { }
}
```

### 4.1 Event catalogue

**Lifecycle**

| `event_type` | When | Key payload fields |
|---|---|---|
| `cohort.assigned` | first run after install | `wave`, `squad_id`, `assignment_hash` |
| `session.started` | workspace opened with kit active | `repo_hash`, `manifest_version` |
| `session.ended` | workspace closed | `duration_ms`, `spans_emitted` |

**Skill usage — the engagement layer**

| `event_type` | When | Key payload fields |
|---|---|---|
| `skill.invoked` | skill starts | `skill_id`, `trigger` (`explicit` \| `suggested` \| `chained`), `context_bytes` |
| `skill.completed` | skill returns | `skill_id`, `duration_ms`, `outcome` (`success` \| `partial` \| `error`), `error_class` |
| `skill.output_disposition` | resolved at commit or session end | `skill_id`, `disposition`, `retention_ratio`, `time_to_disposition_ms` |
| `skill.abandoned` | invoked, never resolved | `skill_id`, `elapsed_ms` |

`disposition` ∈ `accepted_verbatim` | `accepted_edited` | `discarded` | `superseded` | `unresolved`

`retention_ratio` — token-level similarity between what the skill produced and what was committed, in `[0,1]`. Compute with a normalized token-set overlap after stripping whitespace and comments. **The ratio ships; the text does not.**

**Binding health — is the kit's manifest still true**

| `event_type` | When | Key payload fields |
|---|---|---|
| `binding.resolved` | manifest entity/index/CI resolves against live API | `binding_kind`, `latency_ms` |
| `binding.failed` | it does not | `binding_kind`, `failure_class` (`not_found` \| `auth` \| `timeout` \| `throttled`) |

**Evidence**

| `event_type` | When | Key payload fields |
|---|---|---|
| `evidence.emitted` | skill produces a citable artifact | `evidence_id`, `kind` (`splunk_result` \| `dt_entity` \| `dt_trace` \| `snow_record` \| `code_ref`), `row_count`, `source_latency_ms` |
| `evidence.referenced` | artifact appears in a PR description, commit message, or SNOW work note | `evidence_id`, `surface` |

`evidence.referenced` is the highest-value event in the schema. It is behavioural proof that skill output made it into the record, and it is what replaces self-reported attribution.

**SDLC checkpoints**

| `event_type` | Key payload fields |
|---|---|
| `git.branch_created` | `work_id`, `base_sha_age_days` |
| `git.commit` | `files_changed`, `lines_added`, `lines_removed`, `assisted_hunk_ratio` |
| `git.pr_opened` | `pr_id_hash`, `size_lines`, `files`, `description_has_evidence` |
| `git.pr_review_cycle` | `iteration_n`, `comments_count`, `hours_since_open` |
| `git.pr_merged` | `total_iterations`, `hours_open` |
| `git.rework_detected` | `original_pr_hash`, `days_since_merge`, `lines_rewritten` |

**Ops checkpoints**

| `event_type` | Key payload fields |
|---|---|
| `inc.context_loaded` | `inc_number`, `priority`, `ci_hash`, `minutes_since_assigned` |
| `inc.diagnostic_action` | `action_kind` (`splunk` \| `dynatrace` \| `snow_history` \| `code_read` \| `runbook`), `sequence_n`, `source_count_so_far` |
| `inc.hypothesis_recorded` | `evidence_ids[]`, `minutes_since_context_loaded` |
| `inc.mitigation_applied` | `mitigation_class`, `minutes_since_context_loaded` |
| `inc.resolution_submitted` | `evidence_ids[]`, `rca_has_signature`, `close_code` |

---

## 5. Metric tree

Four layers. Each layer gates the next: if the layer below is failing, the layer above is noise.

```
L4  Portfolio outcome ──── does this matter to AWM
     ├─ change-related incident rate
     └─ repeat-incident rate per CI

L3  Work outcome ───────── did the work get better
     ├─ branch → PR-ready duration
     ├─ review iterations to merge
     ├─ 14-day rework rate
     ├─ time to first diagnostic action
     └─ evidence-backed RCA rate

L2  Engagement quality ─── is it used well
     ├─ acceptance rate (disposition ≠ discarded)
     ├─ retention ratio distribution
     ├─ diagnostic source breadth
     └─ abandonment rate

L1  Adoption ──────────── is it used at all
     ├─ weekly active squads
     ├─ % of work items with ≥1 span
     └─ skill coverage (distinct skills / available)

L0  Kit health ────────── does it work
     ├─ skill p95 latency, error rate
     └─ binding failure rate
```

**Gating rule for analysis:** do not report L3 or L4 for a squad whose L2 acceptance rate is below 0.40. Below that threshold the kit isn't really being used and any outcome difference is confounded.

---

## 6. Objectives and key results

### O1 — Engineers reach a reviewable change faster, without shipping more defects

| KR | Definition | Target |
|---|---|---|
| KR1.1 | Median `git.branch_created` → `git.pr_opened`, treatment vs control, same squad | −35% |
| KR1.2 | Median review iterations to merge | ≤ control (guardrail) |
| KR1.3 | 14-day rework rate | ≤ control (guardrail, strict) |
| KR1.4 | PR size, median lines | ≤ control × 1.2 (guardrail) |
| KR1.5 | Skill acceptance rate | ≥ 0.55 |

KR1.2–1.4 are **counter-metrics**, not improvement targets. Their job is to catch the failure mode where the kit makes engineers faster at producing larger, worse changes. A KR1.1 win with a KR1.3 regression is a net loss and must be reported as one.

### O2 — Incidents are diagnosed from evidence, faster, and stop recurring

| KR | Definition | Target |
|---|---|---|
| KR2.1 | Median `inc.context_loaded` → first `inc.diagnostic_action` | −50% |
| KR2.2 | % incidents where ≥2 distinct `action_kind` precede `inc.hypothesis_recorded` | ≥ 0.80 |
| KR2.3 | 30-day recurrence rate on the same `ci_hash` + failure signature | −25% |
| KR2.4 | % resolutions carrying ≥3 `evidence_ids` that trace to real artifacts | ≥ 0.70 |
| KR2.5 | P1/P2 median time-to-mitigate | −20% |

**KR2.3 is the root-cause metric.** "The skill helped find the root cause" is unverifiable and self-report will inflate it. Recurrence is observable, hard to game, and is what root-cause analysis is actually for.

**Do not use MTTR as a headline.** The distribution is heavily skewed, one 14-hour incident dominates a month, and severity mix shifts confound it. Report time-to-mitigate as a percentile distribution (p50/p75/p90) with the severity mix stated alongside.

### O3 — The kit behaves like a service

| KR | Definition | Target |
|---|---|---|
| KR3.1 | Skill p95 execution latency | < 4s |
| KR3.2 | Skill error rate | < 2% |
| KR3.3 | Binding failure rate (manifest drift) | < 5% of resolutions |
| KR3.4 | Weekly active squads / installed squads | ≥ 0.70 |

O3 is the leading indicator. O1 and O2 will not move if O3 is failing, and O3 fails silently — engineers stop invoking a slow or broken skill rather than reporting it.

---

## 7. Metric definitions that need care

### 7.1 Branch → PR-ready, not full lead time

Measure only the segment the kit can affect. Full lead-time-to-production includes review queue latency, change approval, and release windows — human and process queues that will swamp the signal and that the kit does not touch.

**Exclusions:** branches idle >72h (context-switched work), branches with zero commits, reverts, dependency-bump automation.

### 7.2 Acceptance rate is the cheapest high-signal metric

```
acceptance_rate = count(disposition ∈ {accepted_verbatim, accepted_edited})
                  / count(disposition ≠ unresolved)
```

Track the `retention_ratio` distribution, not just the mean. A bimodal distribution — mostly 1.0 and mostly 0.1 — means some skills are excellent and others are being politely ignored. The mean hides that and would tell you the kit is mediocre everywhere.

### 7.3 Diagnostic source breadth

One of the kit's real values is that a junior engineer consults Splunk *and* Dynatrace instead of only the one they know. Count distinct `action_kind` before the first recorded hypothesis. Rising breadth with falling time-to-hypothesis is the signature of the kit working as intended.

### 7.4 Evidence provenance, not self-report

An RCA "counts" as evidence-backed only if its cited `evidence_ids` resolve to artifacts the kit actually minted during that incident's trace, with timestamps preceding the resolution. The verification job re-checks this at aggregation time and emits `evidence_verified` / `evidence_orphaned`. An orphan rate above 10% means people are pasting IDs to satisfy the metric, and the metric should be suspended.

### 7.5 Recurrence signature

```
signature = sha256( ci_hash + close_code + normalized_symptom_class )
```

`normalized_symptom_class` comes from a fixed taxonomy the kit provides at resolution time (latency, saturation, dependency failure, data quality, config, capacity, auth). Free-text symptom fields do not cluster and must not be used.

---

## 8. Local architecture

```
~/.opskit/telemetry/                    ← home dir, NOT the repo
  ├── config.json                       ← pseudonym, salt, squad, wave, sync settings
  ├── spool/
  │   └── 2026-09-03.ndjson             ← append-only, daily rotation
  ├── staged/
  │   └── 2026-09-02.ndjson.gz          ← redacted, validated, ready to ship
  └── state.json                        ← last sync, watermark, drop counters
```

**Why the home directory and not the repo:** a telemetry directory inside a working tree will be committed by accident within a week, and one of those commits will contain something that should not be in a shared repo.

**Write path**

1. Skill or hook emits an event to an in-process ring buffer.
2. Flusher appends to today's spool file every 5s or 64 events, whichever first. Never blocks the IDE — if the buffer is full, drop and increment `telemetry.dropped`.
3. Nightly: rotate, run the redaction validator (§9), gzip into `staged/`.
4. Sync job ships `staged/` and deletes on confirmed push.

**Failure behaviour**

- Spool capped at 50 MB. Beyond that, drop oldest and record the drop count — the count ships even when the events don't, so gaps are visible rather than invisible.
- Sync failure never retries more than 3× per window. A Bitbucket outage must not turn into 200 laptops retrying in a loop.
- The kit must function fully with telemetry disabled. Telemetry is never on the critical path of a skill invocation.

**Kill switch**

```
opskit telemetry status       # on/off, last sync, queued events, drop count
opskit telemetry inspect      # print the exact staged payload, unredacted view of what ships
opskit telemetry disable      # local opt-out, effective immediately, no restart
```

---

## 9. Redaction — enforced, not documented

A validator runs over every staged file before it can be shipped. **A file that fails validation is never sent; it is quarantined and a `telemetry.validation_failed` counter is emitted.**

**Allowlist model.** Fields not in the schema are stripped, not passed through. Schema-driven, so adding a field requires a schema version bump and a review.

**Never emitted, under any circumstance:**

- Source code, diffs, or hunks — only counts and ratios
- Prompt text, skill output text, or model responses
- Log lines, log field values, or query result rows — only row counts
- Incident short descriptions, work notes, or resolution text
- File paths, branch names, repo names, PR titles — hashed only
- Corporate IDs, email addresses, display names, hostnames

**Structural anonymity.** This is the load-bearing control:

```
engineer_pseudonym = "eng_" + HMAC-SHA256(device_local_salt, corporate_id)[:16]
```

The salt is generated on first install, stored only on the device, and **never transmitted**. No mapping from pseudonym to person exists anywhere off the laptop. Consequence: per-engineer performance reporting is not a policy the org chooses to follow, it is a query nobody can run. State this explicitly in the rollout comms — it is the reason engineers will let the thing collect anything at all.

Trade-off to accept: a laptop reimage produces a new pseudonym and breaks that engineer's longitudinal series. That is the correct trade. Do not add a recovery mechanism.

**Suppression floor.** No aggregate is published for a group with fewer than 5 distinct pseudonyms in the window. This blocks re-identification of small squads by inference.

---

## 10. Sync to Bitbucket

The branch is acceptable as **transport**. It is not a store — §11 drains it into something queryable.

**Layout — one file per pseudonym per day, so writers never touch the same path and merges are always clean:**

```
branch: telemetry-ingest          (orphan branch, no shared history with main)

telemetry/
  v1/
    dt=2026-09-02/
      eng_7f3a91c4d2e08b56.ndjson.gz
      eng_a02b18ff43c9d771.ndjson.gz
```

**Protocol**

1. Shallow-clone or fetch the orphan branch to a local cache (`--depth=1`, single branch).
2. Write the file at its unique path. Never modify an existing path.
3. `commit` with a fixed message template: `telemetry: v1 dt=<date>`
4. `push`. On non-fast-forward: `pull --rebase`, then retry, max 3 attempts. Because paths are disjoint per writer, rebase is always conflict-free.
5. On success, delete the staged local file and advance the watermark.

**Scheduling — this is where a naive implementation breaks the git server.** With 100+ engineers, a fixed daily time produces a thundering herd against Bitbucket.

- Sync window: 02:00–06:00 local
- Offset: `HMAC(pseudonym, "sync") mod 14400` seconds into the window — deterministic per device, uniformly spread
- Skip if on VPN-less network, battery below 20%, or the laptop is asleep; carry to the next window
- Hard cap: one sync attempt per 12h per device

**Repo hygiene**

- Orphan branch keeps blobs out of `main`'s history entirely
- Files are gzipped NDJSON, typical daily size per engineer under 40 KB
- A retention job (§11) deletes partitions older than 90 days and force-pushes the pruned branch monthly, in a maintenance window
- Estimated steady state: 150 engineers × 40 KB × 90 days ≈ 540 MB. Set a repo size alert at 1 GB.

**Credentials.** The sync job uses a dedicated service account with **write-only access to the `telemetry-ingest` branch** via a branch permission rule, and no read access to anything else. A personal access token with broad repo scope sitting on 150 laptops is the sort of thing that ends this project.

**Anticipate the InfoSec conversation.** An automated process that writes data from developer laptops to a repository will be read as a potential exfiltration channel. Bring §9's allowlist validator, the write-only scoped account, and the `telemetry inspect` command to that meeting. Get the design reviewed before the first engineer installs it, not after.

---

## 11. Aggregation and analysis

Runs centrally, not on laptops.

**Ingest job** (daily) — drain `telemetry-ingest` into a queryable store (DuckDB over object storage is sufficient at this volume and avoids a platform request). Validate schema, quarantine malformed lines, deduplicate on `event_id`.

**Trace assembly** (daily) — group events by `trace_id`, reconstruct spans, join to the Bitbucket and ServiceNow backfill data on `work_id` / `inc_number`. Emit one row per work item with all L3 metrics precomputed.

**Evidence verification** (daily) — the §7.4 check. Emit orphan rates.

**Cohort analysis** (weekly) — difference-in-differences across the stepped wedge, by squad and wave. Report effect sizes with confidence intervals, never point estimates. Flag squads with pre-period under 4 weeks or acceptance rate under 0.40 and exclude them from the headline with the exclusion stated.

**Scorecard** (weekly) — squad-level, suppression floor applied, every velocity KR shown adjacent to its paired counter-metric. Same page, same table. Never a velocity chart on its own slide.

---

## 12. Known ways this measurement program fails

Build the detection for each of these; do not assume they won't happen.

| Failure | Detection |
|---|---|
| Contamination — control squads adopt early | `installed_at` earlier than `wave_scheduled_at`; report contaminated share every week |
| Novelty effect — usage spikes then decays | Segment all L3 metrics by weeks-since-install; a 4-week effect that vanishes by week 10 is not an effect |
| Selection within squad — only strong engineers use it | Compare distribution of pseudonyms contributing spans against squad headcount |
| Metric gaming after the OKRs are published | Watch for step changes in acceptance rate or evidence citation with no matching change in outcomes; rising `evidence_orphaned` |
| Severity mix shift on ops metrics | Always report incident counts by priority alongside timing metrics |
| Survivorship — abandoned work items never emit `pr_opened` | Count branches created without a terminal event; a rising abandonment rate can masquerade as improved cycle time |
| Small-N noise on recurrence | KR2.3 needs ≥30 incidents per arm per window. Below that, report the raw count, not a rate |

---

## 13. Delivery phases

**Phase 1 — instrument (weeks 1–3)**
Event schema, local spool, redaction validator, `telemetry status/inspect/disable`. No sync yet. Ships with the kit and writes locally so the schema can be validated against real usage before any data moves.

**Phase 2 — backfill (weeks 2–4, parallel)**
Bitbucket and ServiceNow historical extraction. Establishes pre-period for every squad. This is the piece most likely to be deprioritised and the piece without which nothing else works.

**Phase 3 — sync (weeks 4–6)**
Bitbucket transport, scheduling with jitter, ingest and dedup job. InfoSec review completed before this ships.

**Phase 4 — analysis (weeks 6–9)**
Trace assembly, evidence verification, difference-in-differences, weekly scorecard.

**Phase 5 — read out (week 12+)**
First defensible effect estimate. Not before. Anything reported in weeks 1–8 is adoption data, and it must be labelled as adoption data.

---

## 14. Acceptance criteria

- [ ] Every event validates against schema v1; unknown fields are stripped, not forwarded
- [ ] Redaction validator rejects a crafted file containing code, log text, or a corporate ID — with a test that proves it
- [ ] No pseudonym-to-identity mapping exists in any artifact the sync produces
- [ ] `telemetry inspect` output is byte-identical to what the sync ships
- [ ] Kit functions with telemetry disabled; no skill blocks on a telemetry call
- [ ] 200 simulated concurrent syncs complete without a merge conflict
- [ ] Spool survives kill -9 mid-write without corrupting the NDJSON file
- [ ] Backfill reproduces PR cycle time for a known historical quarter within 2% of a manual count
- [ ] Trace assembly correctly joins a synthetic incident from `inc.context_loaded` through `inc.resolution_submitted`
- [ ] Difference-in-differences job runs against synthetic data with a known injected effect and recovers it within the stated confidence interval
- [ ] Scorecard renders no velocity metric without its paired counter-metric

---

## 15. Decisions still open

1. **Ticket system for `work_id`.** Jira, ServiceNow demand records, or branch-name convention? Determines the SDLC join key and cannot be changed later without breaking the series.
2. **Squad roster source.** Where does `squad_id` come from, and what happens when someone moves squads mid-wedge? Recommend: freeze cohort at assignment, record moves as an exclusion flag.
3. **Wave count.** Four waves over 12 weeks assumes ~30 squads. Fewer squads means fewer, larger waves and wider confidence intervals — decide the wave structure before Phase 1 ships.
4. **Retention beyond 90 days.** The recurrence metric wants a 30-day trailing window, so 90 days is the floor. Longitudinal analysis across quarters would want a year, which pushes the Bitbucket branch past its comfortable size and forces the move to a real store earlier.
5. **Whether `evidence.referenced` can be detected in ServiceNow work notes** without reading the note text. Options: the kit writes the note itself and records that it did, or the verification job runs server-side inside the ServiceNow boundary. The first is simpler and is what the schema currently assumes.
