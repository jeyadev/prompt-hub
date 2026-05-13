Here's your turnkey AEGIS workspace for Roo Code. Unzip, `git init`, open in VS Code, and the rules load automatically.

**What's inside**

- **`.roo/rules/` — seven numbered files.** The agent's operating contract. Project context, architecture invariants (the 10 non-negotiables), coding standards, agent-design principles, security/governance, task discipline, MCP patterns. Each one is short on purpose — they're meant to be read on every new session.
- **`.roomodes` — six custom modes.** `architect-aegis` (docs only, no code), `strategist-dev`, `mcp-integrator`, `eval-engineer`, `hardening`, `code-context`. Each scopes file-edit permissions and inherits a tailored system prompt.
- **`.clineignore` and `.gitignore`.** Secrets, audit exports, large fixtures kept out of reach.
- **`PRD.md`, `ARCHITECTURE.md`, `ROADMAP.md`.** The project's anchor docs — what AEGIS is, the layered architecture with the orchestrator-workers diagram, the four-phase plan as workstream tables that map to the modes.
- **`tasks/` — nine decomposed task files.** `001–005` are Phase 1 hardening (Postgres+pgvector, Vault, audit log with hash chain, eval harness v1, OTel). `010–013` are the Phase 2 code-context stack (repo onboarding, tree-sitter indexing, pgvector store + hybrid retrieval, incident-to-code resolver). Each follows the same template: Phase / Mode / Effort / Depends / Blocks / Objective / Scope (in/out) / Acceptance criteria / Implementation notes / Testing / Exit artifacts.
- **`docs/failure-modes.md`** seeded with FM-001 through FM-006 — the catalogue every PR that adds reach must update.
- **`docs/eval-design.md`** — the three-layer scoring methodology (deterministic / judge LLM / human spot check) and golden-set curation rules.
- **`docs/onboarding-runbook.md`** — the Shadow → Suggest → Approve → Auto ladder for app teams.
- **`pyproject.toml`** with Python 3.12, async SQLAlchemy 2.0, asyncpg, Anthropic SDK, MCP, hvac, tenacity, pybreaker, OTel suite, pygit2, tree-sitter, ruff/black/mypy strict/pytest.

**How to use it**

1. Unzip wherever your AEGIS code will live.
2. `git init && git add . && git commit -m "chore: bootstrap AEGIS workspace"`
3. Open in VS Code with Roo Code installed; confirm Opus 4.6 is the model and the rules show as loaded.
4. Start with task 001. In Roo Code: switch to `hardening` mode, then prompt: *"Execute task 001. Read `.roo/rules/` first. Confirm scope before writing code."*
5. PR-per-task, `/newtask` between them, append to `.roo/memory.md` when you learn something durable.

**One stress-test you should know about**

The rules cap Strategist thinking at 8K context + 4K output, Sub-Agent at 4K+2K, classifier at 2K+512. These are starting points — if the eval harness shows Opus needs more headroom for code-aware runbooks in Phase 2, raise the Strategist's cap deliberately, not silently. The point of the cap isn't the number; it's that *there is one*.

Good — this is the question that determines whether the workspace stays sharp or rots into a junk drawer. Most agent-driven projects fail here, not at the code.

The principle: **separate ideation from commitment**. Ideas are cheap and should flow freely. Tasks are expensive and should survive scrutiny. The protocol enforces the boundary.

## The protocol — five stages

**1. Capture (5 min, no quality bar)**

Append to `ideas/INBOX.md`: date, one paragraph, source/trigger. That's it. No design, no scope, no roadmap mapping. The goal is to *not lose the thought*, not to evaluate it. You'll capture ideas in WhatsApp at 11pm, during a Splunk dashboard scroll, while debugging an unrelated thing. INBOX absorbs them all.

**2. Stress-test (15–30 min in `architect-aegis` mode)**

Open Roo Code, switch to `architect-aegis`, paste the idea. The standing prompt is:

> *"Stress-test this idea against AEGIS rules and roadmap. Be ruthless. Run the six questions in `.roo/rules/08-feature-protocol.md`. Tell me whether it dies, gets sharper, or gets committed — and to what artifact."*

The architect mode can only write `.md` and `.txt`, so it cannot leak into code. It produces a structured assessment, not an implementation.

**3. Classify — six possible outcomes**

The stress-test routes the idea to exactly one of:

