# Runbook: Agentic loops over `git worktree` with GitHub Copilot

> Operating model for running iterative and parallel Copilot agent loops against `sre-workshop` (or any repo) without agents stepping on each other. Human-facing runbook — do **not** paste into an agent.

## Mental model (3 primitives)

| Primitive | What it is | SRE framing |
|---|---|---|
| **Worktree** | A real, isolated checkout of one branch at its own path, sharing the repo's git objects/history | An isolated **vertical slice** — its own working dir + index, one branch, one blast zone |
| **Loop** | A bounded agentic iteration: plan → act → verify → repeat until an exit condition | Your **harness cycle** with a verification gate; never unbounded |
| **Fan-out** | N worktrees running N agent loops concurrently | Parallel slices; isolation is what makes parallelism safe |

## Two operating modes — pick per task

| | **Manual (CLI)** | **Managed (Copilot app / IDE background agent)** |
|---|---|---|
| Worktree lifecycle | You run `git worktree` yourself; the standalone CLI has **no** worktree management commands | App/IDE **auto-creates and cleans up** a worktree per session |
| Parallelism | As many `copilot` sessions as you script | Up to **10 parallel sessions**, each in its own worktree |
| Session modes | Interactive, or headless via prompt flag | **Interactive / Plan / Autopilot**, plus My Work dashboard + Agent Merge |
| Best for | VDI, scripted loops, full control, audit | Dashboard oversight, issue→PR flow, hands-off cohorts |
| Requires | Git ≥ 2.5 | Paid Copilot plan (Business/Enterprise at JPMC) |

Rule of thumb: **manual CLI for scripted/gated loops you want to audit line-by-line; the app when you want a supervised fleet with merge control.**

---

## Pattern A — Parallel fan-out (one worktree per slice)

Independent slices, run concurrently, isolated. Manual CLI so every step is scriptable and reviewable.

```bash
WT="$HOME/worktrees/sre-workshop"          # keep all worktrees in one place
SLICES="parity-layer exception-monitor kg-retrieval"

for slice in $SLICES; do
  git worktree add "$WT/$slice" -b "feat/$slice"          # isolated branch + dir
  ( cd "$WT/$slice" \
      && copilot -p "$(cat tasks/$slice.prompt.md)" ) &    # headless session, backgrounded
done
wait                                                        # block until all loops finish
```

- `copilot -p "<prompt>"` runs a non-interactive session. **Confirm the exact flag and any tool-approval flags with `copilot --help`** — CLI flag names move between releases.
- First run in a new worktree may require **directory trust** before Copilot will execute commands.
- Cap concurrency to respect **credit/agent-minute limits and org policy** — parallel autopilot burns allowance fast. Check Copilot org settings before large fan-outs.

## Pattern B — Bounded verify-loop inside one worktree

Single slice, iterate until a deterministic check passes. This is the harness loop — the exit condition is a real test, not the model's self-assessment.

```bash
cd "$HOME/worktrees/sre-workshop/parity-layer"

for i in $(seq 1 5); do                                    # HARD iteration cap
  copilot -p "Address failing checks only. Run: pytest -q. Stop when green."
  pytest -q && { echo "green on iteration $i"; break; }
done
```

- The cap (`5`) is the balancing loop — it prevents a reinforcing spiral of an agent thrashing on a problem it can't solve.
- Gate on a **deterministic signal** (test exit code, linter, schema validator) before any probabilistic "looks done." Deterministic-before-probabilistic applies to the loop's exit condition too.

## Pattern C — App-managed cohort

When you'd rather not manage worktrees: open the Copilot app → My Work → start sessions from prompts/issues. Each gets an auto-managed worktree; use **Plan** mode to review the plan before **Autopilot** executes; use **Agent Merge** to carry a branch through CI + review. No manual `git worktree` at all.

---

## Guardrails (the part a naive setup misses)

1. **Repo-level config is SHARED across all worktrees.** `.github/copilot-instructions.md`, `.github/skills/`, `.github/agents/`, `.github/hooks/`, `.vscode/mcp.json` live at repo root and apply to every worktree/session simultaneously. Editing them in one slice changes the behavior of every parallel loop. **Blast radius = all sessions.** Treat config edits as their own dedicated, non-parallel change — never mutate shared config inside a fan-out loop.
2. **MCP-backed loops hit shared external systems.** Parallel autopilot sessions all pointing at the same Splunk / Jira / Dynatrace MCP servers means N agents concurrently querying (or acting on) the same backends. Keep the **`preToolUse` deny-gate** (from the parity prompt) active for anything write-capable, and prefer read-only tool grants in autopilot. In a regulated env this is the difference between a safe fleet and N uncontrolled actors.
3. **Bound every loop.** Long, tool-heavy CLI sessions in worktrees are known to corrupt session state (upstream issue). Prefer several short, restartable loops over one marathon session; the iteration cap doubles as a corruption circuit-breaker.
4. **Conflicts resolve at integration, not mid-loop.** Isolated worktrees can't overwrite each other, but their branches can still conflict on merge — that's the correct, reviewable place to resolve, exactly like parallel human devs.
5. **Cleanup is not optional.** Orphaned worktrees accumulate silently (drift). Reap them:

```bash
git worktree remove "$HOME/worktrees/sre-workshop/parity-layer"
git worktree prune            # clean stale metadata
git worktree list             # audit what's still checked out
```

## Pre-flight checklist

- [ ] Git ≥ 2.5; worktrees rooted under one parent dir
- [ ] `copilot --help` checked for current headless + tool-approval flag names
- [ ] Shared config (`copilot-instructions.md`, skills, agents, **hooks**, `mcp.json`) committed and reviewed **before** fan-out
- [ ] `preToolUse` write-gate hook active; autopilot sessions granted read-only tools
- [ ] Iteration cap on every loop; exit condition is a deterministic check
- [ ] Concurrency within org credit/policy limits
- [ ] Cleanup step scheduled (`worktree remove` / `prune`)
