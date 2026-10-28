Change of goal. This session now builds a working CDS model in cc-pathfinder that can scale to other apps later; CDS only for now. The feasibility analysis moves to a new session you will create. Keep all rules from my first message: AGENTS.md lifecycle, read-only inputs, no AEGIS kit code copying, cite paths, "I don't know" over invention, no commit or push unless I ask.

STEP 1 — HANDOFF. Write what you've found so far on Q1–Q4 to repos/cc-pathfinder/docs/lifecycle/feasibility/handoff-from-build.md. Then stop working on those questions.

STEP 2 — CREATE FEASIBILITY SESSION. Start a new session with the brief below, verbatim. Don't wait for it to finish.

--- BEGIN FEASIBILITY BRIEF ---
You are in agentic-ai-workspace. Follow AGENTS.md: state the lifecycle stage (likely Stage 1 Architect feeding a Stage 3 option comparison; confirm with me if unsure), load only that stage's specialist, surface failure modes before solutions, and say "I don't know" rather than invent.

Context: a parallel build session is making Pathfinder work for CDS via the CDS MCP POC in a separate worktree. Your job is the scaling question: how Pathfinder should grow to N apps. Analysis only. Don't modify code, branches or worktrees. Read cc-pathfinder from develop, not the build branch. Write artifacts only to repos/cc-pathfinder/docs/lifecycle/feasibility/.

Inputs (read-only):
- repos/cc-pathfinder (base develop), plus its instruction files listed in repos/README.md
- C:\Users\I817586\.copilot\session-state\8d2533fc-b884-4933-8266-f0ff279046bf\files\pathfinder-architecture.html. This is a derived summary, not ground truth; verify claims against code.
- CDS MCP POC: C:\Users\I817586\ds\projects\cds-repos\amcds-migration-utility.worktrees\mcp-availability-performance-queries
- AEGIS kit, branch feature/cockpit-security: C:\Users\I817586\ds\projects\sre-apps\aegis-kit.worktrees\cockpit-security, mapped by kit-manifest.yaml (pinned 008f1ea). Prefer committed content and flag any reliance on uncommitted files.
- docs/lifecycle/feasibility/handoff-from-build.md (findings from the build session)

Answer, scoped to Common Capabilities apps:
1. TOOL OVERLAP: Matrix of Pathfinder CDS tools vs POC MCP tools: duplicate, gap, or conflict.
2. CURRENT vs MCP APPROACH: Rule-agent orchestration inside Pathfinder vs per-app MCP servers. Score both on: routing determinism, auditability, write-action safety, identity propagation (ADFS user → MCP → Splunk/DB), per-app onboarding cost, contract versioning, testability, blast radius of one bad server, and ownership of each server.
3. MULTI-SERVER: One MCP server per app with the orchestrator delegating and composing. Cover tool-name collisions, partial failure, cross-app questions, latency fan-out, transport, and where servers run in Pathfinder's hosting environment.
4. REUSE BOUNDARY: For each AEGIS kit component, state: reuse as dependency, reuse as pattern only, or not applicable.
5. FEASIBILITY BAR: What evidence would make each option feasible, and does it exist today?
6. MIGRATION PATH: From today's state to the recommended option, sequenced, with rollback at each step.

Cite a file path for every claim. List failure modes and open questions, each with a suggested owner, before recommending anything. Start by showing your plan and stop for my go.
--- END FEASIBILITY BRIEF ---

STEP 3 — BUILD CHECKPOINT. State the new lifecycle stage and specialist per AGENTS.md. Show your plan: files to read, files to change, branch name off develop, done criteria. Work in a separate git worktree so develop stays untouched for the feasibility session. Stop for my go.

Build scope once I approve:
a. FIRST QUESTION: Where will the working model run: local (no Mercury/Splunk/document tools) or a dev environment? How does the CDS MCP POC reach its data today? Answer from the POC worktree, or say "I don't know".
b. VERIFY: Check pathfinder-architecture.html claims against cc-pathfinder code before relying on them. List any mismatches.
c. CLASSIFY CDS queries: (a) KB, (b) single live lookup, (c) troubleshoot, (d) write/workflow. Map each to the current path and the POC MCP tool that would serve it, including the queries already tested in the POC.
d. BUILD MINIMUM: Pathfinder's CDS app consumes the CDS MCP POC server as an MCP client, behind a flag, with the existing in-process tools as fallback. Classes a–c only. Class d stays on the existing path with confirmation enforced in code.
e. SCALE SEAMS ONLY: MCP endpoint and tool list come from apps/cds/app.yaml. No CDS-specific logic in core/. Per-app flag, default off for other apps.
f. DONE = 20 golden CDS queries in evals/cds_golden.yaml pass on the branch, using recorded fixtures if live access isn't available (say which). Report main vs branch.
