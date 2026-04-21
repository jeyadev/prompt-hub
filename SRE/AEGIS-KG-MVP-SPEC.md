# AEGIS Knowledge Graph — Phase 1 MVP Specification

**Version:** 0.1.0
**Status:** Draft — pre-implementation
**Owner:** Jey / CDS SRE
**Scope:** Phase 1 MVP only. Phase 2-3 deferred to separate specs.
**Timeline:** 7 weeks, 2 FTE from the 10-person team
**Last updated:** April 2026

---

## 1. Purpose and Intent

### 1.1 What this tool is

A repo-to-runbook pipeline that ingests a service's source code, test suite, OpenAPI spec, and (optionally) CUE schemas, and produces:

1. A **file-based knowledge graph** serialized as multi-file YAML in Git
2. A **materialized runbook corpus** derived deterministically from graph traversals
3. An **MCP server** exposing retrieval over the graph and runbooks, consumable by AEGIS

### 1.2 Why this exists

AEGIS's current design requires teams to hand-author runbooks before onboarding. For a platform with 12 services and 100+ APIs, this is a multi-week authoring exercise per team — the dominant adoption blocker.

This tool collapses that to hours: a team points the extractor at their repo, reviews the generated graph + runbooks, and ships an AEGIS integration in days, not weeks.

### 1.3 Strategic alignment

- Complements AEGIS architecture: **known failures via deterministic retrieval here, novel failures via AEGIS Strategist LLM reasoning**
- Honors deterministic-before-probabilistic: static analysis first, LLM only where static fails, LLM always gated by verifier
- Complements pre-generated runbook system (Phase 1 of that initiative): this tool *is* the runbook generator
- Complements L2 support tool: same graph, multiple consumers

### 1.4 What this MVP does NOT do (explicit deferrals)

| Deferred to Phase 2 | Reason |
|---|---|
| Mutation testing integration (mutmut, cosmic-ray) | Requires stable test-mining foundation first |
| Hypothesis strategy mining | Low coverage in target codebases; Phase 2 when T2 inputs mature |
| Leiden community detection | Not needed until KG has >5 services |
| GraphRAG / HippoRAG retrieval | Over-engineered for MVP; BM25+HNSW+rerank is sufficient |
| Incident data ingestion (ServiceNow/Jira) | Requires separate NER pipeline; Phase 2 T4 |
| Splunk log parsing (Drain3) | Requires separate log access; Phase 2 T4 |
| Additional runbook archetypes (5 more) | MVP ships one; proves the template-plus-prose pattern |
| Calibration dashboard with reliability diagrams | Requires labeled approval history; Phase 2 after data collects |
| Conformal prediction for auto-approval | Same data dependency |
| GNN anomaly detection | Overkill; Phase 3 at scale |
| Frequent subgraph mining (gSpan) | Needs ≥100 labeled incidents |

**Anyone proposing to add these to MVP scope: read §1.4. Not now.**

---

## 2. Architecture Decision Record (ADR) Log

### ADR-001: Four-repo split, not monorepo

**Decision**: Separate into `aegis-kg-schema`, `aegis-kg-extractor`, `aegis-kg-generator`, `aegis-kg-retrieval-mcp`.

**Rationale**: Clean boundaries, independent versioning, CODEOWNERS separation, independently deployable. Schema is the keystone contract.

**Tradeoff accepted**: Coordination cost with 2 FTE. Mitigated by CI contract (§10.5).

**Alternatives rejected**: Monorepo (faster short-term, but schema changes become cross-cutting and painful at scale); three-repo (merging generator+retrieval couples runbook materialization timing with retrieval service uptime).

### ADR-002: Framework-agnostic extractor with adapter pattern

**Decision**: Core extractor framework-neutral. Framework-specific logic lives in `FrameworkAdapter` implementations. Ship with FastAPI adapter as reference.

**Rationale**: JPMC's CDS platform uses multiple frameworks; hardcoding FastAPI forces a retrofit. Adapter pattern isolates framework churn.

**Tradeoff accepted**: More design work upfront. Adapter interface adds one level of indirection.

### ADR-003: Backstage `apiVersion/kind/metadata/spec` envelope

**Decision**: Every node file uses Backstage's entity envelope.

**Rationale**: Inherits proven ergonomics (Kubernetes, Grafana V2, Crossplane converged on this). Permissive unknown-field handling. CODEOWNERS-compatible. Already industry-standard for service catalogs.

**Alternatives rejected**: Custom envelope (reinventing wheel); JSON-LD (graph-diff churn); GraphML (binary-ish, poor diffs).

### ADR-004: Multi-file YAML-per-entity + JSONL for high-volume derived data

**Decision**: One YAML file per node under `graph/<kind>/<namespace>/<name>.yaml`. Edges embedded in `spec.relations[]` by default; edge-with-properties as separate files. High-volume derived data (call graphs) as JSONL under `graph/derived/`.

**Rationale**: Best Git diffability (per research §1 format comparison table). Parallel merges work cleanly. `git blame` is meaningful.

**Tradeoff accepted**: More files. Mitigated by sharded directory layout.

### ADR-005: RFC 8785 JCS canonicalization on every write

**Decision**: All JSON/YAML writes go through a canonicalizer that enforces sorted keys, UTF-8, stable list ordering, explicit null handling.

**Rationale**: Grafana issue #111400 is the cautionary tale — inconsistent serialization produces false-positive diffs that erode reviewer trust.

**Implementation**: `rfc8785` Python library for JSON; for YAML, sort keys alphabetically, preserve quote style, strip trailing whitespace.

### ADR-006: MCP-native retrieval server

**Decision**: Retrieval exposed as an MCP server using the official `mcp` Python SDK. No plain-Python library interface.

**Rationale**: Matches existing MCP ecosystem (Splunk, Domain MCP, ServiceNow). AEGIS calls it the same way it calls everything else. No special-case client code.

**Caveat**: Production auth/deploy pattern on Gaia/CPSR is a platform-team question, not resolved in this spec. Local dev uses stdio transport.

### ADR-007: One runbook archetype in MVP

**Decision**: Ship exactly one archetype — "downstream dependency failure" — in MVP. Five more in Phase 2.

**Rationale**: Proves the template-plus-prose hybrid. Adding archetypes is linear work once the pattern is validated.

**Selection rationale for this specific archetype**: Every service has downstream calls. Derivable from pure static analysis (HTTP/gRPC client call detection). Highest-coverage failure class in distributed systems.

### ADR-008: Claude Opus for prose, Sonnet for classification

**Decision**: Opus for runbook prose sections. Sonnet for dedup classification, node summarization, extraction verification.

**Rationale**: Opus is quality-sensitive / low-volume fit. Sonnet is latency-sensitive / high-volume fit. Matches Jey's existing model-routing pattern.

**Fallback**: Deterministic code path if LLM unavailable. Runbook generation degrades to template-only output with `llm_prose_pending: true` flag.

