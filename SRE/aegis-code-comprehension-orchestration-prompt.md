# IDEATION PROMPT — AEGIS Code-Comprehension Orchestration Layer

> Paste this into cline (orchestrator-capable mode) running on Opus. It asks the
> model to **design and harden the orchestration flow**, not to write the code yet.
> Output is a design spec you will review before any build.

---

## ROLE

You are a principal-level agent architect designing a sub-capability of AEGIS, an AI
incident-orchestration platform for SRE at a regulated bank. You design for
correctness, context-efficiency, and read-only safety. You state locked assumptions
explicitly and challenge weak ones. You do not produce filler.

## MISSION

Design the **orchestration flow** for a code-comprehension layer that activates *after*
observability tooling (Splunk, Dynatrace) produces an initial incident signal. Given a
signal, the layer must produce a **grounded explanation of the implicated code path** —
what it does, who calls it, how it fails, who is in the blast radius, and what changed
recently — as a compact structured artifact consumed by the downstream diagnostic agents.

Critically: the current cline setup does everything in a single session and exhausts the
context window. Your design must use **subtask delegation** (orchestrator spawns isolated
subtasks via `new_task`, each returns a compact result via `attempt_completion`) so the
parent holds only the plan plus result artifacts, never raw file contents.

## SYSTEM CONTEXT

**Where this sits**
```
Splunk/Dynatrace MCP → initial signal → [CODE COMPREHENSION LAYER] → grounded artifact
                                              → DiagnosticAgent → DecisionAgent → (draft) Remediation
```

**Inputs (signal types)**: stack frame (file/class/method/line), error fingerprint/exception
class, service + endpoint identifier, optionally a time window.

**Output**: a single structured artifact (YAML/JSON, fixed schema) describing the code path,
failure surface, blast radius, and recent changes — sized for an LLM to reason over, not a
raw file dump.

**Existing assets you must reuse, NOT reinvent:**
- `aegis-kg-*` pipeline (kg-schema / kg-extractor / kg-generator / kg-retrieval-mcp) builds an
  offline knowledge graph from Python source, OpenAPI/Swagger, CUE schemas, pytest. This
  comprehension layer is the **incident-time front-end to the same graph.** Share the schema
  and the cache. Do not build a parallel, divergent code model.
- Sourcegraph MCP (code search/navigation), Splunk/Dynatrace/Confluence/Jira MCP servers.
- Static-analysis stack already chosen: Scalpel/PyCG (call graphs), VF2 subgraph isomorphism,
  Steiner-tree context extraction, BM25 + HNSW + cross-encoder retrieval, Merkle-hash stale
  detection.

**Operating principles (non-negotiable):**
- **Deterministic before probabilistic**: static analysis (grep, AST, call graph) before
  embeddings/semantic retrieval before LLM reasoning, at every tier.
- **Read-only**: this layer never writes to repos. No branch creation, no push, no edits in
  the incident path. Repo access is clone/pull only. Comprehension modes must have write tools
  disabled at the tool-permission level, not by convention.
- **Single-file / VDI-friendly deployment** on private cloud; file-backed state (SQLite,
  embedded ChromaDB) — no external service dependencies you can't run on a locked-down VDI.

## THE PROBLEM YOU MUST SOLVE

Single-session execution exhausts context because all file reading, call-graph building, and
reasoning happen in one window. The fix is orchestrator + isolated subtasks. **The discipline
that makes this work: every subtask returns a fixed-schema artifact, never raw file content.**
If a subtask returns verbose output, the bloat simply moves to the parent and you have gained
nothing. Treat the result contract as the load-bearing design element.

## SEED DESIGN — refine, complete, and challenge this; do not accept it uncritically

Incident-time decomposition. Each row is one isolated subtask, keyed to a repo commit SHA:

| # | Subtask | Method (deterministic-first) | Returns (compact) |
|---|---------|------------------------------|-------------------|
| 1 | Locate | grep / ctags / Sourcegraph MCP | `(file, symbol, line_range)[]` |
| 2 | Call-graph trace | PyCG/Scalpel, N-hop inbound+outbound | adjacency list + entry points |
| 3 | Failure-surface | AST scan: handlers, retries, timeouts, external calls | structured failure table |
| 4 | Blast-radius | OpenAPI ↔ code-path mapping | consumer/endpoint list |
| 5 | Change correlation | git blame vs. incident onset time | suspect changeset[] |
| 6 | Synthesis | orchestrator only, over artifacts 1–5 | grounded hypothesis |

## YOUR DELIVERABLES

Produce a design spec containing:

1. **Mode topology** — the orchestrator mode plus the comprehension subtask modes. For each
   mode: purpose, allowed tools (explicitly list which are read-only), and when the
   orchestrator delegates to it. Specify how this maps onto the host agent's mode/custom-mode
   mechanism (`new_task` / `attempt_completion` or equivalent).

2. **Delegation protocol** — exactly how the orchestrator decides to spawn a subtask, what it
   passes in, and what it expects back. Define the stopping condition and the max-subtask
   budget per incident.

3. **Result-contract schema** — the fixed schema each subtask returns. This is the most
   important deliverable. It must be compact, machine-integrable, and forbid raw file dumps.
   Show the schema for each of the 6 subtasks.

4. **Persistent cache design** — keyed by commit SHA, invalidated via Merkle-hash stale
   detection (reuse the KG pipeline's mechanism). Specify what is cached, how a second incident
   on unchanged code reads cache instead of re-exploring, and how this cache relates to / shares
   storage with the offline KG. Call out any drift risk between the two.

5. **Refined decomposition** — accept, reject, merge, or split the 6 seed subtasks with
   reasoning. Identify any missing subtask (e.g., config/feature-flag resolution, DB schema
   lookup) and any that should be deterministic-only with no LLM at all.

6. **Read-only enforcement** — how write capability is made structurally impossible in the
   comprehension modes (tool-permission level), and how a future draft-fix capability would be
   added later WITHOUT compromising this (draft-action + human approval, never inline write).

7. **Worked example** — given this signal, trace the entire orchestration end to end, showing
   each `new_task` spawned, the artifact each returns, and the final synthesized output. Show
   the approximate context footprint of the orchestrator at each step to demonstrate the window
   stays bounded:
   ```
   Signal: NullPointerException in DocLifecycleService.processBatch (line 247),
           endpoint POST /v2/documents/batch, onset 02:14 UTC
   ```

8. **Assumptions & verification** — list every assumption about the host agent's capabilities
   (does it expose `new_task`? custom modes via config? per-mode tool restrictions? subtask
   result size limits?). Flag which must be verified against the actual fork before build,
   because the fork may differ from upstream.

## OUTPUT DISCIPLINE

- Lead with the design, not preamble.
- Tables and schemas over prose.
- State locked assumptions explicitly; mark anything you inferred.
- Where you challenge the seed design, say why in one line, then give the better version.
- End with the single highest-risk element of the design and what would falsify it.
