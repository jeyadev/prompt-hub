ROLE: My workspace already has an established modus operandi (MO) in its instruction
files. It is authoritative and must not be rewritten, reordered, or reworded. Your job
is to ADD a skill-routing and model-routing layer that complements it, so that each task
loads only the minimum skills it needs from ~/.copilot/skills/ (≈159 skills).
Optimize for: fewest always-loaded tokens, highest correct-skill selection, grounded outputs.

Edits to existing MO files are limited to a single hook line, and only with my approval.
Work on a git branch (init one if needed). Stop for my approval at each ⛔ checkpoint.
Delete nothing — archive instead.

PRECEDENCE: Existing MO > this routing layer > skill bodies. Where they conflict,
the MO wins; record the conflict in the report instead of resolving it yourself.

---

PHASE 0 — GROUND TRUTH (no design yet)

0a. Read the existing MO in full (every instruction file this platform loads).
    Summarize its structure, rules, and any references to specific skills, models,
    grounding, or task handling.
0b. GAP ANALYSIS: Classify each concern this restructure covers (skill selection,
    progressive loading, grounding, model routing) as: already covered by MO /
    partially covered / not covered. The layer will only address the
    partial and uncovered ones; for partial ones, extend rather than restate.
0c. List every skill the MO names explicitly. These are pinned: they stay in the
    auto-discovered path regardless of later tiering.
0d. Identify how this platform supports modular instructions (separate files,
    path-scoped instructions, file references) and confirm by test that a module
    referenced from the main file is actually loaded or followed.
1.  Determine, from the platform's docs/behavior in THIS environment, exactly how skills
    are discovered and loaded: which paths are auto-scanned, what is injected per turn
    (name? description? full body?), and how a skill gets activated. Cite the source.
    If unsure, say so and propose a test to find out.
2.  Inventory every skill: path, name, description, token count of description and body,
    referenced files/scripts, last-modified date.
3.  Measure baseline: total always-loaded tokens per turn (MO + skill metadata).

⛔ Show me: MO summary, gap analysis, pinned skills, hook mechanism, loading mechanics,
   inventory summary, and the baseline number.

---

PHASE 1 — SKILL GRAPH

1. Cluster skills into 10–20 domains by purpose (not by name).
2. Build a graph (skills-graph.json + a Mermaid view in skills-graph.mmd):
   nodes = skills; edges = depends-on | overlaps-with | supersedes | commonly-co-used.
   Mark pinned skills.
3. Flag: duplicates, near-duplicates (descriptions with high semantic overlap),
   orphans, stale skills, skills whose description doesn't match their body.
4. Recommend for each flagged skill: keep / merge / rewrite description / archive.
   Give the reason in one line each. Pinned skills may only be kept or get a
   description rewrite.

⛔ Show me the recommendations table. Apply only what I approve.

---

PHASE 2 — TIERED LOADING ARCHITECTURE

Design progressive disclosure with hard token budgets:
- L0 (always loaded): existing MO (unchanged) + one hook line + routing module
  (≤800 tokens) + domain index (one line per domain: trigger conditions → manifest path).
- L1 (loaded on domain match): domain manifest listing its skills with sharp,
  mutually exclusive "use when / don't use when" lines.
- L2 (loaded on skill match): the SKILL.md itself.
- L3 (loaded only if the skill says so): references, scripts, examples.

Implementation must match Phase 0 findings: if the platform auto-loads every
skill in the scanned path, move non-pinned L1/L2 skills to a non-scanned path
(e.g. ~/.copilot/skill-library/) and keep only pinned skills, the router, and a small
"always-on" core set in the scanned path. Use symlinks or a manifest so nothing is lost.

ROUTING MODULE (complements the MO; does not replace it):
- Write it as a separate file (e.g. skill-routing.instructions.md), ≤800 tokens.
- Connect it to the MO with ONE hook line, placed where I approve, e.g.:
  "Skill and model selection: follow skill-routing.instructions.md; this MO takes
  precedence on any conflict."
- The module contains only what the gap analysis showed is missing. Don't restate
  anything the MO already says; where needed, refer to the MO section by name.
- Write it in the MO's own terminology, tone, and formatting conventions so it reads
  as part of the same system.
- Header: "Complements MO version <git hash/date>. Re-check alignment if the MO changes."
- Routing rules:
  - Before any task: classify task → ≤1 domain → ≤2 skills (pinned skills always eligible).
  - Never load a skill "just in case." Never load a whole domain's skills at once.
  - If no domain matches with confidence, proceed without skills and say so in one line.
  - Ground outputs in files, tool results, or cited sources; mark anything inferred
    (unless the MO already defines grounding rules — then defer to them).
  - Name the loaded skills at the top of the response in one line, unless the MO
    already defines a response header.

---

PHASE 3 — MODEL ROUTING DOC (model-routing.md)

1. Detect which models are actually available to me in this environment. Don't
   assume names from training data.
2. If the MO already says anything about models, this doc extends it and states which
   MO lines it builds on. It must not contradict them.
3. Define task classes by characteristics, not by tool:
   reasoning depth, context size needed, ambiguity, cost of error, latency sensitivity,
   whether it's agentic/multi-step, and whether it's bulk/mechanical.
4. Map each class → primary model + fallback, with a one-line reason and when to escalate
   (e.g. two failed attempts, ambiguous spec, high blast radius).
5. Add a "review by" date 90 days out and the exact way to re-check available models.

---

PHASE 4 — PROVE IT

1. Routing eval set: 30 realistic tasks drawn from my skill domains, including
   10 ambiguous/overlapping ones and 5 that should use NO skill.
2. Regression set: 10 tasks the MO already handles well. They must produce the same
   behavior after the change (same skills used, same structure, same conventions).
   Any difference counts as a failure, not an improvement.
3. For each task, record the expected skills, then run routing (dry run: selection only)
   under the old and new structure.
4. Report: always-loaded tokens (before/after), routing accuracy, wrong/missed/extra
   selections, regression results, MO conflicts found, and every failure with its fix.

⛔ Final report + diff summary. Don't merge the branch until I approve.

---

OUTPUT FILES
- Hook line in the existing MO file (approved placement only)
- skill-routing.instructions.md (routing module)
- skills-index.md (domain index)
- domains/*.md (domain manifests)
- skills-graph.json, skills-graph.mmd
- model-routing.md
- evals/routing-evals.md, evals/regression-evals.md
- RESTRUCTURE_REPORT.md

RULES
- Be terse in all generated files. Every always-loaded line must earn its tokens.
- Never modify MO content beyond the approved hook line.
- If any phase reveals my plan is wrong (e.g. the platform or the MO already handles
  this), stop and tell me instead of building it.
