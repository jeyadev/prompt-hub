# AEGIS KG — GitHub Copilot Vibe Coding Guide

**Companion to:** `AEGIS-KG-MVP-SPEC.md`
**Purpose:** ordered playbook for driving Copilot through the 7-week MVP build. Paste sections verbatim; don't paste the whole file at once.
**Version:** 0.1.0
**Last updated:** April 2026

---

## How to use this file

This guide is **stateful** — each section builds on the previous one's output. Before starting a new section:

1. Read the section's **Preconditions** — verify they're met
2. Read the section's **Red Flags** — know what to catch in Copilot output
3. Load the relevant section of `AEGIS-KG-MVP-SPEC.md` into Copilot context (not the whole spec)
4. Paste prompts in order

Do not skip sections. Do not paste multiple sections at once. Copilot degrades when context windows saturate; short focused sessions produce better code.

**Rule of thumb**: one Copilot chat per section of this guide. Close and start fresh between sections.

---

## Session 0: Repo setup (day 1, ~2 hours)

### Preconditions
- GitHub org access for creating four repos
- Python 3.11+ locally; `uv` or `poetry` picked (assume `uv` below)
- VS Code + GitHub Copilot enabled; Claude Opus/Sonnet access confirmed
- Pilot service selected (per spec §15 Q1)

### Red flags
- Copilot wanting to create a monorepo — reject; spec §ADR-001 locks four repos
- Copilot skipping pyproject in favor of setup.py — reject; modern Python tooling only
- Copilot proposing `flask` or `django` as a framework-adapter starting point — MVP ships FastAPI only

### Commands (run manually, not Copilot)

```bash
# Create four repos via gh CLI
for repo in aegis-kg-schema aegis-kg-extractor aegis-kg-generator aegis-kg-retrieval-mcp; do
  gh repo create "$YOUR_ORG/$repo" --private --clone
  cd "$repo"
  # scaffold
  uv init --package --name "${repo//-/_}"
  git add -A && git commit -m "chore: initial scaffold"
  git push -u origin main
  cd ..
done
```

### Copilot prompt — scaffold each repo's pyproject

**Paste into Copilot chat (once per repo)**:

```
I'm scaffolding a Python package called `<REPO-NAME>`, part of the AEGIS knowledge-graph tool for SRE runbook generation. 

Target Python 3.11+. Use pydantic >=2.6, pyyaml >=6, pytest for tests. Add ruff + mypy config.

Update pyproject.toml to declare:
- package name: <python-module-name>
- dependency on aegis-kg-schema with version range ">=0.1,<1.0" (for the three non-schema repos)
- dev dependencies: pytest, pytest-asyncio, ruff, mypy
- console script entry point: <script-name>

Do NOT add runtime dependencies beyond what the AEGIS-KG-MVP-SPEC.md §17 dependency table specifies for this repo.

Show me the diff only.
```

For each repo, swap names appropriately:
- `aegis-kg-schema` → module `aegis_kg_schema`, no deps on siblings
- `aegis-kg-extractor` → module `aegis_kg_extractor`, script `aegis-kg-extract`
- `aegis-kg-generator` → module `aegis_kg_generator`, script `aegis-kg-generate`
- `aegis-kg-retrieval-mcp` → module `aegis_kg_retrieval_mcp`, script `aegis-kg-mcp`

### Verification

```bash
cd aegis-kg-schema && uv sync && uv run pytest
# should pass (zero tests is fine; verify no errors)
```

Commit: `chore: scaffold pyproject with deps`

---

## Session 1: Schema repo — CUE schema + Pydantic models (week 1, ~8 hours)