### ADR-009: DeepDiff + Merkle-hash stale detection

**Decision**: Graph diff via `DeepDiff(ignore_order=True)`. Runbook staleness via per-runbook Merkle hash of its neighborhood.

**Rationale**: `DeepDiff` is production-grade, gives path-level change reports. Merkle hashing over sorted neighborhood triples is the proven algorithm for invalidation in content-addressed systems (Git, IPFS, Nix).

### ADR-010: Pydantic validation + Chain-of-Verification (CoVe) on every LLM extraction

**Decision**: Every LLM output is Pydantic-validated; `evidence_span` must be a literal substring of the source; independent verification call checks the span is present.

**Rationale**: Most robust published defense against hallucination compounding. Turns the schema into a hard validator.

### ADR-011: Week 1-2 stubs; real end-to-end demo Week 6; real extractor wired Week 7

**Decision**: Build schema + extractor skeleton weeks 1-2. Generator and retrieval use stub graph data weeks 3-5. Full working demo with hand-written graph at Week 6. Real extractor output wired in Week 7.

**Rationale**: Vertical-slice methodology. Management sees a working demo at Week 6 — delaying to Week 8 erodes buy-in. Week 7 is real-data hardening.

---

## 3. System Architecture

### 3.1 High-level data flow

```
                                                ┌──────────────────┐
                                                │  aegis-kg-schema │  ◄── pinned version
                                                │  (CUE + Pydantic)│       (all consumers)
                                                └────────┬─────────┘
                                                         │
                                                         │ schema contract
                                                         │
   ┌─────────────────────┐     ┌─────────────────────┐  │  ┌─────────────────────┐
   │  pilot service repo │────►│  aegis-kg-extractor │──┼─►│  aegis-kg-generator │
   │                     │     │                     │  │  │                     │
   │  - Python source    │     │  - AST walker       │  │  │  - graph traversal  │
   │  - OpenAPI spec     │     │  - Scalpel/PyCG     │  │  │  - runbook template │
   │  - pytest suite     │     │  - OpenAPI ingest   │  │  │  - LLM prose (CoVe) │
   │  - CUE schemas      │     │  - CUE constraints  │  │  │  - confidence score │
   └─────────────────────┘     │  - test mining      │  │  └──────────┬──────────┘
                               └──────────┬──────────┘  │             │
                                          │             │             │
                                          ▼             │             ▼
                                  ┌──────────────┐      │     ┌──────────────┐
                                  │  graph/ dir  │──────┼────►│ runbooks/ dir│
                                  │  (YAML files)│      │     │ (YAML+MD)    │
                                  └──────┬───────┘      │     └──────┬───────┘
                                         │              │            │
                                         └──────────┬───┴────────────┘
                                                    │
                                                    ▼
                                      ┌───────────────────────────┐
                                      │  aegis-kg-retrieval-mcp   │
                                      │                           │
                                      │  - BM25 + HNSW + rerank   │
                                      │  - MCP server (stdio/SSE) │
                                      │  - tools: retrieve,       │
                                      │    search, explain        │
                                      └──────────────┬────────────┘
                                                     │
                                                     │ MCP protocol
                                                     ▼
                                              ┌─────────────┐
                                              │    AEGIS    │
                                              │ Strategist  │
                                              └─────────────┘
```

### 3.2 Repo responsibilities

**`aegis-kg-schema`**: CUE schema definitions, Pydantic model generation, schema migrations. Versioned via semver. Every other repo pins an exact version. **Nothing else imports the schema — it imports itself into the others.**

**`aegis-kg-extractor`**: Reads a target repo, produces graph YAML files. Output-only, stateless. CLI: `aegis-kg-extract --repo <path> --out <graph-dir>`. Frameworks plug in via adapter interface.

**`aegis-kg-generator`**: Reads a graph directory, produces runbook YAML+MD files. Output-only, stateless. CLI: `aegis-kg-generate --graph <path> --out <runbook-dir> --archetype downstream_dependency_failure`.

**`aegis-kg-retrieval-mcp`**: Reads graph + runbooks, serves MCP. Stateful at runtime (indices in memory). CLI: `aegis-kg-mcp --graph <path> --runbooks <path> --transport stdio`.

### 3.3 Contracts between repos

| Contract | Producer | Consumer | Versioned by |
|---|---|---|---|
| Node schema (YAML shape) | `schema` | `extractor` writes, `generator` reads | `schema.version` |
| Runbook schema (YAML+MD shape) | `schema` | `generator` writes, `retrieval` reads | `schema.version` |
| MCP tool schema | `schema` | `retrieval` serves, AEGIS calls | `schema.version` |
| Framework adapter interface | `extractor` | third-party adapters | extractor package version |

**Schema-version pinning**: each consumer declares `schema_version: ">=1.0,<2.0"` in its pyproject. CI fails any consumer PR that uses a schema field introduced after the pinned version range.

---

## 4. Graph Schema (concrete)

### 4.1 Envelope (applies to all nodes)

```yaml
apiVersion: aegis.dev/v1alpha1
kind: <NodeKind>                    # Service | API | Endpoint | FailureMode | ...
metadata:
  name: <stable-id>                 # human-readable, content-derived
  namespace: <owning-service>       # scoping
  schemaVersion: 1                  # per-file integer, Grafana pattern
  labels: {}                        # free-form tags
  annotations:                      # provenance, always required
    source.file_hash: <sha256>
    source.symbol_hash: <sha256>    # line-number-independent
    source.extractor: <extractor-name@version>
    source.commit_sha: <git-sha>
    source.path: <relative-path>
    source.line: <int>              # optional; best-effort
spec: {}                            # kind-specific
status: {}                          # optional, computed at generation time
```

### 4.2 Core node kinds (MVP)

#### `Service`

```yaml
kind: Service
metadata:
  name: payments-api
  namespace: payments-api
  annotations:
    source.extractor: aegis-kg-extractor@0.1.0
    source.commit_sha: abc123
spec:
  identity:                         # immutable, OTel entity model
    service.name: payments-api
  descriptive:                      # drift-tolerant
    description: "Payment processing API"
    owner:
      kind: Team
      email: payments-sre@example.com
  repo:
    root: .
    framework: fastapi              # detected; "unknown" allowed
  relations:
    - type: OWNS
      target: "API:payments-api/v1"
    - type: DEPENDS_ON
      target: "Service:stripe-gateway"
      confidence: 0.95
      source: static
```

#### `API`

```yaml
kind: API
metadata:
  name: payments-api-v1
  namespace: payments-api
spec:
  identity:
    api.name: payments-api
    api.version: v1
  descriptive:
    protocol: http                  # http | grpc | async
    spec_ref: "openapi.yaml"
  relations:
    - type: EXPOSES
      target: "Endpoint:payments-api/POST_/charge"
    - type: OWNED_BY
      target: "Service:payments-api"
```

