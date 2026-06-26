# Non-Prod 80/20 Sizing — Strategy & Decision Spec

> **What this is:** a handoff spec for the Cline/Opus instance running inside the JPMC VDI. It encodes every decision already made and every decision still open, so that once Gaia portal billing data is pasted in, the agent can resolve the open forks deterministically and emit the final **app-owner-facing guideline**.
>
> **What this is NOT:** the app-owner guideline itself. This is the reasoning spec behind it. Produce the owner-facing doc as the final step (see §13).
>
> **Single-file by design.** No external references required. Everything needed to finish the work is in this file plus the pasted Gaia billing data.

---

## 0. The job in one line

Translate a vague "80/20 prod/non-prod" mandate into an implementable, anti-drift sizing guideline for all app owners, measured in the unit that actually reduces the Gaia bill, with a clean exemption framework so owners can self-serve.

---

## 1. Confirmed context (do NOT re-derive — these are settled)

| # | Fact | Implication |
|---|---|---|
| C1 | **Reading A confirmed**: prod is partitioned per-tenant (IPB / PBA / GVA each have their own prod footprint). | 80/20 is computable per tenant and *holds* per tenant. No structural-impossibility problem in the general case. |
| C2 | **Each tenant requires its own non-prod** — tenants support different regions → different integrating teams. | Tenant is a **structural accounting unit**, not an exemption. |
| C3 | **Unit of accounting = (app × tenant).** | 80/20 is computed independently inside each (app × tenant) cell, against THAT cell's prod. |
| C4 | **Cost is the governing driver.** Not instance count for its own sake, not core for its own sake — spend. | The 80/20 must be expressed in whatever unit Gaia charges back in. Count and core are proxies; pick the proxy that tracks the bill. |
| C5 | Prod carries multi-DC redundancy for HA (e.g. app1 IPB = 2 instances each in eu-1z/2z/2y = 6). | This redundancy is an **availability** property, not a load property. It does NOT carry to non-prod by default. |
| C6 | Environment is **JPMC Private Cloud (Gaia/CPSR)** — single-file/VDI-friendly, enterprise governance, change-managed. | No public-cloud assumptions. Ephemeral provisioning latency is unknown — see Q3. |

---

## 2. Core principle (the spine of the whole guideline)

**Decompose every footprint into two independent axes. Govern each axis with a different control.**

| Axis | What it represents | Governed by | Non-prod rule |
|---|---|---|---|
| **Availability / topology** | Surviving DC loss; quorum; placement | **Instance count + DC placement** | Collapse to **1 DC**, **minimum-viable count** (usually 1). Non-prod has no uptime SLA, so it does not buy redundancy. |
| **Cost envelope** | What you actually spend to run the env | **Core / CU (per §4 decision)** | The **80/20 ratio lives here**, computed per (app × tenant). |

**One-liner for the guideline:** *Count rules the structure, cores rule the budget. Owners implement by choosing count + size; the guideline governs the envelope in the billing unit; governance verifies in chargeback dollars.*

The single most important discipline to defend in owner conversations:
> **Tenant separation is granted. DC redundancy is not.**
> App owners will hear "each tenant gets its own non-prod" and try to smuggle in "…across 2 DCs like prod." Tenant axis = 3 (given). DC axis *within* each tenant = 1 by default. Keep them separate or 20% becomes 60% on first concession.

---

## 3. Why count alone is the wrong unit when cost is the goal

Two structural problems — include both in the owner doc's rationale so the unit choice isn't litigated app-by-app:

**3a. Count is gamed by vertical sizing, invisibly.** Example — app1 IPB prod = 6 × 8-core = 48 cores:

| Owner behavior | By count | By core | Real spend |
|---|---|---|---|
| Good actor: 2 × 2-core (quorum) | 2/6 = 33% ✗ "non-compliant" | 4/48 = 8% | lowest |
| Lazy actor: 1 × 16-core "headroom" | 1/6 = 17% ✓ "compliant" | 16/48 = 33% | ~2× the good actor |

