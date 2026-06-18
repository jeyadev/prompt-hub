---
name: capacity-management-strategy
description: >
  Full capacity-management strategy context for an infrastructure cost-optimization agent in a
  regulated, private-cloud (JPMC Gaia/CPSR) environment. PART A is a complete, workload-agnostic
  capacity-management discipline (demand modelling, forecasting, headroom math, scaling strategy,
  queueing limits, maturity model, cost integration). PART B is a dedicated Elasticsearch section
  that applies Part A to ES on JPMC's managed Gaia Elasticsearch platform (tenant = indexes +
  mappings; the ES estate belongs to the separate Full Text Search app, so posture is advisory).
  Use for capacity planning/forecasting, rightsizing, headroom/scaling policy, demand shaping,
  cost allocation, and ES sizing/tiering — without breaking reliability SLAs.
version: 2.0
audience: Cline / Roo FinOps mode (DevGPT Cline, JPMC)
---

# Capacity Management Strategy — with a dedicated Elasticsearch section

> Wire-in: load as the FinOps/capacity mode's primary context (skill or `customInstructions`).
> Reference + decision logic, not a runbook to execute blindly. Every capacity action is a change
> to a shared, reliability-sensitive system — treat it as such.
>
> **How this file is organized:**
> **PART A** — the general capacity-management strategy (applies to any workload).
> **PART B** — Elasticsearch, as a self-contained section that instantiates Part A for ES.
> Read Part A first; Part B assumes its vocabulary (demand/supply, headroom, the loop, the contract).

---
---

# PART A — CAPACITY MANAGEMENT STRATEGY (general)

## A0. PRIME DIRECTIVE

**Reliability-first cost optimization.** Capacity management's job is to supply *the least capacity
that still meets the SLA at forecast peak* — never the least capacity. Under-provisioning trades a
visible bill for an invisible, larger one: incidents + emergency (premium, lead-time-bound) capacity.
Over-provisioning is quiet waste. Top-tier capacity management lives between those, with the gap
**measured and defended**, not guessed.

Lever order for every recommendation (cheapest/safest first):
1. **Eliminate** — reclaim idle/zombie/stranded capacity. Zero risk; always first.
2. **Rightsize** — match resource to real demand + headroom. Low risk, reversible.
3. **Reshape demand** — shift / smooth / shed / cap load. Often cheaper than buying capacity.
4. **Re-rate** — commitments/discounts. Mostly N/A on private cloud (§A10).

Never invert. Re-rating or scaling an un-rightsized, un-reshaped workload locks in waste.

## A1. WHAT CAPACITY MANAGEMENT IS (and isn't)

Capacity management = **matching the supply of resources to the demand for them, over time, at minimum
cost subject to reliability constraints.** It is the *upstream* discipline; cost and reliability are
its downstream outputs.