#### `Endpoint`

```yaml
kind: Endpoint
metadata:
  name: POST_/charge
  namespace: payments-api
spec:
  identity:
    http.method: POST
    http.route: /charge
  descriptive:
    operation_id: charge_card
    handler_fqn: payments.api.charge.charge_card
    summary: "Charge a credit card"
  relations:
    - type: HANDLED_BY
      target: "Function:payments.api.charge.charge_card"
    - type: CAN_RAISE
      target: "FailureMode:payments-api/charge_stripe_timeout"
      confidence: 0.85
      source: static+openapi
    - type: CAN_RAISE
      target: "FailureMode:payments-api/charge_invalid_input"
      confidence: 1.0
      source: openapi
```

#### `FailureMode`

```yaml
kind: FailureMode
metadata:
  name: charge_stripe_timeout
  namespace: payments-api
spec:
  identity:
    failure.signature: "downstream.http.timeout:stripe"
  descriptive:
    summary: "Stripe API call timed out during /charge"
    category: downstream_dependency_failure
    http_status: 502
    exception_type: requests.exceptions.Timeout
    trigger_signature:
      lexical:
        - "stripe timeout"
        - "stripe.com"
        - "TimeoutError"
      embedding_text: "Stripe API downstream HTTP timeout during charge endpoint"
      structured:
        service: payments-api
        endpoint: POST_/charge
        exception: requests.exceptions.Timeout
  relations:
    - type: MANIFESTS_IN
      target: "Endpoint:payments-api/POST_/charge"
    - type: DETECTED_BY
      target: "TestInvariant:payments-api/test_charge_handles_stripe_timeout"
      confidence: 1.0
    - type: HAS_REMEDIATION
      target: "RemediationAction:payments-api/retry_with_backoff"
      confidence: 0.8
  status:
    runbook_worthiness_score: 0.87
    runbook_generated: true
    runbook_ref: "runbooks/payments-api/charge_stripe_timeout.md"
```

#### `TestInvariant`

```yaml
kind: TestInvariant
metadata:
  name: test_charge_handles_stripe_timeout
  namespace: payments-api
spec:
  identity:
    invariant.kind: pytest_raises
  descriptive:
    test_function_fqn: tests.test_charge.test_charge_handles_stripe_timeout
    scenario: "When Stripe times out, /charge returns 502 and does not retry silently"
    expected_exception: requests.exceptions.Timeout
    mocks:
      - target: "stripe.Charge.create"
        side_effect: "Timeout"
  relations:
    - type: ASSERTS
      target: "FailureMode:payments-api/charge_stripe_timeout"
```

#### `RemediationAction`

```yaml
kind: RemediationAction
metadata:
  name: retry_with_backoff
  namespace: payments-api
spec:
  identity:
    action.name: retry_with_backoff
  descriptive:
    summary: "Retry failed request with exponential backoff"
    action_type: code_change      # code_change | runtime_toggle | escalation
    code_reference:
      symbol: payments.api.charge.charge_card
      line_range: [45, 62]
  relations:
    - type: REMEDIATES
      target: "FailureMode:payments-api/charge_stripe_timeout"
```

#### `ExternalCall`

```yaml
kind: ExternalCall
metadata:
  name: stripe_charge_create
  namespace: payments-api
spec:
  identity:
    call.protocol: http
    call.target_hint: "stripe.com"
  descriptive:
    client_library: requests
    literal_url: null               # if resolved: "https://api.stripe.com/v1/charges"
    confidence: medium              # high | medium | low
  relations:
    - type: CALLED_BY
      target: "Function:payments.api.charge.charge_card"
```

### 4.3 Edge types (canonical list)

| Type | From → To | Source | Confidence |
|---|---|---|---|
| OWNS | Service → API | OpenAPI/config | 1.0 |
| EXPOSES | API → Endpoint | OpenAPI | 1.0 |
| HANDLED_BY | Endpoint → Function | OpenAPI + static | 0.9-1.0 |
| CALLS | Function → Function | Scalpel/PyCG | 0.6-1.0 |
| DEPENDS_ON | Service → Service | static | 0.7-0.95 |
| CAN_RAISE | Endpoint → FailureMode | static + OpenAPI | 0.7-1.0 |
| MANIFESTS_IN | FailureMode → Endpoint | derived | same as inverse |
| DETECTED_BY | FailureMode → TestInvariant | test mining | 1.0 |
| ASSERTS | TestInvariant → FailureMode | test mining | 1.0 |
| HAS_REMEDIATION | FailureMode → RemediationAction | static + LLM-verified | 0.6-1.0 |
| REMEDIATES | RemediationAction → FailureMode | (inverse) | same as inverse |
| CALLED_BY | ExternalCall → Function | static | 1.0 |

### 4.4 File layout on disk

```
graph/
├── schema-version                      # single file, content: "1"
├── services/
│   └── payments-api.yaml               # one file per Service
├── apis/
│   └── payments-api/
│       └── v1.yaml                     # <namespace>/<name>
├── endpoints/
│   └── payments-api/
│       ├── POST__charge.yaml           # slash → __
│       └── GET__charge_{id}.yaml       # braces preserved; path-param marker
├── functions/
│   └── payments-api/
│       ├── payments.api.charge.charge_card.yaml
│       └── ...
├── external_calls/
│   └── payments-api/
│       └── stripe_charge_create.yaml
├── failure_modes/
│   └── payments-api/
│       ├── charge_stripe_timeout.yaml
│       └── charge_invalid_input.yaml
├── test_invariants/
│   └── payments-api/
│       └── test_charge_handles_stripe_timeout.yaml
├── remediation_actions/
│   └── payments-api/
│       └── retry_with_backoff.yaml
└── derived/                            # JSONL, auto-regenerated
    ├── call_graph.jsonl
    └── module_deps.jsonl
```

**Rules**:
- One entity per file. No exceptions.
- Filename = `spec.identity`-derived slug, URL-safe.
- `namespace` directory = owning service.
- `derived/*.jsonl` is gitignored OR committed as `*.jsonl.gz` — configurable per repo, default committed.

### 4.5 Canonicalization rules (JCS-equivalent for YAML)

Every write goes through `aegis_kg_schema.canonicalize()`:

1. Sort top-level keys in envelope order: `apiVersion, kind, metadata, spec, status`
2. Sort `metadata.*` alphabetically
3. Sort `spec.*` alphabetically within each sub-object
4. Sort `spec.relations[]` by `(type, target)` tuple
5. Sort `metadata.labels` and `metadata.annotations` alphabetically
6. Drop keys with null/empty values (Grafana root-cause fix)
7. Use double-quoted strings; preserve multiline with `|` for descriptions
8. Trailing newline, no trailing whitespace, LF line endings
9. UTF-8 encoding

**Validation**: `aegis-kg-schema` ships a `check-canonical` CLI that fails on any non-canonical file. Runs in pre-commit.