Count **punishes the efficient owner and rewards the wasteful one.** It trains the wrong behavior across all services.

**3b. Count only credits one of two cost levers.** Owners can cut non-prod spend by (a) fewer instances and (b) smaller instances. Non-prod *should* be downsized — functional test doesn't need prod-grade boxes. Count gives zero credit for shrinking a box, killing the downsizing lever. A billing-unit-based ratio rewards both → pulls non-prod toward "fewer **and** smaller," the desired outcome.

**Leverage framing:** count is a Meadows level-12 parameter that creates a perverse loop. A unit matched to the bill is a level-8 balancing loop — gaming it = not saving = pointless.

---

## 4. THE DECISION the agent must make from Gaia data

Resolve the governing unit from the chargeback model. **Paste Gaia portal billing detail, identify the chargeback basis, then apply:**

| Gaia chargeback model | Govern 80/20 in | Rationale |
|---|---|---|
| Per vCPU / per resource | **Core** | Bill tracks cores. |
| Composite compute-unit (core + mem weighted) | **CU** | Bill tracks the CU; cleanest, also closes the memory gap (§9). |
| Flat per-VM (same $ regardless of size) | **Count** | Bill tracks instances; core is noise. *Then* harden a separate size-cap rule so owners don't keep prod-grade boxes (count won't push downsizing on its own). |
| Real per-(app×tenant) chargeback exists | **$ directly** for governance; core/CU as the lever owners pull | Govern the outcome, not the proxy. |

**Prior:** for a bank's managed compute, resource-based or CU is far more likely than flat-per-VM — but **confirm from the portal, do not assume.** This single fact decides count-vs-core.

### What to extract from the Gaia portal (checklist for the paste)
- [ ] Chargeback **basis**: per-vCPU? per-CU? flat-per-VM? per-GB-RAM component?
- [ ] Whether a **composite CU** is published, and its formula (core/mem weights)
- [ ] Whether chargeback is already broken out **per (app × tenant)** or only per cost-center/app
- [ ] **Unit price(s)** — enough to convert a sizing change into a dollar delta for the worked examples
- [ ] Any **minimum charge / floor** per instance (changes whether "1 tiny box" actually saves vs. a floor)
- [ ] Whether **non-prod and prod are billed at the same rate** or non-prod is discounted (affects how hard the 20% needs to bite)

---

## 5. Baseline rule (the "80" — default everyone follows unless exempt)

```
Unit of accounting = (app × tenant)

Per (app × tenant) non-prod baseline:
  • Single DC            — collapse prod's multi-DC HA topology
  • DC placed in/near    — region the tenant's team serves
    the tenant's region    (pin to jurisdiction if data-residency applies)
  • Minimum-viable        — usually 1; higher ONLY via quorum floor (§6 #1)
    instance count
  • 80/20 envelope        — computed in the unit chosen in §4, per cell,
                            against THAT cell's prod
```

---

## 6. Exemption catalog (the part owners self-serve against)

**Rule of engagement:** exemptions are **claimed against a named trigger with evidence**, not asserted. Default response to "I need more" is "show me the test you'll run." Every exemption is **time-boxed and registered** (see §7).

| # | Exemption | Trigger (when it legitimately applies) | What you get | Evidence required |
|---|---|---|---|---|
| 1 | **Quorum floor** *(constraint, not perk)* | App can't run representatively below N nodes (leader election, clustering) | N instances, **still 1 DC** | Runtime topology requirement |
| 2 | **Failover validation** | You actually run DC-failover / DR drills in non-prod | **Ephemeral 2nd DC during scheduled drills only — never standing** | Drill cadence + runbook |
| 3 | **Performance validation** | You load-test at prod-like scale pre-release | Prod-proportional depth **in ONE env (perf/UAT) only, during perf windows** — do NOT size DEV/SIT up | Perf test plan |
| 4 | **Tenant-share (inverse exemption)** | **No** distinct team integrates per tenant — one shared QA/team hits non-prod across all tenants | Collapse the 3 tenant non-prods to **1 shared** → saves ⅔ of that app's non-prod fleet | Confirmation that no per-tenant integrating team exists (name the team, or confirm absence) |

