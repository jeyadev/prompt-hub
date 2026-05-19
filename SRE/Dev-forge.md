You are architect-forge mode. Your task is to generate a project called 
`dev-workspace` — a reusable, generic development workspace template that 
Jey drops any code repository into, runs one onboarding prompt, and is 
immediately ready to develop features inside that repo using Cline/Roo.

This is NOT a scaffolding tool. It IS the development environment itself.
The workspace lives alongside the code repo, not inside it.

---

## THE WORKFLOW THIS ENABLES

Step 1 → Jey opens dev-workspace in a new Cline window
Step 2 → Copies repo2 (the target codebase) into dev-workspace/repo/
Step 3 → Runs prompts/00-ONBOARD.md inside Cline
Step 4 → The onboarding agent scans repo2 and UPDATES all context 
          documents in-place (rules, memory-bank, state, CLAUDE.md, .roomodes)
Step 5 → Jey is ready to develop. All modes are live and repo-aware.

---

## PROJECT STRUCTURE TO CREATE

dev-workspace/
├── README.md                        # How to use this workspace
├── CLAUDE.md                        # Operating contract — UPDATED by onboarding
├── .clinerules                      # Cline shim — points to .roo/rules/
│
├── repo/                            # DROP TARGET CODE REPO CONTENTS HERE
│   └── .gitkeep
│
├── .roo/
│   └── rules/
│       ├── 01-project.md            # What the project is — UPDATED by onboarding
│       ├── 02-architecture.md       # Architecture invariants — UPDATED
│       ├── 03-coding-standards.md   # Language/framework conventions — UPDATED
│       ├── 04-feature-protocol.md   # How features are built in this repo — UPDATED
│       ├── 05-testing-discipline.md # Test framework, coverage rules — UPDATED
│       ├── 06-task-discipline.md    # How to work: branches, commits, PRs — STATIC
│       └── 07-agent-boundaries.md  # What each mode can and cannot touch — STATIC
│
├── memory-bank/
│   ├── architecture.md              # Inferred architecture decisions — UPDATED
│   ├── tech-decisions.md            # Known decisions and their rationale — UPDATED
│   ├── known-patterns.md            # Recurring patterns found in codebase — UPDATED
│   ├── external-dependencies.md     # APIs, DBs, queues, third-party — UPDATED
│   └── onboarding-log.md            # Timestamped record of each onboarding run
│
├── state/
│   ├── current-focus.md             # Active development context — UPDATED
│   ├── open-questions.md            # Genuine ambiguities from scan — UPDATED
│   ├── known-debt.md                # Tech debt spotted during scan — UPDATED
│   └── progress.md                  # Task tracking across sessions — STATIC template
│
├── tasks/
│   ├── _template-feature.md         # Copy to create a feature task
│   ├── _template-bug.md             # Copy to create a bug task
│   ├── _template-refactor.md        # Copy to create a refactor task
│   └── _template-spike.md           # Copy to create a research spike
│
├── prompts/
│   ├── 00-ONBOARD.md                # MAIN PROMPT — run after dropping repo
│   ├── 01-new-feature.md            # Prompt to kick off a new feature
│   ├── 02-write-tests.md            # Prompt to generate tests for existing code
│   ├── 03-refactor.md               # Prompt to refactor a module
│   └── 04-refresh-state.md          # Re-sync state docs mid-development
│
└── .roomodes                        # All modes — UPDATED by onboarding

---

## ROOMODES — DEFINE ALL FIVE

architect
  → For: system design, ADRs, stress-testing ideas, dependency decisions
  → Can read: entire workspace + repo/
  → Can write: memory-bank/, state/, .roo/rules/, tasks/
  → Cannot write: repo/ (no production code changes)

feature-dev
  → For: implementing features, fixing bugs, wiring integrations
  → Can read: entire workspace + repo/
  → Can write: repo/ (production code only)
  → Must reference: active task file in tasks/ before starting
  → Cannot write: .roo/rules/, memory-bank/ (no context drift mid-feature)