### Preconditions
- Session 0 complete
- `cue` binary installed locally (https://cuelang.org/docs/install/)

### Red flags
- Copilot generating Pydantic models by hand instead of from CUE — we want CUE as source of truth
- Copilot inventing fields not in spec §4 — reject; §4 is the contract
- Copilot skipping `annotations` block — reject; provenance is mandatory per §10.1

### Step 1.1: Define CUE schema

Load into Copilot context: **Spec §4 (entire section)**.

**Prompt**:

```
Create a CUE schema file at `schemas/v1alpha1.cue` that defines the node kinds from AEGIS-KG-MVP-SPEC.md §4.2.

Rules:
1. Define a common envelope `#Envelope` with apiVersion, kind, metadata, spec, status
2. `#Metadata` must include: name, namespace, schemaVersion (int, default 1), labels, annotations
3. annotations MUST require source.extractor, source.commit_sha, source.path
4. Each node kind (Service, API, Endpoint, FailureMode, TestInvariant, RemediationAction, ExternalCall, Function) unifies with #Envelope and adds its own spec shape
5. Use disjunctions for enum-like fields (protocol: "http" | "grpc" | "async")
6. Relations use a common #Relation type with type, target, confidence (0.0-1.0), source

Match the YAML examples in the spec exactly. Don't invent fields.

Output only the CUE file content.
```

**Verify**:

```bash
cue vet schemas/v1alpha1.cue
# for each example YAML in the spec:
echo '<yaml>' | cue vet schemas/v1alpha1.cue -
```

### Step 1.2: Generate Pydantic models from CUE

**Prompt**:

```
CUE doesn't have a great Python-binding story, so we'll use CUE for validation at the file-layer and hand-author Pydantic models that mirror the CUE shape.

Create `src/aegis_kg_schema/models.py` with Pydantic v2 models for every node kind in schemas/v1alpha1.cue:
- BaseModel envelope with model_config = ConfigDict(extra="forbid")
- Nested models for metadata, spec, relations
- Enum classes for protocols, action_types, invariant_kinds
- Every model has a `to_canonical_yaml()` method that uses JCS-equivalent rules (spec §4.5)

Create `src/aegis_kg_schema/canonicalize.py` with:
- `canonicalize_json(obj: dict) -> str` using rfc8785
- `canonicalize_yaml(obj: dict) -> str` with sorted keys, stripped nulls, LF line endings

Create `src/aegis_kg_schema/validate.py` with:
- `validate_file(path: Path) -> ValidationResult` that:
  1. Parses YAML
  2. Validates against Pydantic model for declared kind
  3. Runs cue vet as subprocess
  4. Returns errors with file path + line number

Write pytest tests that round-trip every example YAML from spec §4.2.
```

### Step 1.3: Schema migrations skeleton

**Prompt**:

```
Create `migrations/__init__.py` and `migrations/0001_init.py` following Alembic's pattern (without actually using Alembic).

Each migration exports:
- `version: str` (e.g., "0001")
- `down_revision: Optional[str]`
- `upgrade(graph_dir: Path) -> None`
- `downgrade(graph_dir: Path) -> None`

Create CLI `aegis-kg-schema migrate --check` that:
1. Reads graph_dir/schema-version
2. Finds all migrations/ files
3. Reports: current version, latest available, pending migrations
4. With --apply, runs pending upgrade() functions in order
5. Writes new version to graph_dir/schema-version

0001_init.py has empty upgrade/downgrade — it's the baseline.
```

### Step 1.4: Publish schema to internal index

**Paste for context**: your JPMC internal PyPI guidance (not in this guide — check with platform team).

Commit: `feat: schema v1alpha1 with pydantic + cue validation`

### Week 1 exit check

```bash
cd aegis-kg-schema
uv run pytest                                    # all green
uv run aegis-kg-schema validate examples/*.yaml   # all green
uv run aegis-kg-schema migrate --check            # reports "baseline, no pending"
```

---

## Session 2: Extractor skeleton + discovery (week 1-2, ~6 hours)

### Preconditions
- Session 1 complete; schema repo installable
- Pilot service repo cloned to `examples/pilot/`

### Red flags
- Copilot hardcoding paths like `app/main.py` — must walk repo generically
- Copilot importing FastAPI in the core (not the adapter) — reject, breaks agnosticism
- Copilot adding parsing logic before the discovery stage ships — stage gating matters

### Step 2.1: Framework adapter interface

Load into Copilot context: **Spec §5.2**.

**Prompt**:

```
In aegis-kg-extractor, create `src/aegis_kg_extractor/adapters/base.py` with the FrameworkAdapter ABC per spec §5.2.

Create typed seed classes:
- EndpointSeed: method, route, handler_fqn, operation_id (optional), source_file, source_line
- MiddlewareSeed: name, applies_to (glob or FQN pattern), source_file, source_line
- ErrorHandlerSeed: exception_class, handler_fqn, source_file, source_line

Create `src/aegis_kg_extractor/adapters/registry.py` with:
- `register(adapter_cls)` decorator
- `detect_adapter(repo_root: Path) -> Optional[FrameworkAdapter]` that tries each registered adapter's .detect()
- Built-in registration order: FastAPIAdapter, GenericPythonAdapter (fallback)

Create a stub `src/aegis_kg_extractor/adapters/generic_python.py` that returns empty iterators for everything — it's the no-framework fallback.

Don't implement FastAPIAdapter yet. Just the interface and stub.
```

### Step 2.2: Discovery CLI

**Prompt**:

```
Create `src/aegis_kg_extractor/cli.py` with a Typer-based CLI.

First command: `aegis-kg-extract discover --repo <path>`.

Discovery output (JSON to stdout):
{
  "repo_root": "...",
  "detected_framework": "fastapi" | "generic_python",
  "python_files": [...],
  "has_openapi": bool,
  "openapi_path": "..." or null,
  "has_cue": bool,
  "cue_files": [...],
  "has_pytest": bool,
  "test_files": [...],
  "pyproject_path": "..." or null
}

Implementation:
- Walk repo with pathlib; respect .gitignore (use `pathspec` package)
- Detect pyproject.toml, openapi.yaml/.json, *.cue, test_*.py and *_test.py
- Call registry.detect_adapter() for framework detection
- Print JSON to stdout, log to stderr

Write tests using tmp_path fixtures that create fake repo structures.
```

### Step 2.3: FastAPI adapter

Load into Copilot context: **Spec §5.2, §5.5** (just these two subsections).

**Prompt**:

```
Create `src/aegis_kg_extractor/adapters/fastapi_adapter.py` implementing FrameworkAdapter for FastAPI.

detect(): returns True if any file imports `fastapi` or has `FastAPI()` or `APIRouter()` instantiation. Use AST, not regex.

extract_endpoints():
- Walk all Python files with `ast.parse`
- For each function/method, check decorators
- Match @app.get, @app.post, @router.get, etc. where app/router is FastAPI/APIRouter instance
- For each match, extract:
  - method (lowercase from decorator name)
  - route (first positional arg string literal; None if not literal)
  - handler_fqn (module.function or module.Class.method)
  - operation_id (kwarg if present)
  - source_file, source_line

Handle edge cases:
- @router.api_route(methods=["GET", "POST"], ...) → emit one EndpointSeed per method
- Route is f-string or computed → yield with route=None and mark
- Decorator is in typing chain (@app.get("/foo", response_model=...)) — parse full call

DON'T try to resolve variables across modules — if `router = APIRouter()` is in module A and used via import in module B, that's fine; we match on the usage site.

Tests: build fixture files in tests/fixtures/fastapi/ covering each edge case.
```

### Step 2.4: Verification on pilot

```bash
cd aegis-kg-extractor
uv run aegis-kg-extract discover --repo ../examples/pilot | jq .

# Expected: detected_framework matches pilot, endpoints list is reasonable
```

Commit: `feat: discovery CLI + FastAPI adapter`

### Week 1 final exit check

```bash
uv run pytest                                             # all green across all 4 repos
uv run aegis-kg-extract discover --repo examples/pilot   # produces valid JSON
```

---

## Session 3: Static AST + call graph (week 2, ~8 hours)

### Preconditions
- Session 2 complete

### Red flags
- Copilot wanting to walk AST without caching — caching is required per spec §10.4
- Copilot ignoring decorators in call-graph construction — FastAPI uses them heavily
- Copilot writing synchronous Scalpel calls without timeout — Scalpel can hang on weird syntax

### Step 3.1: AST extractor for module/function nodes

Load into Copilot context: **Spec §4.2 (Function, Module), §5.1 stages 2-3**.

**Prompt**:

```
Create `src/aegis_kg_extractor/stages/static_ast.py`.

Primary class: `StaticASTExtractor`.

Method: `extract(python_files: list[Path]) -> Iterator[Node]`

For each file:
1. Parse with `ast.parse`; catch SyntaxError and log, skip file
2. Walk module to find:
   - Module-level imports → ImportEdge candidates
   - Top-level functions (ast.FunctionDef, ast.AsyncFunctionDef) → Function nodes
   - Classes → extract methods as Functions with FQN "module.Class.method"
   - Decorators on functions → store as metadata
3. Emit Function node for each with:
   - name (fqn)
   - source.file_hash (sha256 of file content)
   - source.symbol_hash (sha256 of the function's AST dump, line-number-free)
   - source.path, source.line
   - decorators list
   - async flag

Caching: check `.aegis-kg-cache/ast_trees/<file_hash>.pkl` before parsing. Write cache after successful parse.

Don't extract function bodies' call sites yet — that's stage 3.

Tests: fixtures with nested classes, async functions, decorators, class methods.
```

### Step 3.2: Call graph via Scalpel

**Prompt**:

```
Create `src/aegis_kg_extractor/stages/call_graph.py`.

Use Scalpel's call-graph module. Wrap with timeout:

def build_call_graph(repo_root: Path, timeout_sec: int = 300) -> dict[str, list[str]]:
    """Returns dict mapping caller_fqn -> list of callee_fqn."""

Implementation:
1. Run Scalpel in subprocess with timeout (not in-process — it can hang)
2. Parse Scalpel's output into the dict shape
3. On timeout, log warning and return empty dict — graceful degradation
4. Cache result at `.aegis-kg-cache/call_graph/<git_commit_sha>.json`

Emit CALLS edges:
- For each (caller, callee) pair, emit edge with source="static", confidence=1.0 for intra-repo, 0.6 for external (unresolved)
- Dedup edges

Don't emit edges for stdlib calls (filter `sys.`, `os.`, `builtins.`, etc.) — too noisy.

Tests: small fixture with 3 functions calling each other; assert edge set.
```

### Step 3.3: External call detector

Load into Copilot context: **Spec §5.3 (entire subsection)**.

**Prompt**:

```
Create `src/aegis_kg_extractor/stages/external_calls.py`.

Class: `ExternalCallDetector(ast.NodeVisitor)`.

Match FQN patterns from spec §5.3 table. For each match:
1. Identify the enclosing function (FQN)
2. Extract URL/target hint using resolution rules from spec §5.3 "Target hint resolution"
3. Emit ExternalCall node + CALLED_BY edge

Resolver logic:
- Literal string → confidence=high
- Name bound to single assignment of a literal string in same module → confidence=medium, resolve via walk
- Anything else → confidence=low, target_hint="<unresolved>"

Don't call LLM here — MVP leaves stage 9 disabled for external call resolution.

Tests: fixtures covering each library in the match table, each confidence level.
```

### Step 3.4: Emission to disk

**Prompt**:

```
Create `src/aegis_kg_extractor/emit.py`.

Class: `GraphEmitter`.

Methods:
- `add_node(node: BaseNode)`: accumulate in-memory, de-dup by (kind, namespace, name)
- `add_edge(src_id: str, edge_type: str, dst_id: str, confidence: float, source: str)`: accumulate, dedup by tuple
- `emit(output_dir: Path)`:
  1. Resolve edges into `spec.relations[]` on source nodes
  2. Canonicalize each node to YAML via aegis_kg_schema.canonicalize.canonicalize_yaml
  3. Write to output_dir/{kind_plural}/{namespace}/{name_slug}.yaml
  4. Two-phase write: stage to output_dir/.tmp/, compare hash-per-file, only rename changed files
  5. Write derived/ JSONL for call_graph (one edge per line)

Filename slug rules per spec §4.4 (slash → __, path params preserved with braces).

Never write files with line count unchanged but content reordered — verify JCS canonicalization catches this.

Tests: emit a small in-memory graph, assert file layout + content hashes.
```

### Step 3.5: Wire into `aegis-kg-extract` CLI

**Prompt**:

```
Add to cli.py:

Command: `aegis-kg-extract run --repo <path> --out <graph-dir>`

Pipeline (for MVP):
1. discover() — get framework, files
2. StaticASTExtractor.extract(python_files)
3. build_call_graph(repo_root)
4. ExternalCallDetector on each file
5. FastAPIAdapter.extract_endpoints(repo_root) if detected
6. GraphEmitter emits to out dir

Add --verbose, --dry-run flags.

Write a structured log to .aegis-kg-logs/<run_id>.jsonl per spec §10.3.

Exit code 0 on success, 1 on fatal error, 2 on partial success with warnings.
```

### Week 2 exit check

```bash
uv run aegis-kg-extract run --repo examples/pilot --out /tmp/graph-week2
ls -R /tmp/graph-week2/
# Should see services/, apis/, endpoints/, functions/, external_calls/, derived/

# Validate every emitted file
find /tmp/graph-week2 -name '*.yaml' | xargs -n1 uv run aegis-kg-schema validate
# Should be all green
```

Commit: `feat: static AST + call graph + external call extraction`

---

## Session 4: Generator skeleton with stub inputs (week 3, ~8 hours)

### Preconditions
- Session 3 complete
- Hand-written fixture graph at `examples/hand-written-graph/` with at least one FailureMode + RemediationAction + TestInvariant

### Red flags
- Copilot trying to generate prose via LLM this week — defer to Session 6
- Copilot using LangChain/LlamaIndex — we're staying pure Python per your asyncio pattern
- Copilot inventing graph-traversal helpers when networkx has them built in

### Step 4.1: Graph loader

Load into Copilot context: **Spec §6.1 stage 1, §4.4**.

**Prompt**:

```
In aegis-kg-generator, create `src/aegis_kg_generator/graph_loader.py`.

Function: `load_graph(graph_dir: Path) -> nx.DiGraph`

Implementation:
1. Walk graph_dir recursively
2. For each .yaml file, load and validate against schema (aegis_kg_schema.validate_file)
3. On validation error: log, skip, continue
4. Load .jsonl files from derived/ as edges
5. Build networkx.DiGraph:
   - Each node: id = f"{kind}:{namespace}/{name}", attrs = full parsed YAML
   - Each edge from spec.relations[] or derived/ JSONL
6. Return graph

Helper: `node_id(kind: str, namespace: str, name: str) -> str`.

Tests: load fixture graph, assert expected node/edge counts.
```

### Step 4.2: Worthiness scoring

Load into Copilot context: **Spec §6.2**.

**Prompt**:

```
Create `src/aegis_kg_generator/worthiness.py`.

Functions:
- `has_remediation(failure_mode_id, graph) -> bool`: check for outgoing HAS_REMEDIATION edge
- `blast_radius_score(failure_mode_id, graph, max_hops=3) -> float`:
  1. Find all services reachable via DEPENDS_ON from the failure mode's service
  2. Normalize: 0 services = 0.0, 10+ services = 1.0, linear between
- `test_coverage_score(failure_mode_id, graph) -> float`:
  - count incoming ASSERTS edges from TestInvariant nodes
  - 0 tests = 0.0, 3+ tests = 1.0
- `runbook_worthiness(failure_mode_id, graph) -> float`: composite per spec §6.2 formula

Tests with fixture graphs covering hard-gate (no remediation = 0), low coverage, high coverage.
```

### Step 4.3: VF2 archetype matcher

Load into Copilot context: **Spec §6.3**.

**Prompt**:

```
Create `src/aegis_kg_generator/archetypes/__init__.py` and `archetypes/downstream_dependency_failure.py`.

__init__.py exports:
- `ARCHETYPE_REGISTRY: dict[str, Archetype]`
- `classify(failure_mode_id, graph) -> Optional[str]`

Archetype class:
- name: str
- pattern: nx.DiGraph
- node_match: callable
- edge_match: callable

downstream_dependency_failure.py:
- Define the pattern per spec §6.3
- Pattern requires: endpoint can_raise FailureMode, FailureMode caused_by ExternalCall
- Register via @register decorator

classify() uses nx.isomorphism.DiGraphMatcher for each archetype, returns first match or None.

Tests: build synthetic subgraphs that match / don't match.
```

### Step 4.4: Steiner tree context extractor

Load into Copilot context: **Spec §6.4**.

**Prompt**:

```
Create `src/aegis_kg_generator/context.py`.

Function: `extract_context(failure_mode_id, graph, token_budget=2000) -> ContextSubgraph`

Steps:
1. Find terminals: failure_mode, its remediations, detecting tests, owning service
2. Convert to undirected subgraph containing those nodes + 2-hop neighborhood
3. Apply `networkx.algorithms.approximation.steiner_tree(G, terminals, method='mehlhorn')`
4. If result exceeds token budget (rough estimate: 50 tokens per node + edge), trim via PPR within ego graph
5. Return ContextSubgraph dataclass with nodes_by_kind dict

ContextSubgraph.to_llm_prompt_json() serializes to the shape the prompt expects (typed, grouped by kind).

Tests: on fixture graph, assert terminals all present in Steiner output.
```

### Step 4.5: Jinja2 template render

Load into Copilot context: **Spec §6.5 (the entire template)**.

**Prompt**:

```
Create `src/aegis_kg_generator/templates/downstream_dependency_failure.md.j2` with the EXACT template from spec §6.5.

Create `src/aegis_kg_generator/render.py`:
- `render_runbook(failure_mode_id, context, archetype, confidence, provenance, llm_prose=None) -> str`
- Loads template from templates/{archetype}.md.j2
- If llm_prose is None, substitutes placeholder: "[Prose generation disabled]"
- Returns full markdown string

Test: render against fixture graph, assert output contains expected trigger signatures and remediation steps verbatim.
```

### Step 4.6: Confidence scoring

Load into Copilot context: **Spec §6.7**.

**Prompt**:

```
Create `src/aegis_kg_generator/confidence.py` implementing the simplified formula from spec §6.7.

RunbookConfidence dataclass with to_dict() method.

score_confidence() takes (runbook: RenderedRunbook, graph, used_llm: bool, generated_at: datetime).

coverage = count(graph-derived slots populated) / count(total slots)
graph_freshness = exp(-age_days/30)
llm_uncertainty = 0.2 if used_llm else 0.0

Return as dataclass.

Tests: score with/without LLM, varying coverage.
```

### Step 4.7: Wire into CLI

**Prompt**:

```
In aegis-kg-generator cli.py:

Command: `aegis-kg-generate run --graph <path> --out <runbook-dir> --archetype <name> [--no-llm]`

Pipeline:
1. load_graph(graph_dir)
2. For each FailureMode node:
   a. score worthiness; if <0.5, skip (log)
   b. classify archetype; if doesn't match --archetype filter, skip
   c. extract_context
   d. render with llm_prose=None (LLM comes in Session 6)
   e. compute confidence
   f. compute neighborhood_hash (Session 6 stub: placeholder hash)
   g. write to out/<namespace>/<failure_mode_slug>.md

Exit code 0 / 1 / 2 same as extractor.

Progress bar via `rich`.
```

### Week 3 exit check

```bash
cd aegis-kg-generator
uv run aegis-kg-generate run --graph ../examples/hand-written-graph --out /tmp/runbooks-week3 --archetype downstream_dependency_failure --no-llm

cat /tmp/runbooks-week3/*/*.md
# Should see rendered markdown with template-filled slots, placeholder for prose
```

Commit: `feat: generator skeleton with stub-graph input, no LLM yet`

---

## Session 5: OpenAPI + CUE + test mining (week 4, ~10 hours)

### Preconditions
- Session 4 complete

### Red flags
- Copilot skipping graceful degradation when OpenAPI missing — must emit empty + warning, not crash
- Copilot matching operationId case-sensitively — real-world openapi specs are mixed case
- Copilot extracting `ast.Assert` predicates as strings instead of structured representation

### Step 5.1: OpenAPI ingest stage

Load into Copilot context: **Spec §5.5**.

**Prompt**:

```
Add `src/aegis_kg_extractor/stages/openapi_ingest.py` in the extractor repo.

Function: `ingest_openapi(openapi_path: Path, graph: GraphEmitter, discovered_functions: list[Function])`

Steps:
1. Use prance.ResolvingParser(openapi_path, backend='openapi-spec-validator')
2. Walk spec.paths
3. For each (path, method, operation):
   a. Emit Endpoint node (method, path)
   b. If operationId present, try to match against discovered function FQNs:
      - Exact match first
      - Fuzzy via rapidfuzz.process.extractOne, threshold 90
      - On match, emit HANDLED_BY edge
      - On no match, emit Endpoint with handler_fqn=None, gap=unresolved_handler
   c. For each response code in 4xx/5xx, emit FailureMode:
      - name = f"{operation_id}_{status_code}_{description_slug}"
      - http_status from response code
      - category="api_error_response"
      - source="openapi"
      - confidence=1.0
      - Emit CAN_RAISE edge Endpoint → FailureMode

Handle missing operationId: fallback slug per spec §5.5.
Handle missing openapi file: log warning, return, don't crash.

Tests: fixture openapi.yaml with matched + unmatched + mixed-case operationIds.
```

### Step 5.2: CUE ingest

Load into Copilot context: **Spec §5.6**.

**Prompt**:

```
Add `src/aegis_kg_extractor/stages/cue_ingest.py`.

Function: `ingest_cue(cue_files: list[Path], graph: GraphEmitter)`

Two sub-steps:

A) Export to OpenAPI:
   - For each CUE file, run `cue export --out openapi <file>` via subprocess
   - Feed output through ingest_openapi() (Session 5.1)
   - On cue export failure, log and skip