---

## 5. Extractor Architecture

### 5.1 Pipeline stages

```
┌─────────────────┐
│ 1. Discovery    │  Find services, frameworks, specs in repo
└────────┬────────┘
         │
┌────────▼────────┐
│ 2. Static AST   │  Parse Python, build module/function graph
└────────┬────────┘
         │
┌────────▼────────┐
│ 3. Call graph   │  Scalpel + PyCG; FQN resolution
└────────┬────────┘
         │
┌────────▼────────┐
│ 4. External     │  Detect HTTP/gRPC/DB client calls via FQN pattern
│    call detect  │
└────────┬────────┘
         │
┌────────▼────────┐
│ 5. Framework    │  Framework adapter extracts routes, middleware
│    adapter      │
└────────┬────────┘
         │
┌────────▼────────┐
│ 6. OpenAPI      │  Ingest spec; map operationId → handler
│    ingest       │
└────────┬────────┘
         │
┌────────▼────────┐
│ 7. CUE ingest   │  Parse CUE; constraints → ValidationFailureMode
└────────┬────────┘
         │
┌────────▼────────┐
│ 8. Test mining  │  pytest.raises, @patch, ast.Assert
└────────┬────────┘
         │
┌────────▼────────┐
│ 9. LLM verify   │  Optional test-name → scenario (CoVe-gated)
└────────┬────────┘
         │
┌────────▼────────┐
│10. Failure      │  Synthesize FailureMode nodes from stages 4-9
│    synthesis    │
└────────┬────────┘
         │
┌────────▼────────┐
│11. Canonical    │  Write graph/ as YAML
│    emit         │
└─────────────────┘
```

**Graceful degradation**: stages 5, 6, 7, 9 are optional. If a stage fails or inputs are missing, it's skipped with a warning. Downstream stages that depend on missing data mark affected nodes with `confidence: low` and `gap: <reason>`.

### 5.2 Framework adapter interface

```python
# aegis_kg_extractor/adapters/base.py

from abc import ABC, abstractmethod
from pathlib import Path
from typing import Iterator

class FrameworkAdapter(ABC):
    """Adapter for framework-specific extraction."""

    name: str  # "fastapi", "flask", "django", "aiohttp"

    @classmethod
    @abstractmethod
    def detect(cls, repo_root: Path) -> bool:
        """Return True if this adapter can handle the repo."""

    @abstractmethod
    def extract_endpoints(self, repo_root: Path) -> Iterator["EndpointSeed"]:
        """Yield endpoint seeds: (method, route, handler_fqn, metadata)."""

    @abstractmethod
    def extract_middleware(self, repo_root: Path) -> Iterator["MiddlewareSeed"]:
        """Yield middleware/decorator seeds that affect error behavior."""

    def extract_error_handlers(self, repo_root: Path) -> Iterator["ErrorHandlerSeed"]:
        """Yield registered error handlers. Optional."""
        return iter([])
```

**MVP ships**: `FastAPIAdapter`. That's it.

**Fallback**: `GenericPythonAdapter` — no route extraction, code graph only. Used when no framework detected.

### 5.3 External call detection (FQN pattern match)

Detect via AST pattern matching on call expressions. Match table:

| Library | FQN patterns | Emitted `call.protocol` |
|---|---|---|
| requests | `requests.{get,post,put,delete,patch,request}` | http |
| httpx | `httpx.{Client,AsyncClient}.{get,post,...,request}` | http |
| aiohttp | `aiohttp.ClientSession.{get,post,...,request}` | http |
| urllib3 | `urllib3.PoolManager.request` | http |
| grpc | `*_pb2_grpc.*Stub` attribute access followed by call | grpc |
| sqlalchemy | `Session.execute`, `Engine.connect` | sql |
| pymongo | `MongoClient.*` methods | mongo |
| redis | `Redis.*`, `StrictRedis.*` | redis |
| kafka | `KafkaProducer.send`, `KafkaConsumer.poll` | kafka |
| confluent_kafka | `Producer.produce`, `Consumer.poll` | kafka |
| boto3 | `boto3.client(...)` or `boto3.resource(...)` → method call | aws |

**Target hint resolution**:
1. If URL is a string literal in the call, capture it verbatim.
2. If URL is a constant imported from config (single assignment), follow the import and capture.
3. If URL is computed or passed as parameter, mark `confidence: low` and `target_hint: "<unresolved>"`.

Stage 9 (LLM verification) can be enabled to resolve low-confidence target hints by reading surrounding context — ship disabled by default.

### 5.4 Test mining

**`pytest.raises` extraction** (deterministic, confidence=1.0):

```python
# Matches:
#   with pytest.raises(SomeException):
#   with pytest.raises(SomeException, match="pattern"):
#   @pytest.mark.xfail(raises=SomeException)

class PytestRaisesVisitor(ast.NodeVisitor):
    def visit_With(self, node):
        for item in node.items:
            if is_pytest_raises_call(item.context_expr):
                yield ExceptionAssertion(
                    exception_class=extract_exception_class(item),
                    test_function=find_enclosing_function(node),
                )
```

**`unittest.mock.patch` extraction** (deterministic):

```python
# Matches @patch("pkg.mod.Thing") decorators and with patch(...) blocks
# Emits: (test_function, patched_target_fqn, side_effect_if_specified)
```

**`ast.Assert` extraction** (deterministic):

```python
# Captures assertion AST verbatim as Invariant
# Classifies comparator: ==, <, >, <=, >=, in, not in, isinstance
# Extracts call chain on LHS → links to code symbol
```

**Test-name → scenario** (LLM-verified, optional):

Enabled via `--llm-verify`. Prompt emits `{scenario, preconditions, expected_outcome}` Pydantic object. Verifier: every symbol in `expected_outcome` must appear in test body AST. Reject on mismatch, drop with warning.

### 5.5 OpenAPI ingest

Use `prance.ResolvingParser` with `openapi-spec-validator` backend. Extract:

1. Each `paths.<path>.<method>` → `Endpoint` node
2. `operationId` → `Endpoint.spec.descriptive.operation_id`
3. Match `operation_id` against discovered function FQNs (exact, then fuzzy via `rapidfuzz` ≥90)
4. Each `responses.{4xx,5xx}` → `FailureMode` with `http_status` and `response_schema_ref`
5. Each `requestBody.content.*.schema` → `Schema` node (Phase 2 expands; MVP just references)

**Missing `operationId`**: fallback to `{method}_{path-slugified}`, set `confidence: heuristic`.

**Unresolved handler**: emit `Endpoint` with `handler_fqn: null`, mark `gap: unresolved_handler`.

### 5.6 CUE ingest

Shell out to `cue export --out openapi schema.cue` — feed output through OpenAPI pipeline.

Separately, parse CUE constraints via `cue vet` + JSON error output. Each constraint violation type → `ValidationFailureMode`:

```yaml
kind: FailureMode
metadata:
  name: invalid_amount_negative
spec:
  descriptive:
    category: validation_failure
    validation:
      field_path: "$.amount"
      constraint_kind: range
      constraint_repr: ">=0"
      source_cue_file: schemas/payment.cue
```

MVP handles: range constraints (`>=`, `<=`, `>`, `<`), regex constraints, disjunctions (enum), required fields, type mismatches.

### 5.7 Output: graph emission

Emit in two phases:

**Phase A** (in-memory): build a `GraphBuilder` that accumulates all nodes and edges, de-duplicates by `(kind, namespace, name)`, resolves cross-references.

**Phase B** (disk): canonicalize + write. Two-phase write — stage to `.graph.tmp/`, atomic rename to `graph/`. Only files that actually changed get committed (use per-file content hash comparison before rename).

---

## 6. Generator Architecture

### 6.1 Pipeline stages

```
┌─────────────────┐
│ 1. Load graph   │  Parse graph/ into in-memory networkx.DiGraph
└────────┬────────┘
         │
┌────────▼────────┐
│ 2. Score        │  runbook_worthiness per FailureMode
│    worthiness   │
└────────┬────────┘
         │
┌────────▼────────┐
│ 3. Match        │  VF2 archetype classification
│    archetype    │
└────────┬────────┘
         │
┌────────▼────────┐
│ 4. Context      │  Steiner tree + typed k-hop + PPR enrichment
│    extract      │
└────────┬────────┘
         │
┌────────▼────────┐
│ 5. Template     │  Jinja2 deterministic structure
│    render       │
└────────┬────────┘
         │
┌────────▼────────┐
│ 6. LLM prose    │  Opus fills prose slots; CoVe verifies
│    (optional)   │
└────────┬────────┘
         │
┌────────▼────────┐
│ 7. Confidence   │  Composite score
│    score        │
└────────┬────────┘
         │
┌────────▼────────┐
│ 8. Merkle hash  │  neighborhood_hash for stale detection
└────────┬────────┘
         │
┌────────▼────────┐
│ 9. Emit         │  YAML frontmatter + Markdown body
└─────────────────┘
```

### 6.2 Worthiness scoring (MVP simplified)

MVP uses a simplified formula — full formula in research §3 is Phase 2:

```python
def runbook_worthiness(failure_mode: Node, graph: nx.DiGraph) -> float:
    if not has_remediation(failure_mode, graph):
        return 0.0  # hard gate

    score = 0.0
    score += 0.4 * blast_radius_score(failure_mode, graph, max_hops=3)
    score += 0.3 * test_coverage_score(failure_mode, graph)
    score += 0.3 * (1.0 if has_explicit_category(failure_mode) else 0.5)
    return min(score, 1.0)
```

Threshold: `>=0.5` gets a runbook. Tune after Week 7.

### 6.3 Archetype classification (MVP: one archetype)

```python
DOWNSTREAM_DEPENDENCY_FAILURE_PATTERN = nx.DiGraph()
DOWNSTREAM_DEPENDENCY_FAILURE_PATTERN.add_node('e', kind='Endpoint')
DOWNSTREAM_DEPENDENCY_FAILURE_PATTERN.add_node('f', kind='FailureMode')
DOWNSTREAM_DEPENDENCY_FAILURE_PATTERN.add_node('x', kind='ExternalCall')
DOWNSTREAM_DEPENDENCY_FAILURE_PATTERN.add_edge('e', 'f', rel='CAN_RAISE')
DOWNSTREAM_DEPENDENCY_FAILURE_PATTERN.add_edge('e', 'x', rel='MAKES_CALL')  # derived
DOWNSTREAM_DEPENDENCY_FAILURE_PATTERN.add_edge('f', 'x', rel='CAUSED_BY')

def classify_archetype(failure_mode: Node, graph: nx.DiGraph) -> Optional[str]:
    gm = iso.DiGraphMatcher(
        graph, DOWNSTREAM_DEPENDENCY_FAILURE_PATTERN,
        node_match=by_kind,
        edge_match=by_rel,
    )
    if any(gm.subgraph_isomorphisms_iter()):
        return "downstream_dependency_failure"
    return None
```

Failure modes that don't match any archetype are skipped in MVP (no runbook). Phase 2 adds fallback to generic archetype.

### 6.4 Context extraction (Steiner tree)

```python
def extract_context(failure_mode: Node, graph: nx.DiGraph,
                    token_budget: int = 2000) -> ContextSubgraph:
    terminals = {
        failure_mode.id,
        *[n.id for n in get_remediations(failure_mode, graph)],
        *[n.id for n in get_detecting_tests(failure_mode, graph)],
        owning_service(failure_mode, graph).id,
    }
    # Steiner tree produces minimal connecting scaffold
    steiner = nx.algorithms.approximation.steiner_tree(
        graph.to_undirected(), terminals, method='mehlhorn'
    )
    # Enrich within budget using typed k-hop
    expanded = typed_k_hop_expand(steiner, graph, budget=token_budget)
    return serialize_as_typed_context(expanded)
```

### 6.5 Template: downstream dependency failure

`templates/downstream_dependency_failure.md.j2`:

```markdown
# Runbook: {{ failure_mode.descriptive.summary }}

## Metadata
- **Archetype**: downstream_dependency_failure
- **Service**: {{ service.identity["service.name"] }}
- **Endpoint**: {{ endpoint.spec.identity["http.method"] }} {{ endpoint.spec.identity["http.route"] }}
- **Downstream dependency**: {{ external_call.identity["call.target_hint"] }}
- **Protocol**: {{ external_call.identity["call.protocol"] }}
- **Confidence**: {{ confidence.total }} (provenance: {{ confidence.breakdown }})

## Trigger Signature
This runbook is triggered when ANY of the following match an incident:

**Lexical**:
{% for kw in failure_mode.trigger_signature.lexical %}
- `{{ kw }}`
{% endfor %}

**Structured**:
- Service: `{{ failure_mode.trigger_signature.structured.service }}`
- Endpoint: `{{ failure_mode.trigger_signature.structured.endpoint }}`
- Exception: `{{ failure_mode.trigger_signature.structured.exception }}`

## Blast Radius
Services potentially affected (via DEPENDS_ON chains, 3 hops):
{% for svc in blast_radius %}
- {{ svc.name }} ({{ svc.relation }})
{% endfor %}

## Why This Happens
{{ llm_prose.why_this_happens }}  <!-- LLM-generated, verified -->

## Diagnosis Steps

{% for invariant in detecting_invariants %}
{{ loop.index }}. **Verify**: {{ invariant.scenario }}
   - Test reference: `{{ invariant.test_function_fqn }}`
   - Expected exception on failure: `{{ invariant.expected_exception }}`
{% endfor %}

{{ loop.index + 1 }}. **Check downstream**: confirm `{{ external_call.target_hint }}` reachability
   - Suggested query: `curl -v {{ external_call.target_hint }}/health` (if applicable)

## Remediation Steps

{% for action in remediation_actions %}
{{ loop.index }}. **{{ action.summary }}**
   - Action type: `{{ action.action_type }}`
   - Code reference: `{{ action.code_reference.symbol }}` (lines {{ action.code_reference.line_range[0] }}-{{ action.code_reference.line_range[1] }})
{% endfor %}

## Rollback Steps
1. Revert the change made in the remediation step above.
2. Restore the last known-good deployment via standard rollback procedure.

## Escalation Guidance
{{ llm_prose.escalation_guidance }}  <!-- LLM-generated, verified -->

- **Owner**: {{ service.descriptive.owner.email }}
- **On-call runbook**: <link to service on-call doc>

## Provenance
- **Graph commit**: `{{ provenance.commit_sha }}`
- **Extractor version**: `{{ provenance.extractor_version }}`
- **Generator version**: `{{ provenance.generator_version }}`
- **Neighborhood hash**: `{{ provenance.neighborhood_hash }}`
- **Model**: `{{ provenance.model }}` (prose sections only)
- **Generated at**: `{{ provenance.generated_at }}`
```