test-engineer
  → For: writing tests, test fixtures, test utilities — nothing else
  → Can read: entire workspace + repo/
  → Can write: test files inside repo/ only (files matching test patterns)
  → Cannot write: production source files, rules, memory-bank
  → Must reference: 05-testing-discipline.md before writing any test

refactor
  → For: structural improvements, no behavior changes
  → Can read: entire workspace + repo/
  → Can write: repo/ source files only
  → Hard constraint: every refactor must have existing tests passing before and after
  → Cannot introduce new functionality

onboarding
  → For: running 00-ONBOARD.md only — scan and update context
  → Can read: entire repo/
  → Can write: CLAUDE.md, .roo/rules/, memory-bank/, state/, .roomodes
  → Cannot write: repo/ (never touches the codebase itself)
  → Activates only when 00-ONBOARD.md is the active prompt

---

## 00-ONBOARD.md — WRITE THIS FILE IN FULL

This is the most important file in the project.
It must be a complete, runnable Cline prompt, not a description of one.
Write it as the actual prompt text that the agent will execute.

The prompt must instruct the onboarding agent (in onboarding mode) to:

PHASE 1 — SCAN repo/ (read-only, no writes yet)

Scan the following and build an internal understanding before writing anything:

  STACK
  - Primary language(s) and version signals (package.json, pyproject.toml, 
    go.mod, Cargo.toml, etc.)
  - Framework(s) and their versions
  - Package manager (npm/yarn/pnpm/pip/poetry/cargo/etc.)
  - Monorepo or single package?

  ARCHITECTURE
  - Project type: API, frontend SPA, full-stack, CLI tool, agent/AI system, 
    data pipeline, library, mobile — or combination
  - Architecture pattern: MVC, layered, hexagonal, event-driven, 
    agent-orchestrated, microservice, etc.
  - Key entrypoints: main files, route registrations, agent orchestrators, 
    CLI commands
  - Module/package structure: how is code organized? feature-based, 
    layer-based, domain-based?

  TESTING
  - Test framework (jest, pytest, vitest, go test, etc.)
  - Test file location pattern (co-located, __tests__/, spec/, tests/)
  - Coverage tooling if present
  - Existing test coverage signal (many tests? sparse? none?)

  DEPENDENCIES
  - Database(s) and ORM/client
  - External APIs or services being called
  - Queue or pub/sub systems
  - Auth mechanism if present

  CI/CD
  - CI system (.github/workflows, .gitlab-ci, Jenkinsfile, etc.)
  - Deployment target signals (Dockerfile, fly.toml, vercel.json, 
    serverless.yml, k8s manifests, etc.)

  EXISTING DOCUMENTATION
  - README quality and content
  - Any ADRs, architecture docs, or inline doc comments worth preserving
  - CHANGELOG or git log patterns

  DEBT AND AMBIGUITY
  - Obvious TODO/FIXME/HACK comments
  - Inconsistent patterns (mixed paradigms, inconsistent naming, etc.)
  - Missing tests on critical paths
  - Anything you cannot determine with confidence — flag it, don't guess

PHASE 2 — CONFIRM (stop and show the developer)

Display a structured scan report:

  PROJECT: [name inferred from package.json/pyproject/README/folder]
  TYPE: [what kind of project]
  STACK: [language, framework, package manager]
  ARCHITECTURE: [pattern, entrypoints, module structure]
  TESTING: [framework, pattern, coverage signal]
  DEPENDENCIES: [external deps found]
  CI/CD: [what was found]
  OPEN QUESTIONS: [what could not be determined]
  DEBT SIGNALS: [TODOs, inconsistencies, gaps]

Then ask: "Does this look right? Anything to correct or add before I update 
the workspace context?"

WAIT for explicit confirmation (yes / corrections) before writing anything.

PHASE 3 — UPDATE all context documents in-place

After confirmation, update each file using the scan findings.
Do not use placeholder text. Every sentence must reflect actual findings.
If something was not found, say "not detected" — never fabricate.