Notes that must survive into the owner doc:
- **#2 defaults to ephemeral.** Standing multi-DC non-prod is exactly where 20% leaks back to 60%. A 2nd DC spun up for a quarterly drill and torn down is ~0% time-averaged. **Depends on Gaia provisioning latency (Q3)** — if a DC can't be stood up/down inside a drill window without a multi-week change ticket, #2 is forced to standing capacity and gets expensive. Resolve Q3 before finalizing #2's wording.
- **#2 is now 3× costlier than in a single-tenant world** — a drill spins ephemeral DCs across three tenant non-prods. Reinforces ephemeral-only.
- **#4 is the sparring point on the mandate itself.** "Different teams per region" is the *justification* for per-tenant non-prod — so the test is per-app: is there genuinely a distinct team per tenant touching non-prod? For a tenant-agnostic utility service among the 12, maybe not — it shouldn't pay the 3× tax ceremonially. Make the owner name the team or claim the exemption.

---

## 7. Anti-drift mechanism (do not omit — this is the real leverage)

Without it, the 80/20 erodes via locally-rational exemption creep: every owner claims #2 "just in case," and in ~9 months you're back to baseline bloat (textbook drift-into-failure). The balancing loop:

- Every exemption is **time-boxed** and **registered** with the specific test activity that justifies it.
- **Quarterly review:** *is the test that justified this exemption still being run?* If app1 claimed failover-validation but ran no drill in two quarters → exemption expires → drops to baseline.

**Leverage note for the narrative:** the 80/20 *number* is Meadows level 12 (weak parameter). The **exemption-with-review framework** is level 5–8 (rules + balancing loop). Writing a *principle* + review instead of dictating per-app numbers is the higher-leverage move — and the review is what makes it hold.

---

## 8. Worked example — app1, IPB tenant (Reading A)

Fill the cost column once §4 unit + §4 unit prices are known.

| Scenario | Non-prod sizing | Count ratio | Cost ratio (fill from Gaia) |
|---|---|---|---|
| Stateless, no perf/failover testing | 1 instance, 1 EU DC | ~17% ✓ | ___ |
| + needs failover drills | baseline + **ephemeral** 2nd DC during quarterly drill | ~17% time-avg | ___ |
| + needs prod-scale perf test | 2 instances in 1 DC during perf windows; baseline 1 otherwise | ~17–33% | ___ |
| 3-node quorum app | 3 instances, 1 DC (floor) | 50% | ___ |

**The quorum row matters:** a quorum app legitimately blows 80/20, and the guideline must **say so explicitly** so owners don't cheat a broken 1-node deployment that misrepresents prod behavior. "Compliant" ≠ "as small as possible" when the floor is real.

Repeat the same table per tenant (IPB / PBA / GVA) — each computed against its own prod.

---

## 9. Edge cases to encode

**9a. Reading-B carve-out.** Although the general case is Reading A (C1), check each of the 12 services. If any service has **shared prod but tenant-split non-prod** (forced by separate teams), then 80/20 is **mathematically unreachable** for it (3 non-prod ÷ 6 shared prod = 50%). The guideline must carve this out explicitly so those owners aren't marked non-compliant for a structural, non-wasteful reason. Don't make them lose a fight that isn't theirs.

**9b. Licensing multiplier — may dominate infra cost.** If any of the 12 run **per-core-licensed middleware** (Oracle, IBM MQ/WAS, etc.), core reduction in non-prod saves **license cost on top of infra** — often the larger line. Two consequences:
- Strengthens **core/CU** as the unit (count gives zero license relief; a fat single instance keeps full license exposure).
- **Re-prioritize rollout:** sequence license-heavy services **first** — that's where 80/20 frees real money. A stateless lightweight service saves pennies whichever unit is chosen. Add per-core-license status as a column when inventorying the 12.