B) Extract ValidationFailureMode:
   - Parse CUE file text via simple regex (we're not writing a full CUE parser)
   - Look for field definitions with constraints:
     - `field: int & >=0` → range constraint
     - `field: string & =~ "regex"` → regex constraint
     - `field: "a" | "b" | "c"` → disjunction/enum
     - `field: type` → required if no default
   - For each constraint, emit ValidationFailureMode node with field_path, constraint_kind, constraint_repr
   - Link via MANIFESTS_IN to all Endpoints that reference that schema

MVP regex-based CUE parsing is fine; full parser is Phase 2.

Tests: fixture CUE files with each constraint kind.
```

### Step 5.3: Test mining — pytest.raises

Load into Copilot context: **Spec §5.4 (the whole subsection)**.

**Prompt**:

```
Add `src/aegis_kg_extractor/stages/test_mining.py`.

Function: `mine_tests(test_files: list[Path], graph: GraphEmitter)`

Sub-extractor 1: PytestRaisesVisitor
- AST NodeVisitor
- visit_With: check if context manager is pytest.raises or _pytest... 
- visit_Call: check for with pytest.raises(...) one-liners
- Extract exception class (handle: ast.Name, ast.Attribute, ast.Tuple for multiple)
- Find enclosing function
- Emit TestInvariant(kind=pytest_raises, test_function_fqn, expected_exception, scenario=None)
- Emit FailureMode placeholder and ASSERTS edge

Handle:
- `pytest.raises(ExcClass, match="...")` — capture match pattern
- `@pytest.mark.xfail(raises=ExcClass)` — scan FunctionDef decorators
- Nested with blocks

Tests: fixture test files with each case.
```

### Step 5.4: Test mining — @patch extraction

**Prompt**:

```
In test_mining.py, add PatchVisitor.

For each @patch(...) decorator on test functions:
- Extract target FQN (string arg)
- Detect side_effect kwarg if present: raises X, returns Y
- For each: emit ExternalCall node (inferred from patch target) + TestDouble edge

For each `with patch(...):` block:
- Same extraction

Match pattern variants:
- @patch("pkg.mod.Thing")
- @patch.object(cls, "method")
- @mock.patch(...)
- @mocker.patch(...) (pytest-mock)

Tests: fixtures with each variant.
```

### Step 5.5: Test mining — ast.Assert extraction

**Prompt**:

```
In test_mining.py, add AssertVisitor.

For each ast.Assert in test functions:
- Capture the test predicate AST
- Classify comparator:
  - ast.Compare with Eq → equality
  - ast.Compare with Lt/Gt/LtE/GtE → range
  - ast.Compare with In/NotIn → membership
  - ast.Call(isinstance, ...) → type
  - Other → generic
- Extract LHS call chain (dotted attribute access) → link to code symbol if FQN resolves
- Emit Invariant node

Don't try to infer scenario — that's Session 6's LLM-verify step.

Tests: assertions with each comparator type.
```

### Step 5.6: Wire all stages into extractor CLI

**Prompt**:

```
Update aegis-kg-extract run pipeline:

After stage 4 (external calls):
5. ingest_openapi if discovered_framework has openapi AND pipeline gets openapi_path
6. ingest_cue if cue_files present
7. mine_tests if test_files present
8. (deferred to Session 6) LLM-verify test names

Each stage wrapped in try/except that logs and continues to next stage. Per §10.2 graceful degradation.

Structured log: one event per stage with duration, input_count, output_count, errors.
```

### Week 4 exit check

```bash
uv run aegis-kg-extract run --repo examples/pilot --out /tmp/graph-week4

# Check FailureMode count
find /tmp/graph-week4/failure_modes -name '*.yaml' | wc -l
# Expect ≥3 per spec §11 Week 4 exit

# Check TestInvariant count
find /tmp/graph-week4/test_invariants -name '*.yaml' | wc -l
# Expect ≥5 per spec §11 Week 4 exit
```

Commit: `feat: OpenAPI + CUE ingest + test mining`

---

## Session 6: LLM prose + CoVe + MCP retrieval (week 5, ~12 hours)

### Preconditions
- Sessions 1-5 complete
- Anthropic API key accessible from Copilot session (or via Copilot's Claude access)

### Red flags
- Copilot writing prose prompts without Pydantic output schema — reject
- Copilot using LangChain — reject; spec §ADR-008 locks us to pure asyncio + anthropic SDK
- Copilot's CoVe implementation sharing context across verification calls — reject, independence is the whole point
- Copilot omitting the structured pre-filter in retrieval — this is the safety rail, not optional

### Step 6.1: LLM prose with CoVe

Load into Copilot context: **Spec §6.6 (whole subsection)**.

**Prompt**:

```
In aegis-kg-generator, create `src/aegis_kg_generator/llm_prose.py`.

Dependencies: anthropic, pydantic.

Classes:
- ProseSlot(enum): WHY_THIS_HAPPENS, ESCALATION_GUIDANCE
- ProseDraft(BaseModel): slot, text, noun_phrases: list[str]
- CoVeResult(BaseModel): verified: bool, unverified_phrases: list[str], log: list[dict]

Main function:
async def generate_prose(
    slot: ProseSlot,
    context: ContextSubgraph,
    opus_client: AsyncClient,
    sonnet_client: AsyncClient,
    max_retries: int = 2,
) -> Optional[str]:
    """
    1. Draft with Opus at temperature=0.2. Prompt requires Pydantic tool-use output shape ProseDraft.
    2. Extract noun phrases from draft.
    3. For each noun phrase, fire INDEPENDENT Sonnet verification call (new Message, no shared history): "Is the phrase <X> present in this context? <context JSON> Answer yes or no with a literal quote if yes."
    4. Run verifications in parallel via asyncio.gather
    5. If all verified: return text
    6. If any fail: retry draft with stricter prompt appending unverified phrases as banned words
    7. After max_retries: return None (caller falls back to template-only)
    
    Log every draft + verification to .aegis-kg-cache/cove-log.jsonl (append).
    Timeout each call at 30s; use asyncio.timeout.
"""

Prompts:
- Draft prompt: strict instruction to only use facts in context, target 2-4 sentences, tool-use output
- Verification prompt: minimal, single-turn, no memory

Tests: mock both clients, assert retry on verification failure, assert fallback to None after max retries.
```

### Step 6.2: Neighborhood hash

Load into Copilot context: **Spec §6.8**.

**Prompt**:

```
In aegis-kg-generator, create `src/aegis_kg_generator/merkle.py`.

Function: `neighborhood_hash(node_id, graph, radius=2) -> str`

Implementation exactly per spec §6.8.

Node hash: sha256 of canonicalized node JSON with source.file_hash and source.symbol_hash but WITHOUT source.line or timestamps.

Edge hash: sha256 of (src_id, rel_type, dst_id, confidence bucket) where confidence bucket is "high" (>=0.9), "medium" (0.6-0.9), "low" (<0.6) to tolerate small confidence drift.

Tests: mutating a node in the 2-hop neighborhood changes hash; mutating outside doesn't.
```

### Step 6.3: Update generator to use LLM + hash

**Prompt**:

```
Update aegis_kg_generator/cli.py:

Add --llm / --no-llm flag (default: --llm if ANTHROPIC_API_KEY env var set, else --no-llm).

In pipeline, after render:
- If --llm, call generate_prose() for each prose slot, await results
- Substitute into rendered template
- Set used_llm=True in confidence
- neighborhood_hash = merkle.neighborhood_hash(failure_mode_id, graph)

Write YAML frontmatter to runbook file:
---
runbook_id: <namespace>/<slug>
archetype: downstream_dependency_failure
confidence: <0.0-1.0>
confidence_breakdown: {coverage: ..., graph_freshness: ..., llm_uncertainty: ...}
neighborhood_hash: <sha256>
provenance:
  commit_sha: <git>
  extractor_version: <version>
  generator_version: <version>
  model: claude-opus-4-7 (or null if no LLM)
  generated_at: <iso-8601>
---

<markdown body>

Parse existing runbooks (if overwriting) and preserve `human_edited_sections` block if present — never silently overwrite human edits.
```

### Step 6.4: MCP retrieval server — skeleton

Load into Copilot context: **Spec §7 (whole section)**.

**Prompt**:

```
In aegis-kg-retrieval-mcp, create `src/aegis_kg_retrieval_mcp/server.py`.

Use the official `mcp` Python SDK (import mcp.server.stdio, mcp.server.types).

Create MCP server exposing tools from spec §7.1:
- retrieve_runbook(incident_description: str, service: str = None, endpoint: str = None, exception: str = None) -> list[dict]
- search_failure_modes(query: str, limit: int = 5) -> list[dict]
- explain_runbook(runbook_id: str) -> dict
- list_services() -> list[dict]
- get_graph_node(node_id: str) -> dict

For each tool, define tool schema (name, description, inputSchema as JSON schema) via mcp SDK.

Entry point: stdio transport for MVP (server.run_stdio_async()).

Load graph + runbooks on startup. Log to stderr; stdout is MCP protocol.

Add --graph <path> --runbooks <path> CLI flags via argparse.

Tests: use mcp test client to invoke tools against fixture graph + runbooks.
```

### Step 6.5: Retrieval index

Load into Copilot context: **Spec §7.2, §7.3**.

**Prompt**:

```
Create `src/aegis_kg_retrieval_mcp/index.py`.

Class RetrievalIndex per spec §7.3, with these specifics:

__init__(graph_dir, runbooks_dir):
  - Load all runbooks (parse frontmatter + body)
  - Extract trigger_signature from frontmatter (fall back: parse from graph if not in frontmatter)
  - Build BM25 (rank_bm25.BM25Okapi) over lexical tokens
  - Compute embeddings via sentence-transformers BAAI/bge-small-en-v1.5 for embedding_text
  - Build hnswlib HNSW index (ef=200, M=16; 384-dim cosine)
  - Load cross-encoder cross-encoder/ms-marco-MiniLM-L-6-v2
  - Load graph for structured pre-filter

retrieve(query, structured, k=5):
  - BM25 top-50
  - HNSW top-50 (encode query with same model)
  - Reciprocal Rank Fusion (k_rrf=60)
  - STRUCTURED PRE-FILTER — reject where trigger_signature.structured.service doesn't match (after fuzzy normalization, e.g., payments-api matches payments)
  - Cross-encoder rerank top-20
  - Return top-k with scores + explanations

Cache embeddings per text hash at .aegis-kg-cache/embeddings/.

Tests: fixture with 10 runbooks across 3 services; assert service filter rejects cross-service retrievals; assert reranker changes order.
```

### Step 6.6: Wire MCP server to index

**Prompt**:

```
Update server.py to construct RetrievalIndex on startup and route each tool to appropriate method:
- retrieve_runbook → index.retrieve
- search_failure_modes → index.search_failures (add this method: same as retrieve but returns FailureMode nodes, not runbooks)
- explain_runbook → index.explain (loads runbook + ego_graph + provenance)
- list_services → from graph
- get_graph_node → from graph

Each tool returns JSON-serializable dict; MCP SDK handles protocol wrapping.
```

### Week 5 exit check

```bash
# Run full pipeline on hand-written graph (not real extractor output)
cd aegis-kg-generator
export ANTHROPIC_API_KEY=...
uv run aegis-kg-generate run --graph ../examples/hand-written-graph --out /tmp/runbooks-week5

# Inspect cove-log.jsonl
cat .aegis-kg-cache/cove-log.jsonl | jq .

# Start MCP server
cd ../aegis-kg-retrieval-mcp
uv run aegis-kg-mcp --graph ../examples/hand-written-graph --runbooks /tmp/runbooks-week5 --transport stdio &
MCP_PID=$!

# From another terminal, test via a simple MCP client script
uv run python scripts/test_mcp_client.py
# (write this script: invoke retrieve_runbook with sample incident, print result)

kill $MCP_PID
```

Commit: `feat: LLM prose with CoVe + retrieval MCP server`

---

## Session 7: Dashboard + CI + full-stack demo (week 6, ~10 hours)

### Preconditions
- Sessions 1-6 complete
- GitHub Actions access on all four repos

### Red flags
- Copilot reaching for a frontend build system (webpack, vite) — reject, single HTML per spec §7
- Copilot using server-side state in dashboard — reject, single HTML, no backend
- Copilot skipping keyboard shortcuts — they're the whole point of the triage UX

### Step 7.1: Dashboard HTML scaffold

Load into Copilot context: **Spec §8.2**.

**Prompt**:

```
In aegis-kg-generator, create `src/aegis_kg_generator/dashboard/index.html`.

Single file. Load from CDN:
- Cytoscape.js
- marked.js (markdown rendering)
- tailwindcss via cdn (Play CDN is fine for MVP)

Layout: 4 panes per spec §8.2:
- Left 20%: triage queue (list of runbooks sorted by priority)
- Center 40%: runbook markdown preview
- Right 25%: Cytoscape.js graph neighborhood
- Bottom 15%: provenance + "explain this generation" breakdown

Data loading:
- fetch('./data/runbooks.json') — JSON array of all runbooks with frontmatter + body + scores
- fetch('./data/graph.json') — graph as Cytoscape JSON
- Both are pre-exported by generator's new `aegis-kg-generate export-dashboard <runbook-dir>` command

Keyboard shortcuts per spec §8.2 "Keyboard shortcuts".

Approve/Reject/Edit actions emit a patch file downloaded to disk (since no server):
- approve.patch: list of runbook IDs + status=approved
- reject.patch: list of runbook IDs + status=rejected + reason
User applies patches to the actual repo via `aegis-kg-generate apply-patch <file>`.

Don't overthink styling. Function over form.

Tests: none (manual test).
```

### Step 7.2: Export data for dashboard

**Prompt**:

```
Add command to aegis-kg-generator CLI:
`aegis-kg-generate export-dashboard --graph <path> --runbooks <path> --out <dashboard-dir>`

Copies dashboard/index.html to <dashboard-dir>/index.html.
Writes:
- <dashboard-dir>/data/runbooks.json: every runbook with frontmatter parsed + body preserved as markdown string + computed priority
- <dashboard-dir>/data/graph.json: full graph as Cytoscape JSON (nodes with kind, label; edges with rel)

Priority per spec §8.4 simplified formula:
priority = (1 - confidence) * 0.5 + centrality * 0.3 + change_magnitude * 0.2
(centrality from PPR on graph; change_magnitude stub=0 for now — Phase 2)

Manual test: open in browser, verify queue, verify keyboard nav works.
```

### Step 7.3: Apply-patch command

**Prompt**:

```
Add CLI:
`aegis-kg-generate apply-patch <patch-file> --runbooks <dir>`

Parses patch file (JSONL with action + runbook_id + optional fields).
For each patch:
- approve: add `status.review: approved` to runbook YAML frontmatter, `status.reviewer: <from env>` if set
- reject: add status.review: rejected, status.reason
- edit: requires body field; overwrite body, add human_edited_sections marker

Canonicalize on write. Git-add the change. Print summary.
```

### Step 7.4: GitHub Action — PR check

Load into Copilot context: **Spec §9 (whole section)**.

**Prompt**:

```
Create .github/workflows/aegis-kg-pr.yml in pilot repo (NOT in aegis-kg-* repos — this is for the target service).

Triggers: pull_request on any path.

Jobs:
1. extract: uv run aegis-kg-extract run against PR head
2. diff: compare against graph/ in main branch via DeepDiff
   - Post PR comment with added/removed/modified node counts
3. generate: regenerate runbooks for changed FailureModes
4. stale-check: per spec §4 stale-runbook Merkle hash detection
   - Post PR comment listing stale runbooks
5. upload dashboard snapshot as workflow artifact

Use actions/cache for:
- AST trees: key hashFiles('**/*.py') + pyproject
- Call graph: key by HEAD SHA
- Embeddings: key by runbook content hashes

Each step has timeout (5 min max).
```

### Step 7.5: Demo script

**Prompt**:

```
Create scripts/demo.sh in your top-level workspace that demonstrates the full flow for Kunal:

1. Echo header: "AEGIS KG — Phase 1 MVP Demo"
2. Show pilot repo tree (truncated)
3. Run aegis-kg-extract run against pilot, print stats (nodes, edges, elapsed)
4. Run aegis-kg-generate run, print stats (runbooks generated, avg confidence, CoVe pass rate)
5. Export dashboard data; print URL to open
6. Start MCP server in background (stdio transport — but for demo, use mcp-test-client to stand in for AEGIS)
7. Invoke MCP retrieve_runbook with a canned incident description ("503 errors on /charge endpoint, stripe timeouts")
8. Print retrieval results with scores + top runbook
9. Cleanup

Make it work on a fresh checkout with nothing but `uv` installed. Total runtime <3 min.
```

### Week 6 exit check

```bash
# Run demo script
./scripts/demo.sh

# Open dashboard, triage 3 runbooks via keyboard
# Apply the generated patch
uv run aegis-kg-generate apply-patch approve.patch --runbooks /tmp/runbooks-week6

# Commit. Open PR. Verify GH Action runs.
```

**Demo to Kunal + CDS team this week.** Exit criterion: one runbook merged via PR by a human SME.

Commit: `feat: dashboard + GH Actions + demo`

---

## Session 8: Wire real extractor output end-to-end (week 7, ~10 hours)

### Preconditions
- Sessions 1-7 complete
- Demo held with Kunal; feedback captured

### Red flags
- Tempting to add features from feedback — resist, log in Phase 2 backlog instead
- Real-data issues are not bugs in the code, they're gaps in assumptions — document, don't hide
- Copilot wanting to add "quick fixes" — every fix must have a test

This session is about shifting from hand-written graph fixtures to actual extractor output. It's more debugging than building.

### Step 8.1: Run full pipeline on real pilot

```bash
# Clean slate
rm -rf /tmp/graph-real /tmp/runbooks-real

uv run aegis-kg-extract run --repo examples/pilot --out /tmp/graph-real --verbose 2>&1 | tee extract.log
uv run aegis-kg-generate run --graph /tmp/graph-real --out /tmp/runbooks-real --verbose 2>&1 | tee generate.log
```

### Step 8.2: Common real-data issues — triage

Expect to find and fix (paste each to Copilot with the specific error):

**Issue A: Call graph recall too low**

```
Scalpel call-graph recall on pilot is <70%. Manual call-graph construction from pilot traces / inspection suggests we're missing:
<paste list of specific missing edges>

Options:
1. Layer Pyan3 output for additional coverage
2. Add explicit framework-decorator detection (FastAPI @Depends chains)
3. Drop to low-confidence edges from mypy's inferred types

Which is best for our pilot?
```

**Issue B: Many ExternalCall nodes have confidence=low / unresolved target**

```
30% of ExternalCall nodes have target_hint="<unresolved>". Inspection shows these are f-string URLs or URLs from imported config modules.

Write a resolver in aegis_kg_extractor/stages/external_calls.py that does ONE LEVEL of config module following:
- If URL is a Name, find the assignment in the same module
- If that assignment is itself a Name imported from another module, follow the import and look for assignment
- Single module follow only; no recursion

Tests: fixture with config-module pattern.
```

**Issue C: OpenAPI operation_id fuzzy match rate low**

```
Fuzzy match between operationId and function FQN is returning weak matches. Real operation_ids are snake_case and function FQNs are module.snake_case.

In openapi_ingest.py, before rapidfuzz call, normalize both sides:
- lowercase
- strip module prefix (last dotted component only)
- replace underscores with spaces

If still no match at threshold 90, also try exact match on just the function name (not FQN).

Add to tests.
```

### Step 8.3: Evaluation harness

**Prompt**:

```
Create scripts/evaluate.py.

Input: a labeled set of synthetic incidents (scripts/synthetic_incidents.yaml) — 5 incidents minimum, each with:
- incident_description: str
- expected_service: str
- expected_runbook_id: str (or list of acceptable runbooks)
- expected_top_k: 1 | 3 | 5

Runs:
1. Starts MCP server against real graph + runbooks
2. For each incident, invokes retrieve_runbook
3. Computes:
   - precision@1: did expected_runbook appear in position 1?
   - precision@5: did it appear in top 5?
   - service_match_rate: did top result have correct service?

Outputs summary table + fails if precision@5 < 60% or service_match < 80% (spec §13 targets).

Write synthetic_incidents.yaml with 5 plausible incidents based on pilot's failure modes.
```

### Step 8.4: Nightly regeneration stability test

```bash
# Set up cron or launchd to run 3 nights:
./scripts/run_nightly.sh >> /tmp/nightly.log 2>&1

# Compare graphs across nights
diff <(git log --oneline --since="3 days ago" -- graph/) expected_clean_log.txt
# Should show minimal churn — ideally zero spurious diffs
```

If spurious diffs appear, the most common culprits:
- Non-canonical serialization (check JCS impl)
- Timestamps leaking into hashes
- Set ordering not deterministic somewhere

### Step 8.5: Document real-data gaps

Create `PHASE2_BACKLOG.md` in `aegis-kg-schema` repo. For every issue you triaged in 8.2:
- Issue title
- Real-data root cause
- Workaround applied in MVP
- Proposed proper fix for Phase 2
- Estimated complexity

This doc is what drives Phase 2 planning.

### Week 7 exit check

```bash
# Run evaluation against real pilot data
uv run python scripts/evaluate.py

# Expected (per spec §13):
# - Service match rate ≥80%
# - Precision@5 ≥60%

# If pass: publish metrics to Kunal, declare MVP done.
# If fail: document which metric failed, add to PHASE2_BACKLOG.md, do NOT declare done.
```

Commit: `feat: real extractor output wired end-to-end + evaluation harness`

---

## Session N+1: What comes next (post-MVP)

Do not start Phase 2 work in the MVP branch. Open separate branches per Phase 2 workstream.

Phase 2 roadmap (from research §8):
- **WS-A** Graph quality: test mining expansion, mutation testing, Hypothesis
- **WS-B** Archetype expansion: 5 more runbook archetypes
- **WS-C** Retrieval + AEGIS integration: calibration, incident feedback loop, HippoRAG

For each, write a new spec + vibe guide modeled on these two files. Do not try to reuse this guide for Phase 2 — the scope and risks differ.

---

## Appendix A: Copilot anti-patterns to reject

- **Copilot proposes to refactor cross-repo**: reject; we established boundaries
- **Copilot adds dependencies not in spec §17**: reject; add to Phase 2 if needed
- **Copilot writes synchronous code where async is specified**: reject; consistency with AEGIS
- **Copilot generates tests only after code**: write tests first or in parallel; every new function gets a test
- **Copilot's error handling is `except Exception`**: reject; specific exception types only
- **Copilot reaches for regex where AST walking works**: reject; spec requires AST per §5.3
- **Copilot adds `TODO: handle edge case X`**: either handle it now or document in PHASE2_BACKLOG; no silent TODOs
- **Copilot pads prompts with "please" and apologies**: edit these out of your own prompts; waste of tokens

---

## Appendix B: Session recovery — when Copilot goes off the rails

Signs Copilot is confused:
- Generating code for a different framework than specified
- Inventing class/function names from earlier sessions
- Code references imports that don't exist in pyproject
- Output length growing dramatically without adding value

Recovery:
1. Close current Copilot chat
2. Open new chat
3. Paste: "I'm resuming work on AEGIS KG. I'm at Session <N>, Step <X>. Here's the current state: <paste git status + most recent file>. Here's the spec section: <paste relevant spec subsection>. Continue from <specific thing>."
4. If still confused, reduce scope: break the task into smaller prompts

---

## Appendix C: Critical files reference

After Week 7, you should have these files. If any are missing, you skipped something.

```
aegis-kg-schema/
├── schemas/v1alpha1.cue
├── src/aegis_kg_schema/
│   ├── models.py
│   ├── canonicalize.py
│   └── validate.py
├── migrations/
│   └── 0001_init.py
└── PHASE2_BACKLOG.md

aegis-kg-extractor/
├── src/aegis_kg_extractor/
│   ├── cli.py
│   ├── adapters/
│   │   ├── base.py
│   │   ├── registry.py
│   │   ├── fastapi_adapter.py
│   │   └── generic_python.py
│   ├── stages/
│   │   ├── static_ast.py
│   │   ├── call_graph.py
│   │   ├── external_calls.py
│   │   ├── openapi_ingest.py
│   │   ├── cue_ingest.py
│   │   └── test_mining.py
│   └── emit.py

aegis-kg-generator/
├── src/aegis_kg_generator/
│   ├── cli.py
│   ├── graph_loader.py
│   ├── worthiness.py
│   ├── archetypes/
│   │   ├── __init__.py
│   │   └── downstream_dependency_failure.py
│   ├── context.py
│   ├── render.py
│   ├── confidence.py
│   ├── merkle.py
│   ├── llm_prose.py
│   ├── templates/
│   │   └── downstream_dependency_failure.md.j2
│   └── dashboard/
│       └── index.html

aegis-kg-retrieval-mcp/
├── src/aegis_kg_retrieval_mcp/
│   ├── server.py
│   └── index.py

(pilot service repo)
├── .github/workflows/aegis-kg-pr.yml
├── graph/ (generated)
├── runbooks/ (generated)
└── scripts/
    ├── demo.sh
    ├── evaluate.py
    └── synthetic_incidents.yaml
```

---

*End of vibe coding guide. Build well.*
