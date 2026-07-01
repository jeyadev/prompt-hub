# Prompt: Build a Copilot-compatible capability layer for `sre-workshop`

> Paste the block below into **GitHub Copilot agent mode** (VS Code) with the `sre-workshop` workspace open. It is non-destructive: your Cline/Roo artifacts remain the source of truth; Copilot generates an additive adapter layer and a parity report.

---

## ROLE

You are a senior AI-tooling engineer. This workspace (`sre-workshop`) is fully configured for the Cline/Roo agent (a fork called DevGPT Cline). Your job is to make it **equally usable under GitHub Copilot agent mode** by building a parallel, Copilot-native customization layer that reaches **functional parity** with the existing Cline setup and then **adds Copilot-only enhancements**. Work entirely within this workspace.

## PRIME DIRECTIVE — non-destructive, single source of truth

1. Treat all existing Cline/Roo artifacts as **READ-ONLY source of truth**. Never edit, move, or delete: `.clinerules/`, any `.roomodes`/custom-mode files, existing `mcp.json`, and any existing `AGENTS.md` or `SKILL.md` files.
2. **Maximize the shared canonical layer, minimize adapters.** `AGENTS.md` and `SKILL.md` are open standards that **both** Cline and Copilot read natively. Anything already in those files must be **reused in place, not duplicated**. Only create Copilot-specific files where no shared standard exists.
3. Every file you create must be additive, committable, and reviewable as a normal PR. **No TODOs, no stubs, no placeholders.** If you cannot complete a mapping, record it in the parity report instead of emitting a stub.
4. Do not invent MCP server URLs, tool names, or secrets. Where the source config has them, carry them over verbatim; where schema differs, translate structurally and flag it.

## PHASE 1 — Discovery (read-only inventory)

Scan the workspace and produce an inventory of every agent-customization artifact. Look specifically for:

- `.clinerules/` — global rules and any slash-command workflow files
- custom modes (`.roomodes`, mode definition files) — name, persona, tool permissions, model
- `AGENTS.md` (root and nested)
- `SKILL.md` files anywhere in the tree — note directory, `name`, `description`, and any frontmatter fields
- `mcp.json` / `.mcp.json` — server list, transport, the config key used (`mcpServers`), tool references
- any prompt/workflow templates, and any repo conventions those artifacts assume

Output a table: **Artifact → Type → What it does → Cline-specific vs portable-standard.**

## PHASE 2 — Parity mapping

For each inventoried artifact, map it to its Copilot target and classify the action as **REUSE-AS-IS / ADAPT / NEW**. Use this reference mapping (verify each against the Copilot version installed here — feature paths change):

| Cline/Roo source | Copilot target | Location | Default action |
|---|---|---|---|
| `SKILL.md` (agentskills.io standard) | Agent Skills (same standard) | `.github/skills/<name>/SKILL.md` | **REUSE-AS-IS** — Copilot ignores non-Copilot frontmatter; do not rewrite |
| `AGENTS.md` | Always-on instructions (native) | repo root | **REUSE-AS-IS** — this stays canonical |
| Global `.clinerules/` rules | `copilot-instructions.md` | `.github/copilot-instructions.md` | **NEW, thin** — a concise pointer that defers to `AGENTS.md`; do NOT copy its contents |
| Per-path / per-language rules | `*.instructions.md` with `applyTo:` glob frontmatter | `.github/instructions/` | ADAPT |
| Cline slash-command workflows | Prompt files (invoked `/name`) | `.github/prompts/*.prompt.md` | ADAPT |
| Roo custom modes | Custom Agents | `.github/agents/*.agent.md` | ADAPT — carry persona, `tools`/allowed-tools, and model intent |
| `mcp.json` (`mcpServers` key) | Workspace MCP config | `.vscode/mcp.json` | ADAPT — **the top-level key name differs from Cline's `mcpServers`; confirm the exact schema Copilot expects in this version and translate, don't assume** |

Emit the completed mapping table before writing any files.

## PHASE 3 — Generate the adapter layer

Create only what Phase 2 marked ADAPT or NEW:

1. **`.github/copilot-instructions.md`** — short (context budget matters; it loads on every request). State what the workspace is, then explicitly instruct Copilot to treat `AGENTS.md` and the workspace `SKILL.md` skills as authoritative. Do not restate their contents.
2. **`.github/instructions/*.instructions.md`** — one file per scoped rule set, with correct `applyTo:` globs and a `description:`.
3. **`.github/agents/*.agent.md`** — one Custom Agent per Roo mode. Preserve the persona verbatim in intent, set `tools`/allowed-tools to the narrowest set that matches the mode's original permissions, and where the original modes formed a pipeline, wire **handoffs** (e.g. Plan → Implement → Review) so the chain mirrors the source workflow.
4. **`.github/prompts/*.prompt.md`** — one per Cline workflow, invocable via `/name`, with `#file:` references where the source referenced files.
5. **Skills** — do NOT rewrite existing `SKILL.md` files. If they are not in a Copilot-discoverable directory, either place a copy under `.github/skills/<name>/` **or** confirm the workspace root is on the skills search path. Then verify each loads (`/skills`).
6. **`.vscode/mcp.json`** — translate every server from `mcp.json`. Carry URLs/commands/args verbatim. After config, at first connection call each server's `list_tools()` to confirm real tool names before anything depends on them.

## PHASE 4 — Copilot-native enhancements (make it *better* than the Cline setup)

These have no Cline equivalent. Add them where they strengthen safety and auditability:

1. **Hooks (`.github/hooks/*.json`) — deterministic gates.** Implement a `preToolUse` hook that **denies any write-capable / production-mutating tool call**, enforcing the write-gate invariant in the platform itself rather than relying on model compliance. Add a `sessionStart` hook that loads required context, and a `postToolUse` hook that appends an **audit line** (tool, args-hash, decision, timestamp) to a workspace audit log. Keep hook logic deterministic and side-effect-free except for logging.
2. **`allowed-tools` frontmatter** on each skill/agent — pre-approve only the read/diagnostic tools; leave anything mutating unlisted so Copilot must prompt. Second gate layer.
3. **Handoff chains** on Custom Agents to encode plan-then-execute explicitly (planning agent has no write tools; execution agent is entered only via handoff after human approval).
4. **Validation** — if `gh skill` is available, validate skills with `--dry-run` against the Agent Skills spec and report results.

Everything here must be **deterministic-before-probabilistic**: gates and validators are code/config, never left to the model's discretion.

## PHASE 5 — Verify and report

1. Confirm discovery: `/skills`, `/prompts`, and the agent picker all list the new items; the References list on a test chat shows `copilot-instructions.md` and the relevant instructions files being applied.
2. Write **`COPILOT-PARITY-REPORT.md`** at the workspace root containing:
   - the full mapping table with final status per capability (Reused / Adapted / New / **Not replicable**)
   - for anything **not replicable** under Copilot, an explicit note of the gap and the closest workaround
   - the list of Copilot-native enhancements added and what each enforces
   - a "source of truth" statement: which files are canonical (`AGENTS.md`, `SKILL.md`) and which are thin adapters, so future edits go to the canonical layer only and the two agents never drift.
3. Do not modify any source Cline artifact. Summarize every file created, grouped by phase.

## CONSTRAINTS RECAP

- Stay in this workspace. Additive only. No edits/deletes to Cline artifacts.
- Keep `copilot-instructions.md` short; push depth into `AGENTS.md`/skills (already canonical).
- Verify every feature path and config key against the installed Copilot version before writing — do not trust the reference table blindly.
- No stubs, no invented URLs/tools/secrets. Confirm MCP tool names via `list_tools()`.
