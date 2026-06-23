ROLE: Workshop Architect mode.

NEW ARTIFACT TO INTEGRATE
File: ./capacity-management-strategy.skill.md  (adjust path to where it landed in sre-workshop)
A SKILL.md (v2.0). Shape, so you don't have to reverse-engineer intent:
- PART A (§A0–A12): a workload-agnostic capacity-management discipline — demand modelling,
  forecasting, headroom math, scaling strategy, queueing limits, maturity model, cost integration.
  Relevant to many tasks, not just one domain.
- PART B (§B0–B9): a dedicated Elasticsearch section that instantiates Part A for ES on a managed
  internal ES platform; ES posture is advisory/cross-team (tenant surface = indexes + mappings;
  estate owned by a separate Full Text Search app).
- Embedded behavioral contract that must survive integration: reliability-first; lever order
  eliminate→rightsize→reshape→re-rate; deterministic-before-probabilistic; ask-don't-assume; a
  per-recommendation output contract; private-cloud constraints (no RI/spot, change-management
  gates, blast radius, small team).
- Consumption design: token-efficient — read once, hold the section index + output contract, then
  pull sections by ID on demand. Do NOT inject the whole file into every mode's context.

OBJECTIVE
Absorb this artifact into the workspace as the right combination of custom mode(s) + skill(s) that
interoperate cleanly with the existing specialist modes. You own the architecture decision. Be
opinionated — recommend and justify a single decomposition; do not hand me a menu of options.

DECISIONS THAT ARE YOURS (decide using the workspace's actual architecture, conventions, and routing)
- Mode vs skill vs both; the behavioral wrapper vs the knowledge body.
- Whether/how to split Part A (general) from Part B (ES) so other modes can pull the general
  capability without carrying the ES weight.
- Naming/slugs, repo placement, tool-group permissions, any fileRegex edit guards.
- How it registers in the mode index / AGENTS.md / CLAUDE.md.
- Orchestrator routing + handoff semantics.

INTEGRATION BAR (the plan must satisfy all)
- Activation boundary: define when this capability OWNS a task vs CONTRIBUTES a sub-assessment to
  another specialist mode.
- Handoffs: how other modes invoke/defer to it and how it returns results.
- No duplication/conflict: reference existing modes' ownership instead of forking it; surface any
  overlaps you find and resolve them.
- Token-efficient loading preserved (read-once + pull-by-ID; no whole-file injection).
- Portable: SKILL.md / AGENTS.md standards, no lock-in.
- Self-contained: scope strictly to capacity / cost / sizing / forecasting / tiering; don't entangle
  it with unrelated subsystems.
- The embedded behavioral contract (output contract, lever order, advisory ES posture) stays in
  force wherever the capability is active.

DELIVERABLES
1. Integration plan: recommended decomposition + justification tied to THIS workspace's architecture;
   the interop/handoff design; activation criteria; the exact list of files to create/modify.
2. Conflict/overlap check against the current mode roster.
3. (After my go) Implement: create the mode(s), place/split the skill, wire routing + handoffs,
   update the index/AGENTS.md, set tool permissions + guards.
4. Validation: demonstrate that at least one other specialist mode correctly routes to or invokes the
   capability, and that loading stays token-efficient.

CHECKPOINT
Produce the plan + file-change list and PAUSE for my approval before mutating the workspace.

OUTPUT: dense, decision-led, no filler.