### 6.6 LLM prose with CoVe

**Prose slots** (only these): `why_this_happens`, `escalation_guidance`. Nothing else is LLM-generated.

**Prompt shape**:
```
You are writing a single paragraph for a production runbook. You have a failure mode context (below) and a target prose slot. Write 2-4 sentences for the target slot. You may ONLY use facts present in the context. Do NOT invent service names, tool names, or commands.

Context:
<typed-JSON context from Steiner tree>

Target slot: why_this_happens
```

**CoVe verification** (independent calls, no shared context):

1. Draft prose (Opus, temperature=0.2)
2. For each noun phrase in the draft, fire an independent verification call: "Is `<noun phrase>` present in the following context? Answer yes/no with quote." (Sonnet)
3. If any noun phrase is NOT verifiable, retry draft with stricter prompt
4. After 2 retries, fall back to template-only (`llm_prose.why_this_happens = "[Prose generation disabled: verification failed]"`)

Log every CoVe verification to `.aegis-kg-cache/cove-log.jsonl` for audit.

### 6.7 Confidence scoring (MVP simplified)

```python
@dataclass
class RunbookConfidence:
    total: float
    breakdown: dict[str, float]

def score_confidence(runbook: Runbook, graph: nx.DiGraph) -> RunbookConfidence:
    coverage = fraction_of_template_slots_filled_from_graph(runbook)
    # fraction of required fields that came from graph, not LLM, not default
    llm_uncertainty = 0.0 if not runbook.used_llm else 0.2
    # placeholder; Phase 2 uses token entropy
    graph_freshness = math.exp(-age_days(runbook) / 30)
    total = 0.5 * coverage + 0.3 * graph_freshness + 0.2 * (1 - llm_uncertainty)
    return RunbookConfidence(total=total, breakdown={
        "coverage": coverage,
        "graph_freshness": graph_freshness,
        "llm_uncertainty": llm_uncertainty,
    })
```

### 6.8 Merkle hash for staleness

```python
def neighborhood_hash(failure_mode_id: str, graph: nx.DiGraph, radius: int = 2) -> str:
    neighborhood = nx.ego_graph(graph, failure_mode_id, radius=radius)
    triples = sorted(
        (node_hash(n), rel, node_hash(t))
        for n, t, rel in neighborhood.edges(data='rel')
    )
    return hashlib.sha256(json.dumps(triples).encode()).hexdigest()
```

### 6.9 Output: runbook file layout

```
runbooks/
├── schema-version                  # mirrors graph/schema-version
├── payments-api/
│   ├── charge_stripe_timeout.md    # YAML frontmatter + Markdown body
│   └── ...
└── _index.jsonl                    # for retrieval indexing
                                    # one line per runbook: {id, path, tags, hash}
```

---

## 7. Retrieval MCP Server Architecture

### 7.1 MCP tools exposed

| Tool name | Input | Output | Use case |
|---|---|---|---|
| `retrieve_runbook` | incident description (text), optional structured fields (service, endpoint, exception) | ranked list of runbooks with scores | AEGIS Strategist primary retrieval |
| `search_failure_modes` | free-text query | ranked list of failure modes | AEGIS diagnosis / manual lookup |
| `explain_runbook` | runbook_id | full runbook + graph neighborhood + provenance | dashboard detail view / AEGIS RCA |
| `list_services` | — | all services in graph | AEGIS init / service discovery |
| `get_graph_node` | node_id | single node YAML + immediate neighbors | AEGIS deep-dive |

### 7.2 Retrieval pipeline

```
incident (text + structured)
    │
    ├──► BM25 over trigger_signature.lexical
    │        ↓
    ├──► dense HNSW over trigger_signature.embedding_text
    │        ↓
    │    hybrid merge (reciprocal rank fusion)
    │        ↓
    │    STRUCTURED PRE-FILTER ◄─── service overlap required
    │        ↓                      (prevents auth→payments mismatch)
    └──► cross-encoder rerank (ms-marco-MiniLM-L-6-v2)
             ↓
         top-k results with scores
```

**Structured pre-filter is mandatory.** Even if lexical/dense scores are high, reject runbooks whose `trigger_signature.structured.service` does not match the incident's service. This is the "never return a payments runbook for an auth incident" safety rail.

### 7.3 Index construction (at MCP startup)

```python
class RetrievalIndex:
    def __init__(self, graph_dir: Path, runbooks_dir: Path):
        self.runbooks = load_runbooks(runbooks_dir)
        self.bm25 = self._build_bm25([r.trigger_signature.lexical for r in self.runbooks])
        self.hnsw = self._build_hnsw([r.trigger_signature.embedding_text for r in self.runbooks])
        self.reranker = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')
        self.graph = load_graph(graph_dir)

    def retrieve(self, query: str, structured: dict, k: int = 5) -> list[RunbookResult]:
        bm25_hits = self.bm25.get_top_n(query, n=50)
        hnsw_hits = self.hnsw.query(encode(query), k=50)
        fused = reciprocal_rank_fusion(bm25_hits, hnsw_hits, k_rrf=60)
        filtered = self._structured_prefilter(fused, structured)
        reranked = self.reranker.rank(query, filtered[:20])
        return reranked[:k]
```

**Embeddings**: `BAAI/bge-small-en-v1.5` (384-dim, fast, CPU-friendly). Pre-computed at index build, not at query time.

### 7.4 MCP server transport

**MVP**: stdio transport only. Matches how AEGIS will spawn it as a subprocess.
**Post-MVP**: SSE transport for cross-process deployment on Gaia/CPSR — defer auth/deploy to platform team.

### 7.5 Index refresh strategy

