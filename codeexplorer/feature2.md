Proposal: Module-Only “Flow-as-Code”

Feature 2 — Generate and review flow diagrams per module (conceptual design)


---

1) Objective

Enable developers to generate, review, and check in flow diagrams for a single module without touching the whole service’s request-flow. This makes reviews scoped, fast, and accurate—so SREs can reason about localized changes and update monitors only where the blast radius actually lives.

Outcome: Every PR that changes module code also ships an updated module flow artifact (source + render + manifest) for just that module. CI verifies freshness and labels the PR accordingly.


---

2) Scope & Definitions

Module: the smallest, independently reviewable unit of code (e.g., Gradle/Maven module, npm workspace, Go package, or a repo-local “module root” folder).

Module Flow: nodes and edges within a module’s execution path—from public entrypoints (handlers, jobs, consumers) to internal collaborators—plus declared external touchpoints (calls into other modules/services).

Artifacts (conceptual, not implementation):

flow.mmd (Mermaid source; diffable)

flow.svg (render; previewable)

flow.flow.json (machine-readable manifest: nodes, edges, external touchpoints, hashes)




---

3) User Stories (Concept)

As a developer, I run module-only generation and get a fresh diagram for the module I touched, in <30s, with stable node IDs so Git diffs are tiny and meaningful.

As a reviewer, I see just the module’s execution graph that changed, plus a short “what changed” summary.

As an SRE, I can extract the module’s external touchpoints to validate whether downstream monitors need updates.



---

4) UX & Cadence (Concept)

Local (Dev)

# Choose a module or let detection figure it out
codeexplorer flow module <module-name>
# or
codeexplorer flow changed --granularity=module

Default granularity is module.

Output lands under docs/flows/modules/<module>/.


Pull Request (Bitbucket)

Detect impacted modules → regenerate module flows → compare with committed artifacts
   ├─ no drift → add label "flow-module-verified:<module>"
   └─ drift    → fail with guidance "regenerate module flows and commit"

Representations (ASCII)

Module generation sequence

Developer → CodeExplorer (module scope) → Flow Artifacts (mmd/svg/json)
                                 ↑
                 Module Boundary (manifest or heuristics)

PR verification

PR → Pipeline → Identify impacted modules → Re-generate module flows
                                 │
                                 ├─ Matches committed → pass + label
                                 └─ Drift detected    → fail + diff summary


---

5) Defining Module Boundaries (Conceptual Strategies)

1. Explicit Manifest (preferred for clarity)
Place a module.flow.toml (or yaml) at each module root that declares:



entries: public entrypoints (HTTP handlers, cron jobs, consumers)

include / exclude patterns for files

external_touchpoints: known calls to other modules/services (names only)

ownership: CODEOWNERS-like spec (for auto reviewers)


2. Heuristic Fallbacks (when no manifest)



Build system boundaries (Maven <module>, npm workspaces, Go modules)

Directory conventions (e.g., src/<module>/**)

Entrypoint discovery by annotations/conventions (@Controller, handle(), etc.)


3. Cross-Module Awareness
Module flows show stubs for edges crossing boundaries (e.g., “→ CatalogClient.getSku”), but do not expand into foreign modules. This keeps the diagram scoped while preserving context.




---

6) Generation Semantics (Concept)

Inputs: module boundary + AST/bytecode scan + call graph + annotations.

Outputs:

Mermaid source with stable ordering and IDs

SVG render for preview

Manifest JSON with:

nodes[]: { id, kind (handler|db|cache|http|queue|fn), symbol, file, line, hash }

edges[]: { from, to, label, kind (call|async|db|network) }

external_touchpoints[]: { name, type (http|rpc|queue), purpose }

generation: { module, tool_version, schema_version, semantic_hash }



Determinism: sort nodes by (kind, symbol); hash IDs from (symbol_path, signature); strip timestamps so diffs are signal-only.



---

7) Repository Structure (Concept)

docs/
  flows/
    modules/
      <module-name>/
        flow.mmd
        flow.svg
        flow.flow.json
    global/                   # untouched by module-only runs
      request-flow.mmd
module.flow.toml              # at each module root (optional but preferred)


---

8) Freshness Policy & Review Ergonomics

Policy: A PR that modifies files matching the module’s boundary must update that module’s flow artifacts.

Pass Condition: CI re-generation yields zero diff vs committed artifacts.

Review Aid: CI posts a “flow delta summary” (node/edge adds/removes/label changes) for the module.

Labeling: On success, add flow-module-verified:<module> to PR.

Exception: FLOW:SKIP label allowed for comment-only or docs-only changes; CI cross-checks and revokes skip if it detects semantic drift.



---

9) Performance & Scale Considerations (Concept)

Latency Budget: p50 generation < 20s, p95 < 60s per module on CI workers.

Caching: Key cache by (module semantic hash, tool version) to avoid recomputation.

Sharding: For multi-module PRs, generation tasks run in parallel.

Artifacts: Keep both source (.mmd) and render (.svg) in Git for reviewer ergonomics; manifests (.flow.json) unlock downstream automation later.



---

10) Quality Bar (Concept)

Stability: Node IDs must persist across trivial refactors (rename file path only).

Completeness: All public entrypoints in module.flow.toml must appear as roots in the diagram.

Clarity: External edges must be labeled with purpose (“inventory lookup”, “payment auth”).

Noise Control: Collapse uninteresting fan-out (config readers, DI containers) into a single “framework” node.



---

11) Governance & Metrics (Concept)

PR Template Section

[ ] Module(s) impacted: <name>

[ ] Ran module flow generation locally

[ ] Flow artifacts updated and reviewed


Metrics to track

flow_generation_duration_seconds{module} (p50/p95)

flow_drift_detected_total{module}

flow_skip_exemptions_total (should trend to zero)




---

12) Risks & Mitigations (Concept)

Hidden Cross-Module Effects: A local change alters global routing.

Mitigation: If entrypoints/routing signatures change, CI suggests running full global flow as a secondary check.


Diagram Flap (nondeterministic diffs):

Mitigation: deterministic sort + pinned tool versions + ID hashing scheme.


Developer Friction:

Mitigation: default to changed-modules only; pre-commit helper that runs in seconds; clear CI guidance on failures.




---

13) Validation & Simulation (Concept)

Dry-Run Delta: codeexplorer flow module <m> --dry-run --diff prints node/edge deltas vs main.

What-Changed Map: For a PR, list why the module needs regen (files matching boundary and their inferred impact).

Nightly Audit: Rebuild all module flows on main; alert on drift (guards against manual edits to artifacts).



---

14) Ops-Ready Summary (Checklist)

Module boundaries declared (prefer module.flow.toml; heuristics as fallback).

Module-only generation produces: flow.mmd, flow.svg, flow.flow.json.

CI verifies freshness per impacted module; applies flow-module-verified:<module>.

Deterministic outputs minimize diff noise; external touchpoints labeled.

Exceptions exist but are validated; nightly audit prevents silent drift.



---

Next logical piece: define the Checkpoint YAML schema that each module can emit from its flow.flow.json (nodes → “checkpoints” + monitoring intents). That will feed the Terraform alerts module you mentioned and closes the loop from code → flow → monitor.

