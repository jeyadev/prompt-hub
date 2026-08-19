---
title: AEGIS Graduated Autonomy & Runbook Action-Grading Specification
id: AEGIS-SPEC-AUTONOMY-001
version: "1.0"
status: Draft — pending stakeholder validation (Jey, Ashish, Kunal sign-off for AL2+)
owner: Jey — VP SRE, CDS / Common Capabilities
applies_to:
  - aegis-strategist
  - aegis-runbook-generator
  - aegis-subagents (DiagnosticAgent, DecisionAgent, RemediationAgent)
  - aegis-mcp-integrations (Splunk, Dynatrace, Jira, Confluence, AURA)
extends: "AEGIS Core Invariant — Write-Gate (structural, non-LLM-compliance-dependent)"
consumed_by:
  - AGENTS.md (repo root — should reference this spec as required reading)
  - SKILL.md (runbook-generation skill)
  - GitHub Copilot coding agents
  - Claude Code
supersedes: none — this is an amendment/refinement of the existing write-gate invariant, not a replacement
---

# AEGIS Graduated Autonomy & Runbook Action-Grading Specification

## 0. Why This Spec Exists

AEGIS's Dynamic Runbook Generator is currently emitting destructive and non-destructive operations as functionally identical runbook steps. A step that queries Splunk and a step that purges a Pithos document surface with the same "ready to execute" framing at approval time. The existing write-gate invariant (all production writes require human approval) is correct but **binary and ungraded** — it stops an agent from writing unsupervised, but it does not stop the *runbook itself* from treating a reversible restart and an irreversible deletion as the same kind of thing.

This is not a violation of the write-gate invariant. It is a gap the invariant never addressed: **approval friction should scale with consequence, and no amount of accumulated trust should ever collapse that scaling to zero for destructive operations.** This spec closes that gap and defines how AEGIS earns the right to act with less friction over time — narrowly, per operation type, never wholesale.

## 1. Scope

**In scope:** action classification for runbook steps; autonomy-level definitions for AEGIS's existing agent pipeline; the structural enforcement layer between DecisionAgent output and RemediationAgent execution; the trust-accrual mechanism ("loop engineering") that lets specific operation types earn reduced approval friction; phased rollout; governance mapping to Jey/Kunal/SRE governance forum.

**Out of scope:** full autonomy (Google's L4 / CSA's Level 4-5) for any operation touching CDS production — explicitly and permanently out of scope for this spec, not a later phase. Model selection, prompt engineering for the Strategist, and the KG retrieval pipeline are unaffected.

## 2. Research Basis

This spec is not invented from first principles — it adapts four current, independently-converging bodies of practice to AEGIS's specific architecture.

| Source | What AEGIS adopts |
|---|---|
| Google SRE, *"AI in SRE: How Google is Engineering the Future of Reliable Operations"* (2026) | The **Safety Trifecta** (transparency, real-time risk evaluation, progressive authorization); a 5-axis autonomy model (Monitor / Investigate / Mitigate / Actuate / Self-Direct); the pattern of a separate **Actuation Agent** as a control plane between reasoning and execution; mandatory dry-run; agentic circuit breakers; a "Red Button" for instant autonomy revocation; Bronze/Silver/Gold evaluation-data tiers for proving an agent's readiness before promotion. |
| Cloud Security Alliance, *Agentic AI Autonomy Levels and Control Framework v2.0* (2026) | The six-level autonomy taxonomy; the **Capability-Control Matrix** treated as a hard ceiling, not a guideline; the distinction between **action reversibility** and **consequence reversibility** (control requirements are driven by the latter); the requirement that autonomy-level configuration be architecturally separated from the agent's own execution context so the agent cannot self-promote. |
| OWASP Top 10 for LLM Applications (2025), **LLM06: Excessive Agency** | The three root causes of excessive agency — excessive functionality, excessive permissions, excessive autonomy — used here as a per-step self-check before any operation type is proposed for promotion. |
| Industry practice on progressive SRE-agent autonomy (CSO Online, InfoWorld, Rootly, Monte Carlo — 2026) | Trust is a property of a *specific workflow*, not the agent generally; explainability at the moment of action is what actually earns operator trust, not raw success rate; red-button controls are a sign of maturity, not a hedge against immaturity. |

