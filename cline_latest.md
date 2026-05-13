Right — you want the engineering discipline without the AEGIS-specific phase plan. The protocol is feature-by-feature, not phase-by-phase, but every feature gets the full treatment: stress-test → ADR-if-architectural → task → develop → test → review → ship → document.

Let me build `aegis-forge` — same heavy machinery as `aegis-cline`, but the spine is the **ideate → develop → test → ship cycle** rather than a four-phase roadmap.Now the centerpiece — the development guide that documents the full ideate → develop → test → ship cycle.Now the task templates — copy these to create new tasks.Now let me update the inherited files that reference the AEGIS roadmap to remove that context.Now update the modes for forge — drop the phase-bound `hardening` and `code-context` modes, replace with feature-driven ones.Now let me verify everything is in place and zip.`aegis-forge` — 37 files, full production discipline, no roadmap.

## What's different from `aegis-cline`

**Out:** the four-phase `ROADMAP.md`, the 13 pre-decomposed Phase 1/2 tasks (001–013), the AEGIS-specific onboarding-runbook, the "current phase" pointer in rule 01, the codebase-audit task. These were AEGIS roadmap context.

**In:** the **`DEVELOPMENT-GUIDE.md`** — the operating manual for the ideate → develop → test → ship cycle. This is the new spine.

**In:** `tasks/_template-*.md` — four templates (feature, spike, bug, refactor). You copy these to create real tasks as ideas survive stress-testing. No pre-decomposed roadmap; the task queue grows opportunistically.

**In:** `.roo/rules/10-testing-discipline.md` — testing as first-class. The four test layers (unit, integration, eval, manual), which layers a given task requires, coverage gates, pre-merge ritual, TDD vs test-along discipline, what to do when tests surface design problems.

**In:** `docs/testing-strategy.md` — the human-facing testing rationale. Where testing pays back fastest (agent boundaries, MCP adapters, the audit-write path, confidence-gate logic), eval layer in depth, the pre-merge ritual.

**Updated:** the six stress-test questions in rule 08. Question 3 changes from "Phase check" to "Dependency check" — same idea (does this depend on something not yet built?), feature-bound framing instead of phase-bound.

**Updated:** the `.roomodes` — six modes mapped to the cycle. `architect-aegis` (stress-test and ADR work), `feature-dev` (the main building mode), `mcp-integrator`, `eval-engineer`, `test-author` (tests-only — can't touch production code), `polish` (refactors only — can't change behavior).

## The cycle, in one diagram

```
  Ideate    →  Define    →  Develop   →  Test      →  Review    →  Ship      →  Document
  (rule 08)   (templates)   (rules 03+04)  (rule 10)   (rule 09)   (PR gates)   (STATE+memory)
   ↑                                                                                  |
   └─────────────────────────── feeds the next cycle ───────────────────────────────┘
```

## How to use it from where you are today

```bash
unzip aegis-forge.zip
cd aegis-forge

# Drop your demo code on top
rsync -av --exclude='.git' ../your-aegis-demo/ ./

git init && git add . && git commit -m "chore: bootstrap aegis-forge"
code .
```

Then in this order:

1. **Read `DEVELOPMENT-GUIDE.md` end to end.** ~15 minutes. It's the manual.
2. **Skim the rules** in `.roo/rules/` so you know what the agent will enforce. ~20 minutes for the first read.
3. **Seed `STATE.md`** with your demo's actual state. The agent can draft it; you correct it.
4. **First real cycle**: pick an idea, capture in INBOX, run the stress-test in `architect-aegis` mode, generate a task from the right template, claim it, develop. The cycle is self-teaching once you've done it twice.

## One stress-test on this workspace itself

The risk: the discipline becomes ceremony. You'll find yourself filling in task templates with copy-pasted sections that don't apply to the work, "testing requirements" that say "TBD," failure-mode entries that just restate the obvious. When that happens, the template is wrong for the situation, not the situation wrong for the template — edit the template, or drop the section, or write "N/A — bug fix, no new reach." Discipline that defeats its own purpose is worse than no discipline.

You'll know the workspace is working when you can point to a feature that shipped recently and explain it to a reviewer using nothing but the artifacts in the repo. When that becomes routine, the model is real. Until then, it's still being calibrated.