UPDATE CLAUDE.md
  - Project name, type, stack
  - How features are to be built in this specific codebase
  - Operating contract for agents working in this repo

UPDATE .roo/rules/01-project.md
  - What this project is, what it does, who uses it
  - Key entrypoints and how the system starts
  - Module structure summary

UPDATE .roo/rules/02-architecture.md
  - Architecture pattern with evidence from the codebase
  - Non-negotiable architectural constraints inferred from structure
  - Layers and their responsibilities

UPDATE .roo/rules/03-coding-standards.md
  - Language-specific conventions observed in the codebase
  - Naming patterns actually used
  - File/folder naming conventions
  - Import patterns, error handling style

UPDATE .roo/rules/04-feature-protocol.md
  - How a feature should be built in THIS codebase specifically
  - Where new routes/handlers/components/agents belong
  - How this codebase wires new functionality

UPDATE .roo/rules/05-testing-discipline.md
  - The actual test framework and how to use it
  - Where test files go in this repo
  - What test layers exist or should exist
  - Coverage expectations

UPDATE memory-bank/architecture.md
  - Full architecture picture derived from scan
  - Diagram in ASCII or markdown table if helpful

UPDATE memory-bank/tech-decisions.md
  - Observed decisions (e.g., "uses Prisma not raw SQL", "REST not GraphQL")
  - Note: these are inferred, not confirmed by the original author

UPDATE memory-bank/known-patterns.md
  - Actual patterns found in code (how errors are handled, how services are 
    structured, how tests are written, how API clients are wrapped)

UPDATE memory-bank/external-dependencies.md
  - Every external dependency found with its purpose

UPDATE state/current-focus.md
  - Set to: "Freshly onboarded — no active feature in progress"
  - Include date of onboarding

UPDATE state/open-questions.md
  - Populate with genuine ambiguities from scan
  - Each question should be specific and answerable

UPDATE state/known-debt.md
  - Populate with actual debt signals found

UPDATE .roomodes
  - Add repo-specific context to each mode description
  - Update test file patterns to match what was found
  - Update feature-dev write paths to match repo structure

PHASE 4 — HANDOFF

Print a final summary:
  ✓ List every file updated
  ✓ One-line description of what was written into each
  ✓ Top 3 suggested first tasks based on what you found
  ✓ Any open questions the developer should answer before starting

---

## STATIC FILE RULES

06-task-discipline.md and 07-agent-boundaries.md are NEVER updated by onboarding.
They contain workspace-level conventions that apply to all repos equally.
Write them with strong, final content now.

06-task-discipline.md must cover:
  - Every feature starts from a task file in tasks/
  - Branch naming from task ID
  - Commit message format
  - When to update state/progress.md
  - Definition of done

07-agent-boundaries.md must cover:
  - The five modes and exactly what each can/cannot write
  - Why these boundaries exist (context drift prevention)
  - What happens when a mode needs to step outside its boundary 
    (stop, flag, switch modes)

---

## QUALITY RULES FOR ALL TEMPLATE CONTENT

Static content (before onboarding runs) must:
  - Use clear {{PLACEHOLDER}} markers for every repo-specific value
  - Never contain AEGIS-specific, JPMC-specific, or Python-specific assumptions
  - Be stack-agnostic: equally valid for a React app, Go API, Python agent, 
    or Node CLI

After onboarding runs:
  - Zero placeholders should remain — every {{PLACEHOLDER}} gets replaced
  - Every document must read as if written by someone who read the full codebase
  - Open questions must be real, not boilerplate

---

## BUILD ORDER

1. README.md
2. CLAUDE.md (template version with placeholders)
3. .roo/rules/ — all 7 files (01-05 as templates, 06-07 as final static content)
4. memory-bank/ — all 5 files as templates
5. state/ — all 4 files as templates
6. tasks/ — all 4 task templates
7. prompts/00-ONBOARD.md — write in FULL as a runnable prompt
8. prompts/01 through 04 — write in full
9. .roomodes — all five modes defined
10. .clinerules — shim

Do not zip. Confirm structure with me before we proceed.