The convergence across a hyperscaler's internal practice, a vendor-neutral security framework, an OWASP standard, and independent SRE practitioner writing is the basis for treating this as current best practice rather than one vendor's opinion.

## 3. Core Principle: The Write-Gate Becomes Graded, Not Binary

**AEGIS-INV-01 (original):** All write-capable operations must be structurally gated by human approval; write-capability must be unrepresentable in agent output grammar absent that gate.

**AEGIS-INV-01 (amended by this spec):** All write-capable operations remain structurally gated. The **strength** of the gate is determined by the operation's Action Class (Section 4), fixed at generation time, and is visible independently of whatever plan the step is embedded in. Trust accrued through the loop in Section 8 can *reduce* gate friction for Class 0/1 operations. **It can never reduce gate friction below per-action human approval for Class 3 operations, regardless of track record.** This is a ceiling, not a target to eventually lift.

This is additive to the existing invariant. Nothing in this spec permits an unsupervised write that the current architecture disallows; it only permits *less friction* for a narrow, earned, continuously-monitored subset of reversible writes — and it adds friction (structural class-tagging, dry-run, classifier confirmation) to the destructive path that doesn't exist today.

## 4. Action Classification Schema

Every step a RunbookGenerator emits gets exactly one class. Class is assigned by a deterministic classifier, not by the LLM's self-report (Section 12) — this matters because it's the classifier, not the model's stated intent, that the enforcement layer trusts.

| Class | Definition | CDS/AEGIS examples | Action reversibility | Consequence reversibility | Max autonomy ceiling |
|---|---|---|---|---|---|
| **0 — Read/Diagnostic** | No state mutation anywhere | Splunk query, Dynatrace trace pull, Jira ticket read, Confluence lookup | N/A | N/A | AL3 |
| **1 — Reversible, Bounded** | Mutates state; effect is scoped to a single instance/service; trivially reversible; blast radius contained | Restart one unhealthy pod, clear a local cache, bounce a single service instance | High | High | AL2 (AL3 only after Trust Ledger promotion, Section 8) |
| **2 — Reversible, Broad** | Mutates state; effect spans a tenant or multiple consumers; reversible but not trivially, or consequence reversibility is uncertain | Scale replica count platform-wide, push a config change affecting one tenant (IPB/PBA/GVA), drain a node | High (action) / Medium (consequence) | Medium | AL1 (AL2 only with documented executive exception, Section 10) |
| **3 — Irreversible / Destructive** | Action cannot be undone, or consequences propagate beyond AEGIS's control once triggered | Document purge, force-rollback past a Pithos commit point, terminate a migration job, drop an index, revoke credentials, any cross-tenant write | Low or N/A | Low | **AL1 — permanent.** Never eligible for promotion. |

**Consequence reversibility governs classification, not action reversibility.** A step that is technically undoable (restore-from-backup) but whose downstream effect has already propagated to a consumer outside AEGIS's control (a document already served to an AWM downstream application, a Pithos GET already returned to a caller) is classified by the *consequence*, not the mechanism. This directly matches the reported bug: a "revert" step can look safe by action-reversibility alone while carrying Class 2/3 consequence risk.

**Migration-state override:** any step touching Pithos during an active cutover window is unconditionally Class 2 minimum, regardless of what the classifier would otherwise assign — this is a direct consequence of the commit-point/rollback-cost asymmetry already flagged for the IBM CM → Pithos migration: rollback cost increases monotonically as Pithos accumulates documents absent from IBM CM, so even a "should be reversible" step during cutover carries a higher true consequence-reversibility risk than the same step in steady state.

## 5. AEGIS Autonomy Levels

Adapted from Google SRE's 5-axis model, mapped onto AEGIS's existing agent pipeline rather than introducing new agents.

| AEGIS Level | Monitor | Investigate (DiagnosticAgent) | Decide (DecisionAgent) | Actuate (RemediationAgent) | Roughly equiv. |
|---|---|---|---|---|---|
| **AL0 — Manual** | Automated | Automated | Human | Human | CSA L1 / Google L0 |
| **AL1 — Assisted (current AEGIS default, Phase F baseline)** | Automated | Automated | Automated proposal, per-step class-tagged | Human approves **each Class ≥1 step individually** before execution | CSA L1-2 / Google L1 |
| **AL2 — Partial (target: Phase 2, this spec)** | Automated | Automated | Automated, plan-level | Human approves the **plan as a whole**, only if every step in it is Class 0/1 and the operation type clears its Trust Ledger threshold | CSA L2 / Google L2 |
| **AL3 — High (aspirational, narrow pilot only, Phase 3+)** | Automated | Automated | Automated | Bounded autonomous execution for specific, proven Class 0/1 templates; human notified post-hoc, not pre-approval | CSA L3 / Google L3 |
| **AL4 — Full / Self-Direct** | — | — | — | — | **Explicitly excluded from this spec's roadmap.** Not a later phase. Reintroducing it requires a fresh governance decision outside this document. |