Indices rebuild on startup. Post-MVP: watch filesystem for runbook changes, incrementally rebuild. MVP sidesteps this — MCP server restarts on every graph/runbook regeneration.

### 7.6 Index build time budget

For a 12-service / 100+ API graph with ~500 runbooks:
- Graph load: ~2s
- BM25 build: <1s
- HNSW build + embedding: ~30s (one-time)
- Total cold start: <60s acceptable

---

## 8. Review Workflow

### 8.1 Dual-surface principle

**Dashboard drives. PR gates.** Dashboard is the working surface for triage; PR is the merge gate for runbook integration into AEGIS's consumed corpus.

### 8.2 Dashboard (single-file HTML)

`dashboard/index.html` — single file, no build step. Loads graph + runbooks from static JSON exports produced by generator.

**Layout**:
- Left pane: triage queue, ordered by `(1 - confidence) * centrality`, filterable by archetype / service / status
- Center pane: runbook markdown preview (marked.js render)
- Right pane: graph neighborhood visualization (Cytoscape.js, 2-hop by default, expandable)
- Bottom pane: provenance + "explain this generation" breakdown

**Keyboard shortcuts**:
- `J`/`K`: next / previous item
- `A`: approve (sets `status.review = approved` in YAML)
- `R`: reject (sets `status.review = rejected`)
- `E`: open in inline editor
- `O`: open PR for this runbook
- `Space`: select for bulk op
- `?`: show shortcuts