**9c. Memory blind spot.** Cores ignore RAM; a cache/in-memory workload could run 2 cores / 64 GB and slip a core-only metric while costing real money. If Gaia publishes a composite **CU**, use it (closes this). If core-only, add a **one-line memory sanity check** to the §7 review so memory-heavy non-prod can't hide behind a low core count.

**9d. Data residency on placement.** A tenant's "1 DC" is **pinned** to its jurisdiction where residency rules apply (likely GVA) — not free choice. One line in the owner doc.

---

## 10. CDS dogfooding flag (don't skip)

CDS's own 12 services are subject to this same rule. Two self-checks before circulating:
- **Resolve where CDS sits on Reading A vs B** per service — don't publish a rule CDS would itself violate.
- **CDS is a shared service**: downstream AWM teams integration-test against CDS non-prod. That's exemption-#4-in-reverse — CDS likely *cannot* tenant-collapse and must hold modest standing availability so it doesn't become the flaky dependency that blocks every consuming team. Size CDS's own non-prod with that in mind; "as small as possible" is wrong if it makes CDS the integration bottleneck.

---

## 11. Open questions to resolve from the Gaia portal / VDI

| Q | Question | Resolves | Where the answer comes from |
|---|---|---|---|
| Q1 | Gaia chargeback **basis** (vCPU / CU / flat-per-VM / per-app×tenant)? | §4 unit decision — count vs core vs CU vs $ | Gaia billing detail (paste) |
| Q2 | Do any of the 12 run **per-core-licensed** software? Which? | §9b rollout sequencing + strengthens core | App inventory / license records |
| Q3 | Gaia **ephemeral DC provisioning latency** — can a DC be stood up/down inside a drill window without a multi-week change ticket? | §6 #2 ephemeral-vs-standing wording + cost | Gaia platform docs / change-mgmt SLA |
| Q4 | Per-service **Reading A vs B** across all 12 | §9a carve-out scope | Topology inventory |
| Q5 | Is non-prod billed at the **same rate** as prod, or discounted? | How hard the 20% must bite | Gaia billing detail |
| Q6 | Per-instance **minimum charge / floor**? | Whether "1 tiny box" actually saves | Gaia pricing |

---

## 12. Precision still owed on "80/20" itself

Pin these before publishing, so owners aren't measured against a moving definition:
- **20% of what** — count / cores / CU / cost? (Resolved by §4, but state it explicitly in the doc.)
- **Cap or target?** Hard ceiling vs. aspirational. Affects exemption posture.
- **Time-averaged or point-in-time?** Ephemeral exemptions (#2) only make sense under time-averaged. State the measurement window.

---

## 13. Final deliverable the agent should produce

Once Q1–Q6 are resolved, emit the **app-owner-facing guideline** as a standalone doc with:

1. **One-paragraph principle** — decompose availability from load; what unit governs cost; tenant = accounting unit.
2. **Baseline rule** (§5) in plain owner language.
3. **(app × tenant) accounting model** with a filled worked example in the chosen unit (use real Gaia unit prices).
4. **Exemption request table** (§6) with the evidence columns — formatted so an owner can fill and submit it.
5. **The two carve-outs** stated plainly: quorum floor can exceed 20% (legitimately); Reading-B services may be structurally exempt.
6. **The quarterly review** (§7) — one paragraph, so owners know exemptions expire.
7. **Rollout order** (§9b) — license-heavy services first.

Keep it **dense, table-heavy, action-oriented, deterministic** — owners should be able to size their non-prod from it without a meeting. Avoid internal jargon (Meadows/leverage framing stays in *this* spec, not the owner doc — owners get the rule, not the theory).

---

*Source of truth: this spec + pasted Gaia billing data. If Gaia data contradicts an assumption here (e.g. flat-per-VM after all), follow the data and update §4–§9 accordingly.*
