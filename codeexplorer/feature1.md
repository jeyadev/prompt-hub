Proposal: “Flow-as-Code” Cadence for Bitbucket

Feature 1 — Check-in updated flow diagrams with every relevant code change


---

1) Objective

Institutionalize Flow-as-Code: whenever a change alters control-flow or request-flow, developers generate and commit updated flow diagrams (and their source) alongside the code. This keeps operational truth in sync with code, enabling SREs to identify checkpoints for monitoring with zero archeology.

This document covers only Feature 1 (check-in cadence). Module-only generation, checkpoint YAML, and Terraform alerts will be separate follow-ups.


---

2) Scope & Definitions

Flow diagram: the canonical representation of request/feature flow. Stored as:

Source: *.mmd (Mermaid) or *.dot (Graphviz) — diffable, reviewable.

Render: *.svg (preferred) or *.png — previewable in Bitbucket/README.

Manifest: optional machine-readable *.flow.json (node/edge list, hashes).


CodeExplorer: your Copilot Custom Mode, exposed via a local CLI:
codeexplorer generate --scope <all|module> --out docs/flows/ ...

Eligible change: any PR that modifies code in directories the flow depends on (e.g., src/, routes/, handlers/, services/, infra/edge).



---

3) Cadence Design (at a glance)

Dev edits code  ──▶ pre-commit hook runs ▶ generates/updates flow source + svg
        │
        └─▶ PR opened ▶ Bitbucket Pipeline regenerates flows ▶ compares ▶
                          ├─ match → add label “flow-verified”
                          └─ drift → fail build with actionable diff

Principles

Generate before commit (fast local feedback) and on PR (authoritative check).

Store source + render in repo; render deterministically (pinned version).

Keep history lean: one canonical file per module + a dated changelog (optional).



---

4) Repository Layout & Naming

docs/
  flows/
    <service>/
      README.md
      global/
        request-flow.mmd
        request-flow.svg
        request-flow.flow.json
      modules/
        <module-name>/
          flow.mmd
          flow.svg
          flow.flow.json
      _changelog/     # optional, for historical snapshots

Conventions

Node IDs stable across edits (hash of file:line or symbol).

Deterministic sort of nodes/edges before write → minimizes noisy diffs.

Render at CI with pinned CodeExplorer + Mermaid/Graphviz versions.



---

5) Developer Workflow (local)

Make target (language-agnostic)

# file: Makefile
FLOW_SCOPE ?= changed   # changed|module:<name>|all
flows:
	./scripts/gen_flows.sh $(FLOW_SCOPE)

Generation script (wrapper)

# file: scripts/gen_flows.sh
set -euo pipefail
SCOPE="${1:-changed}"

# Pin tool versions for determinism
export CODEEXPLORER_VERSION="1.3.0"
export MERMAID_CLI_VERSION="10.9.1"

mkdir -p docs/flows

case "$SCOPE" in
  all)
    codeexplorer generate --scope all --format mmd \
      --out docs/flows --manifest --deterministic
    ;;
  changed)
    # Detect touched modules since last commit/merge-base
    MODS=$(git diff --name-only "$(git merge-base HEAD origin/main)" -- src/ | \
           awk -F'/' '{print $2}' | sort -u)
    for m in $MODS; do
      codeexplorer generate --scope module --module "$m" --format mmd \
        --out "docs/flows/modules/$m" --manifest --deterministic
    done
    ;;
  module:*)
    m="${SCOPE#module:}"
    codeexplorer generate --scope module --module "$m" --format mmd \
      --out "docs/flows/modules/$m" --manifest --deterministic
    ;;
  *)
    echo "Usage: gen_flows.sh [all|changed|module:<name>]" && exit 2
    ;;
esac

# Render to SVG (prefer SVG over PNG for crisp diffs)
find docs/flows -name "*.mmd" -print0 | while IFS= read -r -d '' f; do
  n="${f%.mmd}"
  npx -y @mermaid-js/mermaid-cli@${MERMAID_CLI_VERSION} -i "$f" -o "${n}.svg" \
     --configFile .mermaid.config.json
done

Pre-commit enforcement

Choose one:

A) pre-commit (polyglot)

# file: .pre-commit-config.yaml
repos:
- repo: local
  hooks:
  - id: flow-as-code
    name: Ensure flows are generated for changed modules
    entry: bash -lc './scripts/gen_flows.sh changed && git add docs/flows'
    language: system
    pass_filenames: false

B) Husky (NodeJS)

// package.json
{
  "scripts": { "flows": "make flows" },
  "devDependencies": { "husky": "^9.0.0" }
}