| Not this | This |
|---|---|
| Monitoring (tells you you're out of capacity *now*) | Forecasting (tells you *when* you will be) |
| Cost management (reports spend) | Capacity management (drives the spend by sizing decisions) |
| Autoscaling alone (reacts to load) | Strategy: demand model + supply plan + headroom policy + reconciliation |
| A quarterly review | A continuous loop (§A2) |

## A2. THE CAPACITY MANAGEMENT LOOP (run continuously)

```
MEASURE demand + utilization (multi-dimensional)
  → MODEL/FORECAST peak demand (with seasonality + growth + error band)
    → PLAN supply = forecast peak + headroom (§A5.3), respecting lead time (§A5.4)
      → PROVISION / rightsize (through change management)
        → RECONCILE: did utilization land in the target band? did the saving hit the bill?
          → feed back
```
Delayed feedback is the killer: a change can take days to show load-driven degradation. Forecast the
**peak**, watch lagging indicators, and keep the loop turning faster than demand changes.

## A3. THREE LAYERS OF CAPACITY (ITIL backbone — useful for governance framing)

Top-tier programs manage all three; most teams only do the third.

| Layer | Question | Owner-facing artifact |
|---|---|---|
| **Business capacity** | What business growth/events drive demand? (accounts, txns, docs, users) | Driver → demand model (§A4.2) |
| **Service capacity** | Can the end-to-end service meet its SLA at that demand? | SLO-vs-forecast capacity plan (§A6) |
| **Component/resource capacity** | Is each resource (CPU/mem/disk/IOPS/heap/shards/licenses) sized right? | Rightsizing + headroom per resource (§A5) |

The leap from component to business/service capacity is what separates top-1% from reactive shops:
you size to **forecast business demand against the SLA**, not to last week's CPU graph.

## A4. DEMAND SIDE

### A4.1 Characterize demand before sizing supply
- **Workload class** decides the binding resource and scaling shape:
  - *Latency-sensitive interactive* (APIs, search): sized by **peak concurrency / queueing knee** (§A6), not average util.
  - *Throughput batch*: sized by **deadline ÷ work**; tolerant of queueing and off-peak shifting.
  - *Storage/state-bound* (search indices, DBs): sized by **data volume, growth, IOPS, working-set vs cache**.
- **Demand drivers**: map a *business* metric to *technical* demand (e.g. documents archived/day → index growth + ingest rate). Driver-based beats blind trend extrapolation.
- **Demand patterns**: diurnal, weekly, seasonal, event-driven (quarter-end, regulatory cycles), and the underlying growth trend. Size to the *envelope*, not the mean.

### A4.2 Forecasting (the core skill)
| Method | Use when | Watch-out |
|---|---|---|
| Trend extrapolation | Stable linear growth | Misses seasonality + step changes |
| Seasonal decomposition (e.g. STL/Holt-Winters/Greykite) | Clear daily/weekly/seasonal shape | Needs enough history |
| **Driver-based regression** | Demand tracks a business metric | Requires a stable, available driver signal |
| Scenario / what-if | Migrations, launches, events | Bounds, not point estimates |

Rules: **forecast the peak (high percentile), not the average.** Carry an **error band** and convert it
into headroom (§A5.3) — a forecast without uncertainty is a guess with false confidence. Pick the
**horizon to the lead time**: you must forecast far enough ahead to clear the procurement/provisioning
pipeline (§A5.4).

### A4.3 Demand shaping — usually cheaper than buying capacity
Before adding supply, ask whether you can change the demand curve:
- **Shift** — move deferrable/batch work into troughs (off-peak windows).
- **Smooth** — rate-limit, queue, backpressure to clip peaks into the existing envelope.
- **Shed** — load-shed / graceful degradation to protect the SLA under overload (a capacity decision, not just resilience).
- **Cap** — quotas / tenant limits so one consumer can't define everyone's capacity.
Demand shaping moves the *peak* you must provision for, which is the number that actually costs money.

## A5. SUPPLY SIDE

### A5.1 Resource types, elasticity, lead time
Size every binding resource, not just CPU. Each has different elasticity and **lead time** (how long to
add more) — lead time is a first-class planning input on private/constrained infra.

| Resource | Typically binds | Elasticity on private cloud |
|---|---|---|
| CPU | Compute-bound services | Medium (request pipeline) |
| Memory / cache | Search, DBs, JVM workloads | Medium |
| Storage capacity | Stateful/data systems | Slow (procurement) |
| IOPS / disk throughput | Hot data paths | Slow / fixed by tier |
| Network | Chatty/distributed | Slow |
| **Licenses / quota** | Commercial software, platform tenancy | Procurement + contract |

### A5.2 Scaling strategy
| Strategy | Best for | Cost/risk note |
|---|---|---|
| **Vertical** (bigger node) | Single-instance bottlenecks; quick wins | Hits a ceiling; bigger blast radius per node |
| **Horizontal** (more nodes) | Stateless/shardable; the default for scale | Subject to USL diminishing returns (§A6) |
| **Autoscaling — reactive** (threshold) | Variable, fast-changing load | Lags spikes; can thrash; needs warm headroom |
| **Autoscaling — predictive/scheduled** | Known patterns + lead time | Top-tier; pre-sizes before the peak arrives |

Default: a **right-sized baseline + elasticity for the variable part.** Static provisioning for dynamic
load is the most common waste; pure reactive scaling for spiky load is the most common incident.

### A5.3 Headroom / buffer policy (compute it, don't guess "80%")
Headroom is reserved capacity above forecast peak. It exists for four distinct reasons — size each:
```
headroom ≥ failure-domain reserve (N+1 / N+2 — survive node/AZ loss)
        + forecast-error band (from A4.2)
        + burst/variability above the peak point estimate
        + lead-time reserve (demand growth during the time it takes to add capacity, A5.4)
```
Then bound the **target utilization** below the queueing knee (§A6) — for latency-sensitive systems
that's often well under 100% (e.g. a 60–75% steady band), not because of a magic number but because
latency explodes near saturation. **"Wasted" headroom may be load-bearing** (failover, seasonal, lead
time) — confirm a buffer is truly idle before reclaiming it. Removing failover headroom to cut cost is
how capacity optimization causes the next outage (Safety-II: you removed the slack that was absorbing failures).

### A5.4 Provisioning model + the private-cloud lead-time problem
Public cloud hides lead time behind an API; private/constrained infra does not. You must **forecast past
the lead time** of the slowest binding resource (often storage or licensed capacity) and keep a
**lead-time reserve** so you don't hit the wall mid-procurement. Treat capacity as **capacity-as-code /
a tracked request pipeline**, not ad-hoc tickets, so lead times are visible and plannable.

## A6. THE RELIABILITY LINK (why you can't just run hot)

Capacity is a reliability discipline. The failure modes are non-linear:
- **Utilization → latency knee (queueing theory):** as utilization → 100%, queue time → ∞. Latency
  rises gently, then explodes near saturation. This is *why* the headroom band exists — you size below
  the knee, not near it.
- **Little's Law (L = λW):** concurrency = arrival-rate × time-in-system. Use it to size pools/threads
  from measured λ and W instead of guessing.
- **Universal Scalability Law:** horizontal scaling has diminishing returns from **contention** and can
  go *negative* from **coherency/coordination** cost. More nodes ≠ proportional throughput — measure the
  scaling curve before assuming linear scale-out.
- **Capacity cliffs / metastable failure:** systems run fine until a threshold, then fall off a cliff
  (cache thrash, GC death-spiral, retry storms). Running with thin headroom puts you near the cliff edge.
- **SLO-driven sizing:** size to meet the SLO **at forecast peak**, and spend the error budget
  deliberately — capacity is the lever that converts budget into either savings (run leaner) or
  reliability (more headroom).

## A7. THE COST LINK (FinOps integration)

Capacity decisions *are* the cost. Tie them to money explicitly.
- **Rightsizing** is the core cost-capacity action — and it's **multi-dimensional**: rightsizing a
  memory-bound workload on CPU cuts the wrong dimension. Always identify the binding resource first.
- **Unit economics:** track cost per unit of demand (per txn / per document / per GB-month / per 1M API
  calls), not absolute spend. Rising spend with falling unit cost = healthy growth.
- **Allocation maturity:** Showback (teams *see* cost) → Chargeback (teams are *billed*). Unallocated
  shared cost kills accountability. Accurate allocation precedes chargeback.
- **FinOps framework (supporting taxonomy):** phases **Inform → Optimize → Operate**; optimization splits
  into **rate** (discounts/commitments) and **usage/workload** (rightsizing, scheduling, tiering,
  eliminate). On private cloud the durable savings are almost entirely **usage/workload** (§A10).
- **Effective Savings Rate / verify-it-landed:** only count savings that show up on the bill/chargeback.
  Dashboard savings that never reach the invoice didn't happen.

## A8. CAPACITY MATURITY LADDER (push every workload up)

| Rung | Practice | Tell |
|---|---|---|
| 0 Reactive | Scale when it breaks; pad "to be safe" | Capacity incidents **and** idle waste coexist |
| 1 Scheduled | Periodic manual rightsizing reviews | Stale recommendations, quarterly cadence |
| 2 Threshold | Auto-actions on utilization/watermarks | Autoscale + watermark alerts |
| 3 **Predictive** | Forecast peak → pre-size with headroom | Forecast plotted vs allocated capacity |
| 4 **Autonomous** | Forecast drives automated rightsizing at scale; humans set policy | Many safe rightsize actions/day; incidents ↓ *and* waste ↓ |

Top 1% = **3–4**: forecast-driven rightsizing with headroom guardrails, which removes toil *and*
prevents capacity incidents at once — they trade off only at low maturity.

## A9. CAPACITY RISK MANAGEMENT

- **Scenario / what-if planning:** peak event, failover (lose a node/AZ), demand spike, supplier/lead-time
  slip, migration cutover. Provision against the credible worst case, not the average day.
- **Stranded-capacity reclamation:** find allocated-but-unused capacity (over-requested, abandoned envs).
- **Zombie elimination:** idle nodes, orphaned volumes, dead indices, unattached resources — pure waste, zero risk.
- **Capacity risk register:** track each binding resource's runway (time-to-exhaustion at forecast growth) and lead time; the gap is your risk.

## A10. JPMC / PRIVATE-CLOUD APPLICABILITY LAYER

The general strategy holds; the available knobs differ on **Gaia/CPSR**. Apply FinOps as a **"Cloud+"
Scope** (the 2025 framework explicitly covers private cloud / data centers).

| Public-cloud assumption | Reality on Gaia/CPSR | Consequence for strategy |
|---|---|---|
| Instant elastic scale | Provisioning lead time / request pipeline | Lead-time reserve + longer forecast horizon (§A5.4) matter more |
| RI / Savings Plans / Spot | ✗ N/A | Savings come from **usage/workload** optimization, not rate (§A7) |
| Provider rightsizing/anomaly APIs | Likely absent | Self-driven loop on Prometheus/Dynatrace + Grafana cost panels |
| Cloud-connected SaaS tooling | Air-gap blocks internet egress | Prefer self-hosted/on-cluster signals; assume external SaaS won't connect |
| Self-service change | Enterprise **change management** gates | Stage in lower env → canary → promote; nothing unilateral on shared svc |

Always also weigh: **shared-service blast radius** (a CDS-class change cascades across AWM),
**audit/compliance** (retention/deletion are regulated — never cut without a confirmed window), and
**team capacity** (a ~10-person team can't run a high-toil scheme; favor rung 3–4 automation over manual reviews).

## A11. OPERATING POSTURE + OUTPUT CONTRACT (general)

- Dense, table-heavy, action-oriented. No filler, no hedging walls.
- **Deterministic before probabilistic:** utilization data, watermarks, ratios, Little's Law decide most
  actions; reserve forecasting/modelling for genuinely variable demand.
- **Quantify or flag:** if a saving/risk can't be tied to a metric, say so. Don't fabricate percentages.
- **Ask, don't assume,** when missing SLA targets, retention windows, demand drivers, or read patterns.

For each recommendation emit:
```
Lever:            <eliminate | rightsize | reshape-demand | scale | re-rate>
Owner:            <who can actually action this>
Mechanism:        <the specific knob/action>
Binding resource: <CPU | memory/cache | storage | IOPS | network | licenses | concurrency>
Est. impact:      <% or absolute + the unit/denominator; modelled vs measured>
Blast radius:     <who/what degrades if it goes wrong>
Reversibility:    <reversible | lossy/one-way | design-dependent>
Rollback:         <concrete rollback step + signal to trigger it>
CM gate:          <whose change management + lower-env stage + canary scope>
Verify:           <the metric that confirms it landed + when it'd show>
```
Lead with highest-impact, lowest-risk. Stack-rank. Separate owned actions from advisory asks.
Exec/governance framing on request: two lines — what it saves, what it risks, the one metric that proves
it. Plain language; "consolidation/orchestration," not "AI cost tool."

## A12. GENERAL ANTI-PATTERNS

| Anti-pattern | Why it bites | Do instead |
|---|---|---|
| Size on average utilization | Misses the peak that causes incidents | Size to forecast peak / percentile (§A4.2) |
| Rightsize on CPU only | Cuts the wrong dimension | Find the binding resource (§A7) |
| Run near 100% to "save" | Latency cliff (§A6) | Stay below the queueing knee |
| Headroom = fixed % guess | Wrong in both directions | Compute it (failure+error+burst+lead time, §A5.3) |
| Reclaim "idle" headroom | It may be failover/seasonal | Confirm it's truly idle first |
| Scale out and assume linear | USL contention/coherency | Measure the scaling curve |
| Commit/scale before rightsizing | Locks in waste | Eliminate→rightsize→reshape→re-rate (§A0) |
| Ignore lead time | Hit the wall mid-procurement | Forecast past lead time + reserve (§A5.4) |
| Savings never reconciled | Never hit the bill | Verify Effective Savings Rate (§A7) |
| Optimize a non-bottleneck | Waste (Goldratt) | Find the constraint first |

---
---

# PART B — ELASTICSEARCH (dedicated section)

> This section applies Part A to Elasticsearch. **Scope reality:** ES runs on JPMC's **managed Gaia
> Elasticsearch** platform (a tenant offering, like Gaia-for-CF / Gaia-for-K8s). The ES estate belongs
> to the **separate Full Text Search (FTS)** application in the document-management ecosystem — **not
> CDS**. Your posture is therefore **advisory / cross-team**, and your tenant surface is **indexes +
> mappings**; the platform owns tiers and nodes.

## B0. OWNERSHIP MODEL — what you can pull vs. what you advise

Do not recommend a platform-owned knob as a tenant action. Frame those as cross-team asks.

| Lever | Owner | Move |
|---|---|---|
| Mappings, what gets indexed, `index:false`, `match_only_text`, norms/`_source` | **Tenant (owned)** | **Primary owned surface** — recommend, FTS team applies |
| Shard/primary count, rollover size, index count, close/delete | **Tenant (owned)** | Right-size; shard sprawl is *your* metered cost |
| Replica count, `best_compression`, force-merge | **Tenant — *if Gaia ES exposes it*** | Confirm exposure; else advisory |
| Retention / delete past compliance window | **Tenant (owned), compliance-gated** | Highest-confidence owned saving |
| Tier placement (hot/warm/cold/frozen), ILM transitions | **Platform-owned** | **Advisory** — biggest saving, you influence not set (§B4) |
| Node hardware, JVM heap, swap, DAS/NAS, page cache | **Platform-owned** | **Literacy** (§B3) — to size requests + root-cause |
| Snapshot repo / Pithos, AutoOps | **Platform-owned** | Advisory (§B6 / §B7) |

**Owned-lever priority order:** bytes-per-document (mappings) → shard/index hygiene → replicas/codec
(if exposed) → retention. Package tiering, sizing, and Pithos as **advisory asks** to the Gaia ES
platform / FTS team. **Verify savings on owned proxies** (`_cat/indices` store size, `_cat/shards`
count, ingest GB/day) that predict the platform's chargeback to FTS — you likely won't see the bill.

## B1. ES CAPACITY MODEL — what consumes capacity (binding resources)

ES cost/capacity is dominated by **how much data sits on hot SSD-backed nodes with replicas**, and by
**shard count** (heap + metadata). Binding resources, in usual order:
1. **Storage volume** (× replicas × tier) — the headline.
2. **OS page cache / RAM** — serves Lucene; the real query-speed resource (not CPU).
3. **JVM heap** — bounded; consumed by shards/field-data/aggregations.
4. **IOPS / disk throughput** — hot indexing + query.
5. **Shard count** — overhead per shard; too many forces bigger/more nodes.

## B2. ES DEMAND SIDE

- **Ingest demand:** documents indexed/sec and bytes/doc → hot-tier storage + indexing CPU/IOPS. For FTS,
  indexing is event-driven and real-time as documents are archived.
- **Query demand:** search QPS and **concurrency** → sized by the queueing knee (§A6), plus aggregation
  heap pressure.
- **Growth:** data volume trend → storage runway + lead time (§A5.4).
- **Read-recency distribution (required input, currently unknown):** how the probability of a document
  being searched decays with age. This sets ILM tier windows.
  - **Banking caveat — "archived ≠ rarely searched."** FTS serves legal hold, audit, and e-discovery,
    which routinely full-text search *old* documents. If old-doc search is material, an aggressive
    hot→frozen policy puts the slowest tier under the queries that matter most — a reliability own-goal
    dressed as a saving.
  - **How to get it (deterministic, cheap):** mine FTS query/slow logs for *query timestamp vs matched-
    document age* — the same query-corpus analysis pattern already run for the Pithos migration. Output a
    data-age × query-frequency heatmap → defensible `min_age` windows + the floor below which data must
    stay warm/cold, not frozen. **Until this exists, tier windows are blocked — state the assumption, don't guess.**

## B3. ES SUPPLY / SIZING — guardrails (PLATFORM-OWNED → capacity literacy)

You don't tune these on Gaia ES; use them to size your cluster/index **request**, to root-cause "my
mappings vs their nodes," and to make a credible advisory ask. The one effectively yours: **shard count**
(your index/rollover design creates the shards the platform must host).

- **JVM heap ≤ 50% of node RAM**, **Xms = Xmx**; the other half is OS page cache (what makes Lucene fast).
- **Heap ≤ ~30 GB** to keep Compressed OOPs; beyond that, *effective* heap drops and GC pauses grow → add nodes/cache, don't grow heap.
- **Disable swap** (swapping wrecks ES latency non-linearly).
- **Shard size 10–50 GB**, **< 200M docs/shard**; too-small wastes heap/metadata, too-large slows query + recovery.
- **≤ 1,000 shards per non-frozen data node.** ILM creates shards automatically — a daily 5-primary×1-replica rollover = 10 shards/day fills a node in ~100 days unattended. Widen rollover or cut primaries before adding nodes.
- **Master nodes ~1 GB heap / 3,000 indices.** (The old "20 shards/GB heap" rule is retired in 8.3+.)
- **Hot = local SSD/DAS; never NAS** (latency + POSIX-semantics corruption risk).
- Sizing ceiling is **access-pattern bound**: a cold/audit node at ~20 TB on 31 GB heap is fine; the same spec under high-concurrency search falls over. **Benchmark with Rally against FTS's own data** before sizing — flag as an advisory ask if the cluster looks mis-sized.

## B4. ES DEMAND SHAPING / DATA LIFECYCLE — tiers + ILM (PLATFORM-OWNED → advisory)

The ES form of demand shaping (§A4.3) is **moving colder data to cheaper storage**. Biggest saving, but
platform-owned — package as the top cross-team ask, backed by the §B2 read-recency evidence.

Tiering economics (Elastic Search Labs benchmark, 90 TB): **all-hot ≈ $28,222/mo → hot+frozen ≈ $3,290/mo (~88% ↓)** — the headline number for governance.

| Tier | Storage | Replicas | Rel. cost | Query | Notes |
|---|---|---|---|---|---|
| Hot | local SSD/DAS | 1(–2) | baseline | fastest | active indexing + recent reads; never NAS |
| Warm | HDD ok | ≥1 | lower | slower | recent weeks |
| Cold | searchable snapshot (fully mounted) | **0** | ~50% < warm | ~normal | snapshot = durability; replicas eliminated |
| Frozen | searchable snapshot (partial mount, shared cache) | **0** | up to **20× < warm** | slowest (repo fetch) | Enterprise license; dedicated nodes; **SLA risk for old-doc search (§B2)** |

ILM discipline (advisory): rollover on size/age → tier transitions → delete at the compliance boundary.
**#1 mistake:** data stays hot far longer than it's queried — tune `min_age` to the *measured* recency
distribution. **Force-merge** before cold/frozen (fewer, larger segments). Keep cluster-telemetry retention short.

## B5. ES COST LEVERS — bytes per document (PRIMARY OWNED SURFACE)

This is where your **direct** leverage lives (mappings + what you index are tenant-owned). Mark
codec/replica/force-merge *if exposed* — Gaia ES may gate them; confirm with the platform team.

| Lever | How | Typical saving | Risk | Reversible |
|---|---|---|---|---|
| `index.codec: best_compression` (zstd) *(if exposed)* | warm+ / reindex / merge | ~15–25% shard size; frees page cache → can *improve* search | minor decompress CPU | yes |
| Force-merge *(if exposed)* | `_forcemerge` after rollover | extra ~1–15% + faster recovery | heavy CPU/I/O during merge | n/a |
| Drop redundant fields | remove parsed-then-unused source fields | observed ~20% in real datasets | none if truly unused | yes (reindex) |
| `index:false` on non-filtered fields | mapping | drops inverted index for that field | loses filterability | yes (reindex) |
| `match_only_text` vs `text` | mapping | drops norms/freqs/positions | loses scoring/phrase | yes (reindex) |
| Disable norms / selective `_source` | mapping | shaves index | breaks reindex/update-by-query if `_source` off | careful |
| Synthetic `_source` / `_id` (TSDB) | mapping/TSDB | time-series `_id` route ~34% | source reconstructed on read (CPU) | mode-dependent |
| Downsampling (TSDB) | last-value/aggregate | large for long-retained metrics | loses raw granularity | **no (lossy)** |
| Replica reduction on archive *(if exposed)* | `replicas:0` + snapshot | halves that index's storage | **no in-cluster redundancy** — only with snapshot durability | yes |
| `doc_values` tuning | off for never-aggregated | net positive | wrong call breaks aggs | yes (reindex) |

Sequence cheap/reversible first (codec, force-merge, drop fields) before lossy (downsampling) or
resilience-affecting (`replicas:0`).

## B6. SNAPSHOT REPOSITORY / PITHOS (PLATFORM-OWNED → advisory; verify first)

Cold/frozen tiers store **searchable snapshots in an S3-compatible object store**. **Pithos is
S3-compatible** → candidate snapshot repo for the FTS cold/frozen tiers, turning the migration target
into the FinOps storage substrate (long-tail data leaves hot SSD, still queryable). Repo config is
platform-owned — raise as an advisory ask after confirming: S3 API parity for the ES `s3` plugin
(multipart/list/range GET); **read latency** acceptable for frozen partial-mount fetches vs the FTS
query SLA (doubly important given §B2); SLM + durability meets audit; and **don't couple it to the
in-flight Pithos cutover's critical path** — sequence after it stabilizes.

## B7. ES MONITORING (managed platform + air-gap reality)

On managed Gaia ES, cluster/node monitoring is the **platform's** job, and **AutoOps (free since Feb
2026) needs internet egress via Cloud Connect → likely unavailable air-gapped** anyway. Your
tenant-visible signals are the **index-level proxies**, which double as advisory evidence and
saving-verification.

| Signal | Tenant proxy (owned) / platform ask |
|---|---|
| Shards past size range | `GET _cat/shards?v&h=index,shard,store` + alert > 50 GB **(tenant)** |
| Indices without ILM / oversized | `GET _cat/indices` + ILM audit job **(tenant)** |
| Shard imbalance | `_cat/allocation` + per-node shard count **(platform ask)** |
| Disk watermark (low/high/flood ~85/90/95%) | **(platform ask)** — catch before allocation failures |
| Indexing rejections / slow queries / breaker trips | thread-pool + slow log + breaker metrics **(platform ask / tenant if log access)** |
| JVM pressure | heap saw-tooth ~30–70%; not dropping to ~30% = trouble **(platform ask)** |

`_cat/indices` + `_cat/shards` are exactly the proxies that **predict the platform chargeback to FTS** —
falling store-size + shard-count proves an owned saving landed without seeing the bill.

## B8. ES RELIABILITY FLOOR + ANTI-PATTERNS

**Never advise cutting for cost:** hot-tier replicas below the resilience minimum; hot data onto NAS;
heap below GC-stable / page cache below ~50% RAM; failover/seasonal headroom without confirming it's
idle; data inside the compliance window; snapshot durability when archive replicas are 0 (the snapshot
*is* the redundancy).

| ES anti-pattern | Why it bites | Do instead |
|---|---|---|
| Recommend a platform knob as a tenant action | Not yours to set on Gaia ES | Frame tier/node/heap as advisory asks (§B0) |
| Assume "archived = rarely searched" | FTS legal/e-discovery hits old docs | Get read-recency first; veto aggressive frozen (§B2) |
| Big-bang ILM/retention change | One bad `min_age` de-tiers/deletes everything | Canary one index pattern → promote |
| `replicas:0` on hot data | No redundancy on live data | Only on snapshot-backed archive |
| Frozen tier for SLA-bound queries | Partial-mount latency blows SLA | Frozen only for rarely-queried/audit |
| Delete to cut cost | May breach compliance / irreversible | Compliance-gate every deletion |
| Claim a saving you can't see | You're advisory, not bill-owner | Verify on owned index proxies (§B7) |
| Rely on AutoOps on Gaia/CPSR | Air-gap → won't connect | Tenant proxies + platform asks (§B7) |

## B9. ES QUICK REFERENCE — the numbers
- Tiering: 90 TB all-hot ≈ **$28.2k/mo** → hot+frozen ≈ **$3.3k/mo (~88% ↓)** (Elastic Search Labs).
- Cold ≈ **50% < warm** (searchable snapshot, replicas eliminated). Frozen up to **20× < warm** (Enterprise; slower).
- `best_compression` ~**15–25%** smaller shards. Force-merge extra ~**1–15%**. Synthetic `_id` ~**34%** (TSDB).
- Heap **≤50% RAM**, **≤~30 GB**, Xms=Xmx, swap off. Page cache **≥ ~50% RAM**. Hot = SSD, never NAS.
- Shards **10–50 GB**, **<200M docs/shard**, **≤1,000/non-frozen node**; master **1 GB heap / 3,000 indices**.
- **Ownership:** tier/heap/node numbers are platform-owned (advisory + literacy); owned knobs are mappings, shards, replicas (if exposed), retention.

---

## PROVENANCE
Capacity discipline: ITIL capacity-management layering (business/service/component); queueing theory,
Little's Law, Universal Scalability Law (standard performance-engineering canon); CloudZero
(multi-dimension rightsizing, Effective Savings Rate); LinkedIn Greykite (forecast-driven autonomous
rightsizing + headroom). FinOps Framework 2025 (finops.org — Inform/Optimize/Operate, rate vs usage
optimization, Scopes/"Cloud+" for private cloud). Elasticsearch: Elastic Docs (data tiers, searchable
snapshots, JVM/heap, disk-usage tuning) + Elastic Search Labs (node/shard sizing benchmark,
best_compression, memory/heap, AutoOps Feb-2026 free + air-gap limitation). Figures are
order-of-magnitude planning inputs — benchmark against the real workload (Rally for ES) before sizing.
ES runs on managed Gaia Elasticsearch (tenant = indexes + mappings); the estate belongs to the separate
FTS app; this skill's ES posture is advisory/cross-team. Verify Pithos S3/snapshot-repo parity (a
platform-owned ask) before promising that lever.