**What the dashboard does NOT do in MVP**:
- Server-side state (it's single-HTML)
- User auth (it's per-user local)
- Edit persistence beyond emitting an edit patch to download

### 8.3 PR workflow

Per-runbook PRs via GitHub Actions bot. Each PR:
- Title: `[aegis-kg] Runbook: <failure_mode_name> (confidence: <score>)`
- Body: auto-generated from template with confidence breakdown, graph diff summary, provenance
- Labels: `aegis-kg`, `runbook`, `confidence-high|medium|low`
- Assignees: CODEOWNERS of the failure_mode's service
- Required checks: `aegis-kg/schema-check`, `aegis-kg/runbook-lint`, `aegis-kg/cove-verify`

### 8.4 Rate limits (anti-flood)

- Max 5 bot PRs opened per hour
- Max 20 open bot PRs at any time
- Max 15 runbooks per reviewer per day (dashboard warns when exceeded)

---

## 9. CI / CD Integration

### 9.1 Three stages

**Stage A — Pre-commit (local, <3s)**:
- JCS canonicalization check on changed YAML files
- Schema validation against pinned schema version
- Lint for required provenance fields

**Stage B — PR check (1-5 min)**:
- Diff-aware extraction against merge-base
- DeepDiff graph-plan output posted as PR comment
- Stale-runbook detection; list stale as PR comment
- Upload dashboard snapshot as workflow artifact

**Stage C — Nightly full regeneration**:
- Full extraction on HEAD
- Regenerate all runbooks
- Open "Nightly KG refresh" PR with all changes
- Rate-limit per §8.4

### 9.2 `aegis-kg-schema` compat check

Every downstream repo's CI runs:
```bash
aegis-kg-schema check-compat --graph graph/ --version $(cat graph/schema-version)
```
Fails if graph uses schema fields not in the pinned version range.

---

## 10. Cross-cutting Concerns

### 10.1 Provenance (enforced at schema layer)

Every node, every edge, every runbook carries provenance. Schema validation REJECTS any artifact missing:
- `source.extractor` (or `source.generator`)
- `source.commit_sha`
- `source.path` (where applicable)

### 10.2 Graceful degradation

At every stage boundary, define: if upstream stage produced partial data, what does downstream stage do?

| Scenario | Behavior |
|---|---|
| No OpenAPI spec | Extractor skips stage 6; endpoints come from framework adapter only; mark gap |
| No pytest suite | Extractor skips stage 8; FailureModes come from OpenAPI 4xx/5xx only |
| LLM unavailable | Generator skips stage 6; runbooks ship with `llm_prose_pending: true` |
| Scalpel fails on a module | Skip that module; log; continue with partial call graph |
| External call unresolved | Emit `ExternalCall` with `confidence: low`, `target_hint: unresolved` |

### 10.3 Logging and observability

Structured JSONL logs per stage with `correlation_id` linking a single extractor run. Logged: stage name, inputs, outputs (counts, not content), errors, duration. Output to `.aegis-kg-logs/<run_id>.jsonl`.

### 10.4 Caching

`.aegis-kg-cache/`:
- `file_hashes.json` — per-file sha256 for change detection
- `ast_trees/<file_hash>.pkl` — parsed AST cache
- `call_graph/<commit_sha>.json` — Scalpel output cache
- `cove-log.jsonl` — LLM verification audit
- `embeddings/<model>/<text_hash>.npy` — embedding cache

Cache invalidation: by content hash, never by mtime.

### 10.5 Multi-repo CI contract

Schema repo CI runs:
1. Schema self-check (CUE vet + round-trip Pydantic)
2. Integration test: extract a fixture repo, generate runbooks, run MCP retrieval — end-to-end with current HEAD of all downstream repos
3. On pass, tag and publish to internal PyPI

Downstream repo CI runs:
1. Pin check: schema_version matches pyproject declaration
2. Integration test against pinned schema version
3. On schema major-version bump, require explicit migration PR

---

## 11. Week-by-week Exit Criteria

### Week 1: `aegis-kg-schema` skeleton + `aegis-kg-extractor` discovery
- Four repos initialized, pyproject, CI skeleton
- Schema v1 authored in CUE, Pydantic models generated
- `aegis-kg-extractor discover` CLI walks a repo, emits `discovered.json`
- CI green on all four repos (even if tests are trivial)
- **Exit**: `aegis-kg-extract --repo examples/pilot --dry-run` lists detected framework + files

### Week 2: Extractor core (stages 1-4, 5, 11)
- AST walk produces Function, Module nodes
- Scalpel/PyCG integration; call graph JSONL
- External call detection
- FastAPI adapter extracts endpoints
- Canonical YAML emission
- **Exit**: `aegis-kg-extract --repo examples/pilot` produces valid `graph/` with Services, APIs, Endpoints, Functions, ExternalCalls, Module imports

### Week 3: Generator skeleton + stub inputs
- Generator loads graph into networkx
- Worthiness scoring, VF2 archetype matcher for downstream_dependency_failure
- Template rendering (no LLM)
- Confidence scoring (coverage only)
- Works against **hand-written** graph YAML (not extractor output)
- **Exit**: `aegis-kg-generate --graph fixtures/hand-written-graph` produces valid runbook for one hand-written failure mode

### Week 4: OpenAPI + CUE ingest; test mining
- prance OpenAPI ingest; operationId match; FailureMode from 4xx/5xx
- CUE constraint ingest → ValidationFailureMode
- pytest.raises, @patch, ast.Assert extraction
- **Exit**: Extractor run on pilot produces ≥3 FailureMode nodes, ≥5 TestInvariant nodes

### Week 5: LLM prose + CoVe; retrieval MCP skeleton
- LLM prose generation for why_this_happens, escalation_guidance
- CoVe verification with independent calls
- Retrieval MCP server with stdio transport, `retrieve_runbook` + `search_failure_modes` tools
- BM25 + HNSW + cross-encoder rerank
- **Exit**: End-to-end demo with hand-written graph: AEGIS client calls MCP, retrieves runbook

### Week 6: Walking-skeleton demo + dashboard v0
- Dashboard HTML with triage queue, keyboard shortcuts
- GH Actions PR check stage
- Stale-runbook detection via Merkle hash
- Polish: CLI UX, error messages, docs
- **Demo to Kunal + CDS team**: full flow on hand-written graph
- **Exit**: demo passes; at least one runbook merged via PR by a human SME

### Week 7: Wire real extractor output end-to-end
- Extractor output → generator (not hand-written)
- Generator output → retrieval MCP
- Fix the real-data issues that surface (there will be many)
- Run regeneration nightly on pilot repo for 3 days
- Measure: retrieval precision@5 on 5 synthetic incidents
- **Exit**: on 5 synthetic incidents, AEGIS retrieves right service ≥80%, useful runbook ≥60%

---

## 12. Risk Register (top 5 for MVP)

| # | Risk | Severity | Mitigation |
|---|---|---|---|
| 1 | Scalpel/PyCG call graph recall <85% on pilot repo | High | Week 1: benchmark call-graph recall vs hand-labeled ground truth. If <70%, switch to Jarvis or ship with coverage gap explicit. |
| 2 | Pilot repo has no OpenAPI spec | High | Check before committing pilot selection in Week 1. If no OpenAPI, extractor runs without stage 6 — validate graceful degradation works end-to-end. |
| 3 | LLM prose CoVe verification rejects >50% of drafts | Medium | Week 5: tune verification prompts, relax noun-phrase matching (allow partial lexical overlap). If still failing, ship MVP with prose disabled and flag in Phase 2 spec. |
| 4 | MCP server auth/deploy on Gaia/CPSR unresolved | Medium | MVP uses stdio locally. Open ticket with platform team Week 1. If production auth is a blocker for Week 7, ship as local-only tool and negotiate production path separately. |
| 5 | Reviewer fatigue — SMEs refuse to review bot PRs | High | Dashboard v0 must ship Week 6 (keyboard-first triage). Rate-limit PRs per §8.4 from day one. If Week 7 measures reviewer rejection rate >30%, redesign before Phase 2. |

---

## 13. Success Metrics (measured at end of Week 7)

| Metric | Target | How measured |
|---|---|---|
| Extraction wall-clock on pilot | <5 min cold, <1 min warm | CI logs |
| Graph node count on pilot | proportional to service (e.g., 200-2000 for mid-size service) | generator pre-flight output |
| Runbook generation rate | ≥5 runbooks per service with suitable failure modes | generator stats |
| Retrieval precision@5 on synthetic incidents | ≥80% correct service; ≥60% useful runbook | evaluation harness with 5 labeled synthetic incidents |
| CoVe verification pass rate | ≥70% (drafts accepted without retry) | cove-log.jsonl |
| Nightly regeneration stability | 3 consecutive nights with zero spurious diffs | git log on nightly PRs |
| SME approval rate (dashboard) | ≥60% approve or edit+approve | dashboard telemetry |

**If any metric fails**: document, add to Phase 2 risk register, do not scale.

---

## 14. Out-of-Scope (reminder)

See §1.4. Not adding these in MVP. Phase 2 spec will cover them.

---

## 15. Open Questions

| # | Question | Owner | Resolution deadline |
|---|---|---|---|
| 1 | Which pilot service? | Jey + Ashish | End of Week 1 |
| 2 | Production MCP auth on Gaia/CPSR | Platform team | End of Week 4 |
| 3 | Does Copilot running from VDI support Opus/Sonnet tool-use with JSON schema? | Jey | End of Week 2 |
| 4 | Where does the graph repo live — one-per-service or aggregated? | Jey + Kunal | End of Week 3 |
| 5 | Who owns nightly PR review rota? | Jey | End of Week 6 |

---

## 16. Appendix A: Schema version policy

- Current: `aegis.dev/v1alpha1`, schemaVersion: 1
- Bumps: minor schema additions → schemaVersion++; breaking → apiVersion alpha→beta→v1
- Migrations: Alembic-style under `migrations/` in schema repo; each consumer runs `aegis-kg-schema migrate --check` in CI
- Deprecation: mark `"deprecated_in": N, "removed_in": N+2`; warn but don't fail for 2 versions

## 17. Appendix B: Dependencies (pinned ranges)

| Package | Range | Purpose |
|---|---|---|
| pydantic | >=2.6,<3 | Schema validation |
| pyyaml | >=6.0,<7 | YAML I/O |
| rfc8785 | >=0.1 | JCS canonicalization |
| networkx | >=3.2,<4 | In-memory graph |
| scalpel | >=1.0beta | Python static analysis |
| prance | >=23.6 | OpenAPI parsing |
| cuelang-py-binding | TBD | CUE ingest (may require shell-out to `cue` binary) |
| rapidfuzz | >=3 | Fuzzy FQN matching |
| rank-bm25 | >=0.2 | BM25 retrieval |
| hnswlib | >=0.8 | Dense vector index |
| sentence-transformers | >=2.5 | BGE embeddings |
| mcp | >=0.9 | MCP server SDK |
| jinja2 | >=3.1 | Runbook templates |
| deepdiff | >=6.7 | Graph diff |
| anthropic | >=0.25 | LLM API |

## 18. Appendix C: Glossary

- **Archetype**: a graph subgraph pattern that describes a class of failure (e.g., downstream_dependency_failure)
- **CoVe**: Chain-of-Verification; independent LLM calls verify claims in draft output
- **JCS**: JSON Canonicalization Scheme (RFC 8785); deterministic JSON serialization
- **Materialized view**: pre-computed result stored to disk, vs computed on-demand
- **MCP**: Model Context Protocol; Anthropic's protocol for LLM↔tool communication
- **Merkle hash**: content-addressed hash of a subgraph used for staleness detection
- **Neighborhood**: the subgraph within K hops of a given node
- **PPR**: Personalized PageRank; random-walk-based node importance scoring
- **Provenance**: metadata tracing a fact to its source artifact and commit
- **Runbook-worthiness**: score determining whether a failure mode deserves a runbook
- **Steiner tree**: minimal subgraph connecting a set of terminal nodes
- **VF2**: subgraph isomorphism algorithm; NetworkX built-in

---

*End of spec. Next: see `AEGIS-KG-VIBE-GUIDE.md` for Copilot session playbook.*