A single plan containing a mix of classes is always gated at the level of its *strictest* step. A Class 3 step anywhere in a plan forces the entire plan to AL1 per-step review — there is no such thing as a "mostly-safe" plan for approval purposes.

## 6. Structural Enforcement Architecture

This introduces one new architectural component: **AEGIS Gate**, sitting between DecisionAgent output and RemediationAgent execution — the same separation Google implements between its AI Operator (reasoning) and Actuation Agent (execution), and the same architectural-separation principle CSA specifies as mandatory at Level 3+ (autonomy configuration must live outside the agent's own execution context, or the agent can effectively self-promote).

```
 Strategist ──► DiagnosticAgent ──► DecisionAgent ──► [ AEGIS GATE ] ──► RemediationAgent ──► MCP write
                                    (proposes plan,                │
                                     step classes                  ├─ schema validation (reject unclassified steps)
                                     advisory only)                ├─ deterministic classifier confirms/overrides class
                                                                    ├─ approval-tier lookup from cap table (§4/§5)
                                                                    ├─ dry-run enforcement (Class 1/2, MCP-supported)
                                                                    ├─ Trust Ledger read (promotion eligibility)
                                                                    └─ Red Button check (global override)

 AUTONOMY CONFIG STORE (level caps, thresholds, Trust Ledger)
   — readable by AEGIS Gate
   — NOT writable by Strategist / DecisionAgent / RemediationAgent runtime
   — writable only by: Trust Ledger promotion job (human-signed-off, §8) + direct human admin action
```

Gate responsibilities:
- **Reject, don't default.** A step missing any required field from Section 7's schema is a hard validation failure. It is never silently treated as Class 0.
- **Classifier confirms, LLM proposes.** DecisionAgent's stated class is advisory. A rule-based classifier (keyed to MCP tool name + verb + target scope) makes the binding determination. This is the "deterministic before probabilistic" principle already governing AEGIS, applied specifically to the thing that decides how much friction a write gets.
- **Cannot be reconfigured by the pipeline it governs.** Level caps, Trust Ledger scores, and promotion thresholds live in a config store the agent runtime can read but never write — this is the direct implementation of CSA's escalation-prevention principle and closes the exact failure mode CSA's incident evidence flagged: an agent (or an attacker manipulating it) altering its own boundary configuration from within its own execution context.
- **Red Button.** Independently authenticated, not reachable through any agent tool-call surface. Any on-call SRE can instantly revoke AL2/AL3 grants platform-wide, reverting every operation type to AL1 pending review — this is the emergency stop CDS on-call needs available during a bad migration cutover window without waiting on AEGIS's own reasoning loop.

## 7. Runbook Step Schema

RunbookGenerator MUST emit every step in this shape. This is the artifact that makes classification structural rather than descriptive.

```yaml
step:
  step_id: string
  description: string
  mcp_target: string            # e.g. "splunk", "dynatrace", "pithos-admin"
  action_type: read | write
  action_class: 0 | 1 | 2 | 3
  reversibility:
    action: high | medium | low | n/a
    consequence: high | medium | low | n/a
  control_driver: consequence   # fixed — control decisions are driven by consequence reversibility
  blast_radius: instance | service | tenant | cross_tenant | platform
  affected_tenants: [IPB, PBA, GVA]   # empty list if none
  dry_run_supported: boolean
  dry_run_required: boolean     # true for all Class 1/2 on first-ever execution of an operation type
  rollback_procedure: string    # required for action_class >= 1; "n/a" invalid for class >= 2
  required_approval: none | single | dual
  max_autonomy_level_allowed: AL0 | AL1 | AL2 | AL3
  rationale: string             # agent's stated reasoning — surfaced to the approver, not just logged
```

**Example — Class 0 (contrast case):**
```yaml
step:
  step_id: "diag-001"
  description: "Query Splunk for 5xx rate on cds-document-api over last 30m"
  mcp_target: "splunk"
  action_type: read
  action_class: 0
  reversibility: { action: n/a, consequence: n/a }
  control_driver: consequence
  blast_radius: instance
  affected_tenants: []
  dry_run_supported: false
  dry_run_required: false
  rollback_procedure: "n/a"
  required_approval: none
  max_autonomy_level_allowed: AL3
  rationale: "Establish current error rate baseline before proposing remediation."
```

**Example — Class 3 (the case that's currently mis-surfaced):**
```yaml
step:
  step_id: "remediate-004"
  description: "Purge orphaned Pithos documents flagged by store-to-find consistency check"
  mcp_target: "pithos-admin"
  action_type: write
  action_class: 3
  reversibility: { action: low, consequence: low }
  control_driver: consequence
  blast_radius: cross_tenant
  affected_tenants: [PBA, GVA]
  dry_run_supported: true
  dry_run_required: true
  rollback_procedure: "No reliable rollback once purged — restore requires IBM CM source-of-truth reconciliation, hours-scale, manual"
  required_approval: dual
  max_autonomy_level_allowed: AL1
  rationale: "Documents match orphan signature (S3 GET 404 correlation) but purge is irreversible pending full commit-point audit."
```

These two steps must never render identically in an approval UI or CLI. Step 004 requires visually distinct, harder-to-dismiss confirmation — this is the direct fix for the reported problem.

## 8. The Trust Ledger — Loop Engineering

This is the mechanism that lets AEGIS earn reduced friction, narrowly and reversibly.

**Unit of trust: operation type, not agent.** "Restart single pod on health-check-fail signature X" and "scale replica count" accrue trust independently. A perfect record on one buys nothing for the other. This is deliberate — it's the direct antidote to the failure mode where a system's overall success rate is used to justify autonomy for an operation that's never actually been exercised.

**Reinforcing loop:** each human-approved execution of an operation type that completes without rollback or incident attribution increments that operation type's trust score.

**Balancing loop — asymmetric by design:** a single rollback, human rejection, or incident attribution demotes the operation type sharply — further down than a single success moves it up, and enough to force re-earning from a much lower baseline. Trust here should be hard to earn and easy to lose; that asymmetry is what Google's own framing describes as the thing that's actually required to overcome legitimate operator hesitancy before a system is allowed to act without real-time approval.

**Promotion is never fully automatic.** Meeting the numeric threshold makes an operation type *eligible*; a human still has to sign off. This mirrors both CSA's Autonomy Review Board requirement at Level 3 and Google's requirement of demonstrated, statistically significant success against human-verified data before advancing past human-approved execution.

**Evaluation tiers (adapted from Google's Bronze/Silver/Gold):**
| Tier | Source | Use |
|---|---|---|
| Bronze | Auto-logged execution outcome (did the alert clear, did rollback fire) | Raw signal, noisy |
| Silver | Bronze calibrated against a sampled subset of Gold | Used for trust-score math |
| Gold | Human-verified correct outcome | Aayushi's domain review of demo/pilot scenarios is the natural Gold-labeling function here — this folds directly into the existing SV-1.4 stakeholder item rather than creating new process |

**Decay:** an operation type's score decays if unexercised for 30 days (default, tunable). An automation nobody has run recently shouldn't carry the same trust as one exercised weekly — this is the drift-into-failure guardrail applied to the ledger itself.

**Default thresholds (illustrative — Jey to tune against real CDS incident volume):**

| Transition | Threshold |
|---|---|
| AL1 → AL2 eligible | ≥10 consecutive clean human-approved executions, over ≥14 days, zero rollbacks, zero incident attribution |
| AL2 → AL3 eligible | ≥50 clean executions at AL2, over ≥60 days, Gold-verified outcome match ≥98%, **Kunal sign-off required** |
| Any single AL2/AL3 failure | Immediate demotion to AL1 for that operation type; re-earn from zero; Red Button remains available for platform-wide emergency revocation regardless of any individual operation type's standing |

## 9. Phased Rollout

| Phase | Scope | Grants new autonomy? |
|---|---|---|
| **Phase 0 — Immediate** | Retrofit RunbookGenerator to the Section 7 schema. Unclassified/malformed write steps become a hard validation failure. Approval UI/CLI structurally distinguishes Class 2/3 steps from Class 0/1 (separate confirmation friction — typed confirmation for Class 3). | **No.** This phase only fixes grading of what already runs at AL1 today. This is the fix for the reported bug. |
| **Phase 1 — Trust Ledger instrumentation (4–6 wks)** | Instrument the ledger for all Class 0/1 operation types already running at AL1. Start accumulating Bronze/Silver data. | No promotions. Purely observational. |
| **Phase 2 — Pilot AL2 (next quarter)** | 2–3 highest-volume, lowest-blast-radius operation types (e.g., single-pod restart on a known health-check-fail signature, non-prod only). Kunal sign-off before pilot start. Shadow/dry-run mode for 2 weeks before real AL2 promotion. | Yes — narrow, reversible, non-prod first. |
| **Phase 3 — Narrow AL3 pilot (contingent)** | Only for Phase 2 operation types that clear the AL2→AL3 threshold. Non-prod first, then IPB → PBA → GVA sequentially — never all tenants at once, consistent with the existing incremental-rollout guardrail already applied to the Pithos migration itself. | Yes — narrowest possible scope, sequential tenant rollout. |

**Explicit non-goal:** there is no phase in this roadmap that reaches AL4 or that permits Class 2/3 auto-execution. Reaching either requires a new governance decision outside this document, not a checkbox in this rollout.

## 10. Governance & Stakeholder Mapping

| Decision | Authority |
|---|---|
| Classification taxonomy changes (Section 4) | Jey + Ashish (domain correctness for CDS/Pithos specifics) |
| New operation type onboarded to Trust Ledger | Jey |
| AL1 → AL2 promotion | Jey; reported at weekly SRE governance forum alongside existing aging-ticket reporting |
| AL2 → AL3 promotion | **Kunal sign-off required** + documented risk acceptance — matches both CSA's executive-authorization requirement at Level 3 and Google's requirement of executive gating before autonomous actuation without real-time approval |
| Red Button invocation | Any on-call SRE, no approval needed — always available |
| Scope expansion of an already-promoted operation type to a new tenant (IPB/PBA/GVA) | Treated as a fresh promotion request — trust does not transfer across tenant boundaries |

**Two-sentence Kunal framing:** AEGIS runbooks are now graded by blast radius instead of treated uniformly — destructive operations stay permanently human-gated, while routine reversible fixes only earn reduced approval friction after a proven track record we can show you. This closes the gap where the tool could previously surface a destructive step with the same ease as a read-only check.

## 11. Failure Mode Checklist

| Failure mode | Mitigation in this spec |
|---|---|
| Automation complacency / rubber-stamping | Structural visual/friction distinction for Class 2/3 (§7, §9 Phase 0); periodic random audit-sampling of AL2 approvals even at high trust, mirroring Google's stratified Gold-data sampling |
| Toil spiral inversion (ledger review becomes its own toil) | Ledger math is automated; only the promotion sign-off requires a human |
| Drift into failure (silent laxity over time) | Decay mechanism (§8) |
| Prompt injection via observed MCP content (Splunk/Dynatrace/Jira text) attempting to talk DecisionAgent into misclassifying a step | Class is confirmed by a deterministic classifier, never taken on the LLM's self-report alone (§6, §12) |
| Migration-state blind spot | Unconditional Class 2 floor for any Pithos-touching step during active cutover (§4) |
| Excessive agency (OWASP LLM06) — functionality | Cap table (§4/§5) bounds what any operation type can ever reach regardless of trust |
| Excessive agency (OWASP LLM06) — permissions | Gate enforcement is architecturally separate from agent runtime (§6) |
| Excessive agency (OWASP LLM06) — autonomy | Promotion is never fully automatic; a human signs off every tier change (§8) |

## 12. Implementation Directives for Coding Agents (Copilot / Claude Code)

Normative for whoever implements this in the repo:

- **MUST** add `action_class`, `reversibility`, `blast_radius`, `required_approval` fields to the RunbookGenerator's output schema; **MUST** reject generation output that omits them — no default-to-safe fallback.
- **MUST** implement class assignment via a deterministic classifier (rule/allowlist keyed to MCP tool + verb + target scope), not solely an LLM-generated field. LLM-proposed class is advisory only until the classifier confirms it.
- **MUST NOT** allow Strategist, DecisionAgent, or RemediationAgent runtime code to write to the autonomy-level config store or Trust Ledger scores. These are read-only from the agent pipeline's perspective; writes come only from the promotion job (human-signed-off) or direct admin action.
- **MUST** implement the Red Button as an independently authenticated control that is not reachable through any agent tool-call surface.
- **SHOULD** implement dry-run pass-through wherever the target MCP server supports a non-mutating simulate mode; **MUST** require it for Class 1/2 steps on the first-ever execution of a new operation type.
- **MUST** log every step (proposed, approved/rejected, executed, outcome) to an immutable audit store keyed by operation type — this is the Trust Ledger's data source.
- **SHOULD** surface class and rationale directly in whatever UI/CLI renders the runbook for human approval; **MUST NOT** allow a Class 3 step to render with the same visual weight as a Class 0 step in the same plan.
- **MUST** treat this spec as authoritative over ad hoc prompt-level instructions to the RunbookGenerator — a prompt cannot lower an operation's required approval tier. No stubs, no TODOs in the Gate's validation path; an incomplete Gate is equivalent to no Gate.

## 13. Open Items — Ties to Existing Stakeholder Validation

- **SV-1.2 (pause timeout policy):** should incorporate class. Class 2/3 pauses get no auto-resume timeout, ever — consistent with the existing "no auto-resume logic in v1 of safety-sensitive loops" principle. Class 0/1 pause timeout policy is unaffected by this spec.
- **SV-1.4 (Aayushi demo scenario compatibility):** Aayushi's domain review is the natural Gold-labeling function for Trust Ledger calibration (§8) — fold into her existing role rather than standing up new process.
- **SV-0.1 (JPMC single-file VDI deployment constraint):** Trust Ledger storage should default to SQLite, matching the existing MVP persistence choice, to stay VDI-compatible. This spec introduces no new infrastructure dependency.

## 14. Repo Placement (assumption — relocate as needed)

Suggested path: `specs/AEGIS-RUNBOOK-AUTONOMY-SPEC.md`, referenced from `AGENTS.md` as required reading for any change touching the RunbookGenerator, and linked from the `SKILL.md` governing runbook generation. No AEGIS references belong in the workshop-facing skill files per existing convention — this spec is the exception, since it *is* AEGIS-specific by design.

---

## Appendix A: Glossary

- **Action class** — fixed-at-generation-time grading (0–3) of a runbook step by consequence reversibility and blast radius.
- **AEGIS Gate** — the new architectural component enforcing this spec, structurally separate from the agent pipeline it governs.
- **Consequence reversibility** — whether the downstream effect of an action can be meaningfully undone, distinct from whether the technical action itself can be undone (action reversibility). Control decisions in this spec are driven by consequence reversibility.
- **Trust Ledger** — per-operation-type accrual mechanism that makes reduced approval friction earnable, never granted wholesale.
- **Red Button** — independently authenticated, instantly-available, platform-wide revocation of AL2/AL3 grants.

## Appendix B: References

- Google SRE, "AI in SRE: How Google is Engineering the Future of Reliable Operations," 2026 — https://sre.google/resources/practices-and-processes/ai-engineering-reliable-operations/
- Cloud Security Alliance AI Safety Initiative, "Agentic AI Autonomy Levels and Control Framework," v2.0, March 2026 — https://labs.cloudsecurityalliance.org/wp-content/uploads/2026/03/agentic-ai-autonomy-levels-control-framework-v2-csa-styled.pdf
- OWASP Top 10 for LLM Applications (2025), LLM06: Excessive Agency — https://owasp.org/www-project-top-10-for-large-language-model-applications/
- CSO Online, "What SRE teams need before they trust AI agents," June 2026 — https://www.csoonline.com/article/4183666/what-sre-teams-need-before-they-trust-ai-agents.html
- InfoWorld, "How to teach SRE AI agents to fail safely and earn your team's trust," July 2026 — https://www.infoworld.com/article/4195114/how-to-teach-sre-ai-agents-to-fail-safely-and-earn-your-teams-trust.html
- Monte Carlo Data, "Agentic Autonomy Is A Trust Score," April 2026 — https://montecarlo.ai/blog-agentic-autonomy-is-a-trust-score/
