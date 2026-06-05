# Splunk SRE Kit — Roo Code + Cline (CDS / AWM)

A drop-in context that turns Roo Code **or** Cline into a principal-SRE Splunk engineer for the
CDS platform: scoped cost-aware SPL, golden-signal / error-budget searches, and Dashboard Studio
JSON — all behind hard guardrails so an autonomous agent can't fire an `index=*` at shared JPMC
Splunk or publish a dashboard straight to prod.

## What's in here
| File | Purpose | Used by |
|---|---|---|
| `splunk-sre-context.md` | **The brain.** SPL methodology, SRE query library, dashboard design, Dashboard Studio JSON, guardrails. | both |
| `cds-splunk-data-dictionary.md` | **The lever.** Real indexes/sourcetypes/fields/services/SLOs. You fill this in. | both |
| `.roomodes` | Roo Code custom mode shell (`🔭 Splunk SRE (CDS)`). | Roo |
| `workflows/generate-spl.md` | `/generate-spl` slash command. | Cline |
| `workflows/build-dashboard.md` | `/build-dashboard` slash command (with requirements intake). | Cline |

## One brain, two tools — install
The knowledge lives once (`splunk-sre-context.md` + `cds-splunk-data-dictionary.md`). Place copies
where each tool reads them:

**Roo Code**
```
<repo>/.roomodes
<repo>/.roo/rules-splunk-sre/01-splunk-sre-context.md      # copy of splunk-sre-context.md
<repo>/.roo/rules-splunk-sre/02-cds-data-dictionary.md     # copy of cds-splunk-data-dictionary.md
```
Then pick the **🔭 Splunk SRE (CDS)** mode in the Roo mode selector.

**Cline**
```
<repo>/.clinerules/splunk-sre-context.md                   # copy of splunk-sre-context.md
<repo>/.clinerules/cds-data-dictionary.md                  # copy of cds-splunk-data-dictionary.md
<repo>/.clinerules/workflows/generate-spl.md
<repo>/.clinerules/workflows/build-dashboard.md
```
Cline loads `.clinerules/*.md` always-on; run workflows with `/generate-spl` and `/build-dashboard`.

> Keep ONE canonical copy (suggest `splunk-sre-context.md` at repo root) and a tiny sync script
> or pre-commit hook that copies it into both locations. Editing two copies by hand is how the
> SPL methodology drifts. To avoid copies entirely you can instead put the brain in `AGENTS.md`
> at repo root — both Roo and Cline read it — and keep `.roomodes` / workflows as thin shells.

## Optional: wire the Splunk MCP (turns generate-only into validated)
Official Splunk MCP Server (Splunkbase app 7931, GA — RBAC-aware, `splunk_`-namespaced, tools-only,
guardrails against destructive ops). Example client config (token injected via the `mcp-remote`
proxy):
```json
{
  "mcpServers": {
    "splunk": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://<SPLUNK_HOST>:8089/services/mcp",
               "--header", "Authorization: Bearer <READ_ONLY_TOKEN>"]
    }
  }
}
```
Use a **read-only** role for `<READ_ONLY_TOKEN>`. With this wired, the mode validates SPL against
a **dev/sandbox** Splunk first and reports row counts/runtime. Without it, the mode runs
generate-only and labels everything `generate-only, NOT run`.

## Two operating tiers
- **Generate-only** (no MCP): produces SPL + dashboard JSON, never claims it ran. Fully useful
  on its own — this is the safe default and works even if Splunk is unreachable from your VDI.
- **Validated** (MCP wired, reachable, read-only): same output, plus dry-run validation against
  sandbox Splunk before you ever touch prod.

## JPMC landmines — confirm before relying on the validated tier
1. **MCP reachability from VDI.** The MCP talks to Splunk's mgmt port `:8089`. Whether your
   Gaia/VDI network permits the agent host to reach it is **unconfirmed** and may be blocked.
   If blocked → stay in generate-only. (Tracked in the dictionary §7.)
2. **Auth model.** Your Splunk is SID-based from VDI; the MCP expects OAuth 2.1 / bearer token.
   How those map is a **Splunk-admin conversation**, not assumed solved here.
3. **Knowledge objects are change-managed.** Dashboards/alerts go to prod via dashboard-as-code
   (export JSON → git → PR → CI), never via the agent. The kit enforces this.
4. **Shared-indexer cost = blast radius.** Guardrails (no `index=*`, mandatory scope/time, base+
   chain, cost ceiling, sandbox-first) exist because an expensive agent search on shared Splunk
   is an AWM-wide noisy-neighbor risk.
5. **Data sensitivity.** CDS logs may carry document metadata / PII. The kit never pastes raw
   payloads into output — aggregate or hash.

## Do this first
1. Fill in `cds-splunk-data-dictionary.md` (indexes, sourcetypes, **real field names**, the 12
   services, SLO thresholds). Output quality tracks this directly.
2. Decide generate-only vs validated; if validated, resolve landmines 1–2 with your Splunk admin
   and provision a read-only role.
3. Drop the files into Roo and/or Cline per the install section.
4. Try it: `/generate-spl` → "error-budget burn for cds-find over the last hour", and
   `/build-dashboard` → "Tier-1 health overview for the Pithos cutover".