# .husky/pre-commit
#!/usr/bin/env sh
. "$(dirname "$0")/_/husky.sh"
npm run flows
git add docs/flows


---

6) PR Gate (Bitbucket Pipelines)

Goal: reproducibly regenerate flows and fail if committed files are stale.

# file: bitbucket-pipelines.yml
image: node:20-bullseye

pipelines:
  pull-requests:
    '**':
      - step:
          name: Flow-as-Code Verification
          caches: [node]
          script:
            - apt-get update && apt-get install -y graphviz jq
            - npm i -g @mermaid-js/mermaid-cli@10.9.1
            - curl -fsSL https://example.com/codeexplorer/install.sh | bash -s -- 1.3.0
            - ./scripts/gen_flows.sh changed

            # Compare working tree outputs with committed state
            - >
              DIFFS=$(git status --porcelain docs/flows |
                awk '{print $2}' | tr '\n' ' ');
              if [ -n "$DIFFS" ]; then
                echo "❌ Flow diagrams out of date for: $DIFFS";
                echo "Run: make flows && commit the updated files.";
                exit 1;
              fi
          artifacts:
            - docs/flows/**

      - step:
          name: Label PR on Success
          script:
            - echo "Flows verified ✅"
            # Optionally call Bitbucket API to add label "flow-verified"

Notes

Pin exact versions (CodeExplorer, Mermaid) for deterministic renders.

artifacts allow reviewers to download CI-generated flows if we choose not to store renders; here we do store both source + render for reviewer convenience.



---

7) Review Ergonomics

PR checklist (Bitbucket default PR template):

[ ] make flows ran locally.

[ ] Updated docs/flows/modules/<module>/flow.mmd (and .svg).

[ ] Global docs/flows/global/request-flow.mmd updated if routing changed.


Diff sanity:

Keep Mermaid source normalized: stable node ordering, stable edge labels.

Disallow binary-only updates: a commit that changes .svg must also change .mmd.


Ownership: add default reviewers for docs/flows/** (e.g., SRE + TL).



---

8) Exception Policy (low-noise, high-signal)

Auto-exempt PRs: docs-only, comments-only, version bumps without code path change. Mark with no-flow-change label or FLOW:SKIP in commit message.

Guardrail: pipeline verifies exemption is consistent — if generator detects drift, it overrides exemption and fails the build.



---

9) KPIs & Governance

Freshness SLA: 100% of merged PRs that change eligible code have flow-verified.

Lead Time to Truth: median hours from first commit to verified flow ≤ PR median.

Drift Rate: < 2% PRs fail flow gate after policy adoption.

Fitness Function (optional nightly): regenerate all flows on main and assert zero drift; report to Slack.



---

10) Risks & Mitigations

Noisy diffs from nondeterministic render → Mitigation: pin versions, sort nodes/edges, prefer SVG, strip timestamps from output.

Developer friction → Mitigation: pre-commit auto-generation; fast local CLI; module-scoped runs by default.

Repo bloat from binary assets → Mitigation: prefer SVG; prune _changelog; optionally store renders as pipeline artifacts only (if Bitbucket preview suffices).

Partial coverage (flows not capturing new patterns) → Mitigation: add “Flow Impact” PR question; if routing/contracts changed, update global flow too.



---

11) Representations (lightweight)

Sequence of checks in CI

Generate (changed modules) → git status docs/flows → if dirty → fail + hint

PR template snippet

### Flow-as-Code
- [ ] Ran `make flows`
- [ ] Updated module flows:
  - modules/<module-a>/flow.mmd (+svg)
  - modules/<module-b>/flow.mmd (+svg)
- [ ] Global flow updated (if applicable)


---

12) Rollout Plan

1. Week 0: add scaffolding (scripts/, docs/flows/, pipeline step); opt-in on 1–2 services.


2. Week 1–2: make the gate advisory (warn only); gather friction points.


3. Week 3: enforce gate on selected repos; publish quickstart and examples.


4. Week 4: nightly full-regen on main with drift report; add KPIs to SRE dashboard.




---

13) Ops-Ready Summary (checklist)

Repo contains canonical docs/flows/** with source .mmd and .svg.

Local pre-commit auto-generates flows for changed modules.

Bitbucket Pipeline regenerates and fails on drift; adds “flow-verified” label on success.

Exceptions exist but are validated; no silent drift.

Versions pinned; diffs are clean; ownership for flow files is explicit.



---

Next, we’ll extend this cadence to module-only generation UX (quick, single-module updates) and the checkpoint YAML that powers alert Terraform.