- **Reject** → `ideas/rejected.md` with the rationale. Don't delete; ideas come back, and your future self deserves to know why you said no the first time.
- **Defer** → `ideas/deferred.md` with the target phase and the *condition* that would activate it ("when 5 apps are in Approval mode," not "later").
- **Spike** → `tasks/9NN-spike-<topic>.md` — time-boxed research (default 1 day), output is a doc, not code.
- **ADR** → `docs/adr/NNNN-<title>.md` — architectural decision needed first; the implementation tasks come after the decision is recorded.
- **Task** → `tasks/NNN-<title>.md` — committed work, on the roadmap, ready for a `code`-mode session.
- **Rule/invariant change** → updates `.roo/rules/`. Rare, deliberate, announced in the PR. Affects every future task.

**4. Commit**

Generate the artifact in the right place. Add one line back in `ideas/INBOX.md` pointing to it. If it adds a workstream, cross-link from `ROADMAP.md`. The idea now lives in exactly one place; it's no longer "an idea," it's a thing with a status.

**5. Cadence — 30 min weekly**

Without this, the protocol decays. Three sweeps:

- INBOX older than 7 days: stress-test or reject. No idea sits in INBOX for two weeks unexamined.
- Deferred items older than 90 days: revive or kill. The world has moved.
- Tasks blocked > 2 weeks: unblock or close. Open tasks that aren't moving signal a deeper problem.

## The six stress-test questions

These are the heart of the protocol. The architect mode runs them; you read the answers.

1. **Invariant check.** Does this require violating any of the 10 invariants in rule 02? If yes — *kill the idea, or amend the invariant in writing first*. Never both at once.
2. **Problem check.** Whose pain does this solve, named specifically? If the answer is "we might want this someday," reject or defer. AEGIS only adds reach when a real user has a real problem.
3. **Phase check.** Does this depend on infrastructure we haven't built? Then it's a Phase N+1 idea at best — defer with the activation condition.
4. **Smallest version.** What's the cheapest thing we could ship that would tell us if the idea is real? If you can't name a < 1-week version, the idea isn't ready.
5. **Failure mode.** What's the worst thing it does broken? Can existing gates contain that? If not — what new gate, written down before the feature ships?
6. **Carrying cost.** What does it cost to *maintain*, not build? Every feature is a tax forever. Be honest about the tax.

A single "no" on questions 1, 2, or 3 should kill or defer the idea. Questions 4–6 sharpen it.

## What this gives you

Cline doesn't drift into building random features because *you* don't. The architect mode is the gatekeeper; the rules are the criteria; the artifacts are the receipts. When you brainstorm with Claude in chat about AEGIS, the protocol fires automatically — the agent knows to stress-test before scaffolding.

## What I'm adding to the workspace

Let me extend the workspace with the protocol artifacts.

Re-zipped with the protocol layer wired in. What was added:

- **`.roo/rules/08-feature-protocol.md`** — codified as a rule so the agent enforces it. When you say *"let's add X"*, the agent runs the six stress-test questions before doing anything else; it cannot bypass it.
- **`ideas/INBOX.md`, `ideas/deferred.md`, `ideas/rejected.md`** — the three-file pipeline. INBOX is no-quality-bar capture; deferred carries activation conditions; rejected keeps the history so the same idea coming back gets fresh treatment.
- **`docs/adr/0000-template.md`** — the ADR template. Copy to `0001-<title>.md`, `0002-<title>.md` as decisions arise. Two-option minimum (if you only have one option, you don't need an ADR).
- **`docs/feature-protocol.md`** — the human-facing version with the ASCII flow diagram and the longer rationale.

## Your Sunday-evening ritual

Block 30 minutes on the calendar. Open Roo Code in `architect-aegis` mode. Run this:

> *"Weekly protocol review. List everything in `ideas/INBOX.md` older than 7 days, everything in `ideas/deferred.md` older than 90 days, and every task in `tasks/` blocked more than 2 weeks. For each, recommend an action: stress-test, revive, kill, or escalate."*

The agent produces the worklist. You make the calls. The whole thing takes 20 minutes once you're in rhythm.

## One stress-test on this protocol itself

The risk: **it becomes ritual without substance.** You run the six questions, the agent answers them confidently, you ratify the answers, the idea ships. That's the protocol pretending to do its job while doing nothing.

The defence: *you* answer at least one of the six questions yourself, in your own words, before reading the agent's assessment. Questions 2 (whose pain) and 4 (smallest version) are the ones you should always answer first — they're the ones where your skin in the game matters. The agent can map the architecture; only you know whether the user is real and the cheap version is acceptable.

Test it on the next idea that hits your INBOX. If it's still useful at task 020, the protocol is real.
