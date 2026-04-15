# AEGIS — Day 8+ Spec: Pre-warm, Retrieve, Chat

> **Audience**: GitHub Copilot (IDE) + Jey (review/clarify).
> **Predecessor**: 8-day AEGIS MVP demo build (end of Day 7 inventory).
> **Status**: This is an *additive* spec. It does not break or rewrite Day 1–7 work. Existing CLI green path (`test_pipeline.py`) must keep passing throughout.

---

## 0. Context

End of Day 7 leaves AEGIS with a working CLI pipeline (~111s end-to-end), a wired FastAPI+SSE+Alpine.js frontend, three agents, demo-mode mocking, and zero stubs. The single biggest gap for a *live demo* is not functionality — it's **dead air**. Most of the 111s is consumed by serial MCP gather + LLM runbook generation. Audiences disengage during silent gaps.

This spec adds five modules that collectively:
1. **Pre-warm** known intelligence into a vector cache so gather phase becomes sub-second for matched incidents.
2. **Retrieve** runbooks and domain knowledge by semantic similarity to inject into Strategist context.
3. **Constrain** Splunk query generation to vetted patterns first; LLM-ad-hoc only as fallback.
4. **Capture** the incident from a chatbox with LLM-assisted structured field extraction.
5. **Persist** all of the above on disk so a Windows scheduler can populate the cache without the FastAPI app being aware.

The architectural principle throughout: **deterministic before probabilistic**. Cache + patterns + retrieved runbooks short-circuit the LLM where possible. The LLM only sees a smaller, richer, pre-curated context.

---

## 1. Objective & Non-Goals

### Objective
Reduce demo-time gather phase from ~30–60s to <5s for the two seeded scenarios (upload failure, latency degradation), while making the Strategist's context dramatically richer through retrieved domain knowledge and runbooks.

### Non-Goals (explicitly out of scope for this spec)
- Runbook *authoring* UI / approval workflows
- Multi-user collaboration on the cache
- Cache invalidation across multiple AEGIS instances
- Production deployment of the prewarmer (it runs on Jey's VDI only)
- Replacing live MCP calls entirely (cache *augments*, never *replaces*)
- Domain MCP integration (we use the local KB folder as a stand-in)
- Embedding model fine-tuning
- Vector DB clustering, sharding, or HA

### Success Criteria
| # | Metric | Target |
|---|---|---|
| 1 | Gather phase latency for cached incidents | <5s (P95) |
| 2 | Strategist context size (tokens) | ≤ existing budget (12k) — retrieval *replaces*, doesn't *add* |
| 3 | Splunk query failure rate on demo scenarios | 0% (patterns must match) |
| 4 | Runbook retrieval recall@3 on seeded scenarios | 100% (deterministic test fixtures) |
| 5 | Chatbox → structured extraction accuracy on Aayushi's two scenarios | 100% (manual eval) |
| 6 | CLI green path (`test_pipeline.py`) | Still passes after every phase |

---

## 2. Architecture Overview

### 2.1 Component diagram (ASCII)

```
                   ┌─────────────────────────────────────────────┐
                   │           Jey's Windows VDI                 │
                   │                                             │
  ┌─────────────┐  │   ┌──────────────────┐                      │
  │  Windows    │──┼──▶│   prewarmer/     │  scheduled (10 min)  │
  │  Scheduler  │  │   │   runner.py      │                      │
  │  (Task      │  │   └────────┬─────────┘                      │
  │   Sched.)   │  │            │                                │
  └─────────────┘  │            ▼                                │
                   │   ┌──────────────────┐    ┌──────────────┐  │
                   │   │ MCP servers      │    │  domain_kb/  │  │
                   │   │ (Splunk, DT,     │    │  (markdown,  │  │
                   │   │  Confluence,     │    │   YAML)      │  │
                   │   │  Jira)           │    └──────┬───────┘  │
                   │   └────────┬─────────┘           │          │
                   │            │                     │          │
                   │            ▼                     ▼          │
                   │   ┌─────────────────────────────────────┐   │
                   │   │      vector_store/                  │   │
                   │   │  (ChromaDB embedded — file-backed)  │   │
                   │   │                                     │   │
                   │   │  collections:                       │   │
                   │   │   - splunk_results                  │   │
                   │   │   - domain_kb                       │   │
                   │   │   - runbooks                        │   │
                   │   │   - query_pattern_examples          │   │
                   │   └──────────────┬──────────────────────┘   │
                   │                  │                          │
                   │                  │ read at runtime          │
                   │                  ▼                          │
                   │   ┌─────────────────────────────────────┐   │
                   │   │       FastAPI (uvicorn)             │   │
                   │   │                                     │   │
                   │   │   ┌──────────┐    ┌─────────────┐   │   │
                   │   │   │ chatbox  │───▶│  intake/    │   │   │
                   │   │   │ (Alpine) │    │  extractor  │   │   │
                   │   │   └──────────┘    └──────┬──────┘   │   │
                   │   │                          │          │   │
                   │   │                          ▼          │   │
                   │   │   ┌──────────────────────────────┐  │   │
                   │   │   │      strategist (existing)   │  │   │
                   │   │   │   gather_intelligence() now  │  │   │
                   │   │   │   queries vector_store FIRST │  │   │
                   │   │   │   and consults              │  │   │
                   │   │   │   query_patterns/ for SPL    │  │   │
                   │   │   └──────────────┬───────────────┘  │   │
                   │   │                  │                  │   │
                   │   │                  ▼                  │   │
                   │   │     existing executor + agents      │   │
                   │   └─────────────────────────────────────┘   │
                   └─────────────────────────────────────────────┘
```

### 2.2 Data flow at incident time (cached path)

1. User opens chatbox in browser → types: *"upload failed for user 12345 on Foundation app, returning 500s for last 10 minutes"*
2. Frontend posts to `POST /api/intake/parse` — backend calls LLM to extract `IncidentContext` fields
3. Backend returns extracted fields → frontend shows editable preview → user clicks **Run Diagnosis**
4. Standard `POST /api/trigger` flow begins, but `gather_intelligence()` now:
   - Embeds the incident description
   - Queries `vector_store` collections (splunk_results, domain_kb, runbooks) with hybrid filter (semantic + `app_name` + freshness ≤ 30 min)
   - Consults `query_patterns/` for matching SPL templates
   - Fires live Splunk MCP call **in parallel** with cache lookup (freshness check)
   - Returns combined `list[ToolResponse]` — cache hits arrive in <500ms; live calls return when ready
5. Strategist `generate_runbook()` now sees: cached intelligence + retrieved runbooks + retrieved domain KB snippets + live MCP responses. Context is richer; LLM call duration is unchanged but quality is higher.
6. SSE event stream now includes `cache_hit`, `pattern_matched`, `runbook_retrieved` events — visible in UI as polish.

### 2.3 Data flow at incident time (cold path — fallback)

1. No matching pattern, no relevant cache entries.
2. Strategist falls back to existing Day 7 behavior (live MCP gather + ad-hoc LLM-generated SPL).
3. Pipeline still works. Slower, but not broken.

---

## 3. Module: `vector_store/`

### 3.1 Purpose
Single abstraction over the vector DB. Decouples downstream code from ChromaDB/Qdrant choice.

### 3.2 File structure
```
backend/
└── vector_store/
    ├── __init__.py             # Re-exports VectorStore
    ├── base.py                 # Abstract base class
    ├── chroma_backend.py       # ChromaDB embedded implementation (default)
    ├── qdrant_backend.py       # Qdrant adapter (stub for future)
    ├── embedder.py             # Embedding model wrapper
    └── collections.py          # Collection definitions + schemas
```

### 3.3 Backend choice
**Default: ChromaDB embedded** (file-backed, no Docker daemon, runs in-process).
- Path: `~/.aegis/vector_store/` (configurable via `settings.vector_store_path`)
- Persistence: ChromaDB's built-in DuckDB+Parquet backend
- Why not Qdrant: requires Docker daemon → unverified on VDI. Qdrant adapter stubbed for swap-in if confirmed.

### 3.4 Embedding model
**Default: `sentence-transformers/all-MiniLM-L6-v2`** (~80MB, runs on CPU, no GPU needed).
- Loaded once at FastAPI startup (lifespan hook)
- Dimension: 384
- Configurable via `settings.embedding_model_name` and `settings.embedding_model_path` (for offline/airgapped install)

> ⚠️ **Open question for Jey**: confirm whether `docviewer-be` exposes an `/embeddings` endpoint. If yes, prefer that over local model. Spec defaults to local; swap is a 30-line change in `embedder.py`.

### 3.5 Collections
| Collection | Document content | Metadata |
|---|---|---|
| `splunk_results` | concatenated raw event text (compressed) | `app_name`, `query_pattern_id`, `captured_at`, `result_count`, `time_window` |
| `domain_kb` | section text from KB markdown/YAML | `kb_path`, `app_name`, `section_title`, `doc_type` (one of: `runbook`, `architecture`, `failure_mode`, `glossary`) |
| `runbooks` | full runbook YAML serialized | `runbook_id`, `app_name`, `incident_signature`, `last_validated_at` |
| `query_pattern_examples` | example incident descriptions for each SPL pattern | `pattern_id`, `app_name` |

### 3.6 API
```python
class VectorStore(ABC):
    async def upsert(self, collection: str, docs: list[VectorDoc]) -> None: ...
    async def query(
        self,
        collection: str,
        text: str,
        k: int = 5,
        filters: dict | None = None,
        max_age_seconds: int | None = None,
    ) -> list[VectorHit]: ...
    async def count(self, collection: str) -> int: ...
    async def reset(self, collection: str) -> None: ...
```

```python
class VectorDoc(BaseModel):
    id: str  # stable hash of (source, content) — idempotent upsert
    text: str
    metadata: dict[str, Any]

class VectorHit(BaseModel):
    id: str
    text: str
    metadata: dict[str, Any]
    score: float  # cosine similarity, 0-1
    age_seconds: int  # derived from metadata.captured_at if present
```

### 3.7 Tests
- `tests/test_vector_store.py` — upsert + query roundtrip on each collection, with and without metadata filters, age filter behavior
- `tests/test_embedder.py` — embedding determinism, batch behavior, dimension check

---

## 4. Module: `prewarmer/`

### 4.1 Purpose
Standalone Windows-schedulable script that periodically pulls fresh intelligence from MCP servers + the domain KB folder and upserts to vector store. **Runs out-of-process from FastAPI.**

### 4.2 File structure
```
prewarmer/
├── __init__.py
├── runner.py               # Entry point — `python -m prewarmer.runner`
├── jobs.py                 # Job definitions (one per data source)
├── manifest.py             # Run manifest (timestamps, counts, errors)
├── schedule.bat            # Windows Task Scheduler hook
└── README.md               # Setup instructions for Windows Task Scheduler
```

### 4.3 Jobs
Each job is an async function returning `JobResult`. Runner executes them in parallel and writes manifest.

| Job | What it does | Frequency |
|---|---|---|
| `prewarm_splunk_patterns` | For each Splunk pattern in `query_patterns/` with a `prewarm: true` flag, render the SPL with default params and execute via Splunk MCP. Upsert results to `splunk_results` collection with metadata. | Every 10 min |
| `prewarm_dynatrace_signals` | Pull golden signals for each entity in `settings.dt_entity_names`. Upsert to `splunk_results` collection (yes, same collection — keeps Strategist gather code uniform; metadata.source distinguishes). | Every 10 min |
| `index_domain_kb` | Walk `domain_kb/` directory, parse each file, chunk by section, upsert to `domain_kb` collection. Skip files where on-disk mtime ≤ last indexed mtime. | Every run (cheap if no changes) |
| `index_runbooks` | Walk `domain_kb/runbooks/` (or wherever runbooks live), validate against `RunbookSchema`, upsert to `runbooks` collection. | Every run |
| `index_query_pattern_examples` | For each pattern in `query_patterns/`, upsert its example incident descriptions to `query_pattern_examples`. | On pattern file change |

### 4.4 Manifest
After each run, write `~/.aegis/prewarm_manifest.json`:
```json
{
  "last_run_at": "2026-04-15T09:30:00Z",
  "duration_seconds": 47.2,
  "jobs": {
    "prewarm_splunk_patterns": {"status": "ok", "upserted": 12, "errors": []},
    "prewarm_dynatrace_signals": {"status": "ok", "upserted": 4, "errors": []},
    "index_domain_kb": {"status": "ok", "upserted": 0, "skipped": 23, "errors": []},
    "index_runbooks": {"status": "ok", "upserted": 2, "errors": []},
    "index_query_pattern_examples": {"status": "ok", "upserted": 8, "errors": []}
  }
}
```

FastAPI reads this manifest on `/api/health` and exposes freshness in the UI.

### 4.5 Windows Task Scheduler integration
`schedule.bat` is a one-shot setup script:
```bat
@echo off
schtasks /Create /SC MINUTE /MO 10 /TN "AEGIS_Prewarm" ^
  /TR "C:\path\to\python.exe -m prewarmer.runner" /F
```
README explains how to install, view logs, disable.

### 4.6 Auth model
Prewarmer reuses `backend/mcp/client_pool.py` and shares `.token_cache.json` with FastAPI. ADFS device-code flow runs **only** during interactive setup; subsequent scheduled runs use the cached token. If token expires, the job for that MCP fails gracefully and is logged in the manifest — does not crash the runner.

### 4.7 Tests
- `tests/test_prewarmer_jobs.py` — each job runs against mock MCP pool, assert upsert calls
- `tests/test_prewarmer_manifest.py` — manifest read/write/merge

### 4.8 Risks specific to prewarmer
| Risk | Mitigation |
|---|---|
| Token expires between scheduled runs (no human at VDI to do device-code) | Detect expiry, write `auth_required: true` to manifest, surface in UI. Acceptable for demo (Jey is at VDI). |
| Prewarmer runs while FastAPI is querying — file lock contention | ChromaDB handles concurrent reads/writes via DuckDB. Verify with `tests/test_concurrent_access.py`. If contention occurs, gate via `fcntl`/`msvcrt` file lock around upserts. |
| Splunk MCP rate limits | Patterns marked `prewarm: true` should be a small set (≤ 10). Document in README. |

---

## 5. Module: `query_patterns/`

### 5.1 Purpose
A library of SPL templates keyed by incident signature. Strategist consults this **before** asking the LLM to generate ad-hoc SPL. Implements the **deterministic-before-probabilistic** principle at the query-formation layer.

### 5.2 File structure
```
backend/
└── query_patterns/
    ├── __init__.py
    ├── loader.py               # Loads YAML files into PatternRegistry
    ├── matcher.py              # Match incident to pattern (LLM + heuristic)
    ├── renderer.py             # Render SPL from template + params
    ├── schemas.py              # PatternSchema (Pydantic)
    └── patterns/               # One YAML per pattern
        ├── upload_failure_500.yaml
        ├── upload_latency_p99_breach.yaml
        ├── ingest_queue_backup.yaml
        └── ...                 # Add as Aayushi confirms scenarios
```

### 5.3 Pattern YAML schema
```yaml
pattern_id: upload_failure_500
description: Document upload returning HTTP 500 errors
trigger_keywords:
  - upload failed
  - upload error
  - 500 on upload
example_incidents:
  - "upload failed for user 12345 on Foundation app, returning 500s"
  - "documents not uploading, getting server errors"
applies_to_apps: [cds-store, foundation, document-ingest]
required_params:
  - app_name
  - time_window  # default: -15m
optional_params:
  - user_id
  - doc_id
spl_template: |
  index={splunk_index} app={app_name}
  ("upload" OR "POST /documents")
  (status=500 OR error)
  earliest={time_window}
  | head 100
prewarm: true
prewarm_params:
  app_name: cds-store
  time_window: -15m
confidence_floor: 0.65  # match below this → don't use, fall back to LLM
```

### 5.4 Matcher logic
```
def match(incident: IncidentContext, registry: PatternRegistry) -> MatchResult:
    # Step 1: hard filter by app_name (if pattern declares applies_to_apps)
    candidates = [p for p in registry.all() if incident.app_name in p.applies_to_apps or not p.applies_to_apps]

    # Step 2: vector search using query_pattern_examples collection
    hits = vector_store.query("query_pattern_examples", incident.description, k=5)
    candidates_by_score = {hit.metadata["pattern_id"]: hit.score for hit in hits}

    # Step 3: keyword overlap as tiebreaker
    for p in candidates:
        kw_score = keyword_overlap(incident.description, p.trigger_keywords)
        candidates_by_score[p.pattern_id] = (
            0.7 * candidates_by_score.get(p.pattern_id, 0)
            + 0.3 * kw_score
        )

    # Step 4: pick best, check floor
    best = max(candidates_by_score.items(), key=lambda x: x[1])
    if best[1] < pattern.confidence_floor:
        return MatchResult(matched=False, fallback_reason="below_confidence_floor")
    return MatchResult(matched=True, pattern_id=best[0], score=best[1])
```

### 5.5 Renderer
Uses Jinja2 with strict undefined behavior. Missing required params → raise `PatternRenderError`.

### 5.6 Integration with Strategist
In `strategist._build_intelligence_calls()`:
```python
match_result = pattern_matcher.match(incident, pattern_registry)
if match_result.matched:
    spl = pattern_renderer.render(match_result.pattern_id, incident.dict())
    calls.append(McpCall(server="splunk", method="search", params={"query": spl}))
    event_callback("pattern_matched", {"pattern_id": match_result.pattern_id, "score": match_result.score})
else:
    # Existing Day 7 behavior — let LLM generate SPL
    ...
```

### 5.7 Tests
- `tests/test_pattern_matcher.py` — Aayushi's two scenarios must match upload_failure_500 and upload_latency_p99_breach with score ≥ 0.7
- `tests/test_pattern_renderer.py` — required param missing → raises; all params present → valid SPL string
- `tests/test_pattern_loader.py` — invalid YAML rejected with clear error

---

## 6. Module: `runbook_lib/`

### 6.1 Purpose
Load runbooks from disk, embed them, retrieve at runtime. Inject top-K matched runbooks into Strategist context.

> Scope is **retrieval only**. Authoring/editing/approval workflows are out of scope.

### 6.2 File structure
```
backend/
└── runbook_lib/
    ├── __init__.py
    ├── loader.py           # Reads runbooks from domain_kb/runbooks/
    ├── retriever.py        # Vector retrieval + filter
    ├── injector.py         # Formats retrieved runbooks for LLM context
    └── schemas.py          # RunbookDoc (distinct from existing GeneratedRunbook)
```

### 6.3 RunbookDoc schema
```python
class RunbookDoc(BaseModel):
    runbook_id: str          # stable, e.g. "RB-CDS-001"
    title: str
    incident_signature: str  # human-readable description
    applies_to_apps: list[str]
    severity_levels: list[str] = ["P1", "P2", "P3"]
    steps: list[RunbookDocStep]
    last_validated_at: datetime
    source_path: str         # for debugging
```

> ⚠️ Note: This is **distinct** from the existing `GeneratedRunbook` (in `models/schemas.py`) which is what the Strategist *produces*. `RunbookDoc` is what we *retrieve as reference*. Do not conflate.

### 6.4 Loader
- Walks `domain_kb/runbooks/*.yaml`
- Validates each against `RunbookDoc` schema
- Skips invalid files with logged warning (does not crash)
- Returns `list[RunbookDoc]`

### 6.5 Retriever
```python
async def retrieve(
    incident: IncidentContext,
    k: int = 3,
) -> list[RunbookHit]:
    query_text = f"{incident.app_name} {incident.description}"
    hits = await vector_store.query(
        "runbooks",
        query_text,
        k=k,
        filters={"applies_to_apps": {"$contains": incident.app_name}},
    )
    return [RunbookHit.from_vector_hit(h) for h in hits]
```

### 6.6 Injector
Formats top-K runbooks into the Strategist's user message (under existing `max_context_tokens` budget). Format:
```
## Reference Runbooks (retrieved by similarity)

### RB-CDS-001 — Upload failure: HTTP 500 from cds-store
Last validated: 2026-04-10
Steps:
  1. Check cds-store pod restart count in last 30m...
  2. Inspect Splunk for upstream auth failures...
  ...
```

### 6.7 Strategist integration
In `generate_runbook()`, before calling LLM:
```python
retrieved = await runbook_retriever.retrieve(incident, k=3)
if retrieved:
    context_addition = runbook_injector.format(retrieved)
    user_message = inject_runbooks_section(user_message, context_addition)
    event_callback("runbook_retrieved", {"runbook_ids": [r.runbook_id for r in retrieved]})
```

### 6.8 Tests
- `tests/test_runbook_loader.py` — fixture with valid + invalid YAMLs; loader returns valid only, logs warnings
- `tests/test_runbook_retriever.py` — Aayushi's two scenarios must retrieve their corresponding runbooks at rank 1
- `tests/test_runbook_injector.py` — output respects token budget

---

## 7. Module: `domain_kb/` (loader + structure convention)

### 7.1 Purpose
The on-disk KB folder is the source of truth for everything domain-specific. Until Domain MCP exists, this is its file-system stand-in.

### 7.2 Convention
```
domain_kb/
├── apps/
│   ├── cds-store.md
│   ├── foundation.md
│   └── document-ingest.md
├── failure_modes/
│   ├── upload_failure.md
│   └── ingest_latency.md
├── architecture/
│   └── cds_overview.md
├── runbooks/
│   ├── RB-CDS-001-upload-500.yaml
│   └── RB-CDS-002-upload-latency.yaml
└── glossary.md
```

> ⚠️ **Open question for Jey**: confirm the actual structure of your `domain kb` folder. Spec assumes the above; if different, adjust `loader.py` patterns. **Do not silently rename existing files.**

### 7.3 Loader behavior
- Markdown files: chunk by `##` headers, each chunk = one `VectorDoc` in `domain_kb` collection
- YAML files in `runbooks/`: parsed by `runbook_lib.loader`
- All other files: skipped with debug log

### 7.4 Tests
- `tests/test_domain_kb_loader.py` — fixture KB → expected number of chunks per file

---

## 8. Frontend: chatbox + retrieval indicators

### 8.1 Goal
Replace the static dropdown form with a free-text chatbox + LLM-extracted structured preview, and add visual indicators for cache/pattern/runbook hits. Preserve all existing SSE wiring.

### 8.2 New UI sections (Alpine.js, in `frontend/index.html`)

#### 8.2.1 Chatbox panel (left, replaces existing form)
- Large `<textarea>` placeholder: *"Describe the incident in your own words..."*
- Submit button: **Parse Intent** (not "Run Diagnosis" yet)
- On submit: `POST /api/intake/parse` with `{description}`
- On response: render structured preview below

#### 8.2.2 Structured preview (editable)
Fields, each as a small editable input pre-filled by LLM extraction:
- `app_name` (dropdown, defaults to LLM extraction)
- `environment` (dropdown: dev/uat/prod)
- `severity` (dropdown: P1/P2/P3)
- `error_signature` (text)
- `affected_user_or_doc_id` (text, optional)
- `time_window` (text, default `-15m`)

Big green button: **Run Diagnosis** → existing `POST /api/trigger` with the structured fields.

#### 8.2.3 Retrieval indicator strip (above the existing tool cards)
Three pill badges:
- 🟢 `Cache: 12 hits, freshest 2m ago` (or 🟡 `Cache: stale (>30m)`, or ⚪ `Cache: cold`)
- 🟢 `Pattern: upload_failure_500 (0.87)` (or ⚪ `Pattern: none — using LLM`)
- 🟢 `Runbooks: 2 retrieved` (or ⚪ `Runbooks: none`)

These update via SSE events `cache_status`, `pattern_matched`, `runbook_retrieved`.

#### 8.2.4 Freshness footer
Bottom-right: `Prewarm last ran: 4 min ago` — read from `/api/health` (which reads the manifest).

### 8.3 New SSE event types
| Event | Payload | Handler |
|---|---|---|
| `intake_parsed` | `{extracted_fields}` | Populate structured preview |
| `cache_status` | `{hit_count, freshest_age_seconds}` | Update cache pill |
| `pattern_matched` | `{pattern_id, score}` or `null` | Update pattern pill |
| `runbook_retrieved` | `{runbook_ids: list[str]}` | Update runbook pill |

### 8.4 New backend endpoint
```python
@app.post("/api/intake/parse")
async def parse_intake(req: IntakeRequest) -> IncidentContext:
    """LLM-extract structured fields from free-text description."""
    extracted = await intake_extractor.extract(req.description)
    return extracted
```

`intake_extractor` is a thin wrapper around `LlmClient.ask()` with a system prompt that constrains output to JSON matching `IncidentContext` schema. Reuse existing JSON extraction utilities from `strategist.py` (do not duplicate).

### 8.5 Backwards compatibility
Existing endpoints (`/api/trigger`, `/api/stream`, `/api/approve`, `/api/status`) **must not change signatures**. The frontend's existing flow (after structured fields are populated) is unchanged.

### 8.6 Tests
- `tests/test_intake_extractor.py` — Aayushi's two scenario descriptions → expected fields (≥ 80% field accuracy)
- Browser smoke test (manual): chatbox → preview → run → SSE events arrive in correct order

---

## 9. Pipeline integration changes

### 9.1 `strategist.gather_intelligence()` — augmented flow

```python
async def gather_intelligence(self, incident, event_callback):
    # NEW: Phase -1 — pattern matching
    pattern_match = pattern_matcher.match(incident, self.pattern_registry)
    if pattern_match.matched:
        await event_callback("pattern_matched", pattern_match.dict())

    # NEW: Phase 0 — cache lookup (parallel with live calls)
    cache_task = asyncio.create_task(self._query_cache(incident))

    # EXISTING: Phase 0 (DT timestamp), Phase 1 (live MCP calls)
    # — but Splunk call now uses the rendered pattern SPL if pattern_match.matched
    live_responses = await self._gather_live(incident, pattern_match)

    # NEW: merge cache + live
    cache_responses = await cache_task
    await event_callback("cache_status", {
        "hit_count": len(cache_responses),
        "freshest_age_seconds": min((r.age_seconds for r in cache_responses), default=None),
    })

    return self._merge_responses(cache_responses, live_responses)
```

### 9.2 `strategist.generate_runbook()` — augmented flow

```python
async def generate_runbook(self, incident, intelligence, event_callback):
    # NEW: retrieve reference runbooks
    retrieved = await runbook_retriever.retrieve(incident, k=3)
    if retrieved:
        await event_callback("runbook_retrieved", {
            "runbook_ids": [r.runbook_id for r in retrieved],
        })

    # NEW: retrieve domain KB snippets
    kb_snippets = await self._retrieve_kb(incident, k=5)

    # EXISTING: build user message — now includes retrieved runbooks + KB
    user_message = self._build_user_message(
        incident, intelligence, retrieved_runbooks=retrieved, kb_snippets=kb_snippets
    )

    # EXISTING: LLM call, JSON extraction, validation
    ...
```

### 9.3 Token budget
The `max_context_tokens=12000` budget is **shared** across: live intelligence + cache hits + retrieved runbooks + KB snippets + the prompt itself. Compression order (most-aggressive first when budget is tight):
1. Truncate cached Splunk results (keep most recent)
2. Truncate KB snippets to top-3
3. Truncate runbook step descriptions to one line
4. Drop oldest cache hits

Implement as a `ContextBudgetManager` helper.

---

## 10. Schemas (Pydantic — additions to `models/schemas.py`)

```python
# Vector store
class VectorDoc(BaseModel): ...
class VectorHit(BaseModel): ...

# Patterns
class PatternSchema(BaseModel): ...
class MatchResult(BaseModel): ...

# Runbook docs (distinct from GeneratedRunbook!)
class RunbookDocStep(BaseModel): ...
class RunbookDoc(BaseModel): ...
class RunbookHit(BaseModel): ...

# Intake
class IntakeRequest(BaseModel):
    description: str

# Prewarm manifest
class JobResult(BaseModel): ...
class PrewarmManifest(BaseModel): ...
```

Place additions in a new file `models/retrieval_schemas.py` and re-export from `models/__init__.py` to keep `schemas.py` focused.

---

## 11. Implementation phases (vertical slices, in order)

Each phase ships an independently testable, demo-improving artifact. **Do not start phase N+1 until phase N is green.**

### Phase A — Foundation (Day 8) — ~1 day
- [ ] `vector_store/` with ChromaDB backend + embedder
- [ ] `tests/test_vector_store.py` passes
- [ ] FastAPI lifespan loads embedder once
- [ ] Demo value: none yet, but unblocks everything else

### Phase B — Domain KB indexing (Day 9 morning) — ~0.5 day
- [ ] `domain_kb/` directory convention finalized with Jey
- [ ] `domain_kb` loader + chunker
- [ ] One-shot script: `python -m prewarmer.runner --jobs index_domain_kb`
- [ ] KB injection in `generate_runbook()` (top-5 snippets)
- [ ] Demo value: **Strategist diagnosis becomes visibly more CDS-specific**

### Phase C — Runbook retrieval (Day 9 afternoon) — ~0.5 day
- [ ] `runbook_lib/` complete
- [ ] Aayushi's two scenarios → corresponding runbook YAML in `domain_kb/runbooks/`
- [ ] Retriever integrated into `generate_runbook()`
- [ ] New SSE event `runbook_retrieved` + UI pill
- [ ] Demo value: **"AEGIS pulled up runbook RB-CDS-001 for this exact failure" — strong narrative beat**

### Phase D — Query patterns (Day 10) — ~1 day
- [ ] `query_patterns/` complete
- [ ] Two patterns authored: `upload_failure_500.yaml`, `upload_latency_p99_breach.yaml`
- [ ] Matcher + renderer integrated into `_build_intelligence_calls()`
- [ ] New SSE event `pattern_matched` + UI pill
- [ ] Demo value: **Splunk queries are deterministic on demo day; no LLM hallucinated SPL**

### Phase E — Prewarmer + cache reads (Day 11) — ~1 day
- [ ] `prewarmer/runner.py` runnable manually
- [ ] All 5 jobs implemented
- [ ] Manifest written + read by `/api/health`
- [ ] `gather_intelligence()` queries cache in parallel with live calls
- [ ] New SSE event `cache_status` + UI pill + freshness footer
- [ ] Windows Task Scheduler `schedule.bat` tested on Jey's VDI
- [ ] Demo value: **Gather phase visibly faster on cached scenarios**

### Phase F — Chatbox UI (Day 12) — ~1 day
- [ ] `/api/intake/parse` endpoint
- [ ] `intake_extractor` with LLM prompt
- [ ] Chatbox + structured preview replaces existing form
- [ ] New SSE event `intake_parsed`
- [ ] Demo value: **Demo opening becomes "watch me describe an incident in plain English"**

### Phase G — Demo replay mode + rehearsal (Day 13) — ~1 day
- [ ] `settings.replay_mode=True` pins MCP responses to disk snapshots
- [ ] Snapshot capture script: `python -m prewarmer.runner --capture-snapshots`
- [ ] Two end-to-end demo rehearsals with someone watching
- [ ] Presenter notes / known-issues whitelist
- [ ] Demo value: **Determinism on demo day**

> ⚠️ **Cut order if time runs short**: drop in this order — F (chatbox), then E (prewarmer scheduling — keep ad-hoc cache populate), then D (patterns — keep ad-hoc SPL). C and B are the highest-leverage demo wins.

---

## 12. Demo-day flow (what the audience sees)

```
[00:00]  Jey opens browser → AEGIS war-room loaded
         Footer: "Prewarm last ran: 3 min ago"

[00:05]  Jey types in chatbox: "Documents are failing to upload on
         Foundation — users seeing 500 errors for the last 10 min,
         affecting prod."
         Click [Parse Intent]

[00:08]  Structured preview appears:
            app_name: foundation
            environment: prod
            severity: P1
            error_signature: HTTP 500 on upload
            time_window: -10m
         Jey reviews, clicks [Run Diagnosis]

[00:09]  Three pills light up immediately:
            🟢 Pattern: upload_failure_500 (0.91)
            🟢 Cache: 8 hits, freshest 1m ago
            🟢 Runbooks: 2 retrieved

[00:11]  Tool cards complete (Splunk green, DT green, Confluence green,
         Jira green) — gather phase done in ~2s instead of 30s

[00:12]  Planning starts → diagnosis appears in right panel within 15s
         (LLM now has cache + 2 runbooks + KB snippets — denser context,
         same call duration, dramatically better output)

[00:30]  Executor begins. First step is approval-gated. Modal opens with
         clear rollback plan (sourced from retrieved runbook RB-CDS-001).

[00:35]  Jey clicks Approve. Demo-mode remediation completes in 2s.

[00:40]  Run completes. Total wall clock: ~35s instead of ~111s.
```

**Audience takeaway**: AEGIS doesn't just react — it remembers. It's seen this kind of incident before. The demo narrative writes itself.

---

## 13. ADR log

| # | Decision | Alternatives rejected | Rationale |
|---|---|---|---|
| 1 | ChromaDB embedded as default vector backend | Qdrant Docker, Qdrant embedded, FAISS, Pinecone | No daemon required; file-backed; runs on locked-down VDI; abstraction lets us swap |
| 2 | `sentence-transformers/all-MiniLM-L6-v2` for embeddings | OpenAI embeddings, docviewer-be embeddings (unconfirmed), instructor-xl | Small, fast, CPU-friendly, offline-capable; can swap if docviewer-be exposes endpoint |
| 3 | Prewarmer runs out-of-process via Windows Task Scheduler | In-process asyncio cron inside FastAPI | Decouples cache freshness from app uptime; survives FastAPI restarts; testable independently |
| 4 | Cache *augments*, never *replaces*, live MCP calls | Cache-first with live fallback only on miss | Cache staleness during a brand-new incident is a real failure mode; parallel live call is cheap insurance |
| 5 | `query_patterns/` is YAML on disk, not in vector store | Pure vector retrieval of patterns | Patterns are small, structured, manually curated — file system is the right fit; vector store indexes their *examples* for matching |
| 6 | Chatbox + LLM extraction + editable preview, not pure chatbox | Conversational multi-turn, pure structured form | Free-text feels modern; structured preview keeps user in control; one round-trip is fast enough |
| 7 | `RunbookDoc` (retrieved) and `GeneratedRunbook` (produced) are distinct schemas | Single unified schema | They serve different purposes; conflating creates upsert/conflict bugs |
| 8 | Demo replay mode pins MCP responses to disk snapshots | Live MCP on demo day with fingers crossed | Determinism on demo day is non-negotiable; snapshot capture is cheap |
| 9 | Domain KB is file-system stand-in for future Domain MCP | Wait for Domain MCP | Demo can't wait; loader interface is small enough to swap later |
| 10 | Token budget is shared across cache + retrieved + live, with compression manager | Add to existing budget | Honors existing 12k cap; avoids LLM context overflow surprises |

---

## 14. Risks + mitigations

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 1 | Embedding model can't be downloaded on VDI | Med | High | Pre-download on a permitted machine, copy to VDI as offline model directory; spec defaults to `embedding_model_path` |
| 2 | ChromaDB write contention between prewarmer and FastAPI | Low | Med | DuckDB handles it; verify with concurrent-access test; add file lock if needed |
| 3 | Retrieved runbooks bloat token budget → LLM truncates real intelligence | Med | High | `ContextBudgetManager` with explicit compression order; cap retrieved-runbook tokens at 25% of budget |
| 4 | Chatbox extraction LLM call adds latency to incident start | Low | Low | One short LLM call (~2s); acceptable; can be optimistic-rendered with skeleton fields |
| 5 | Pattern matcher false positives → wrong SPL for the actual incident | Med | High | `confidence_floor` in pattern YAML; below threshold falls back to LLM-generated SPL; log all matches for review |
| 6 | Aayushi's intake form responses arrive after demo day | Med | Med | Author scenario fixtures from existing `intake format/` markdown; iterate when she responds |
| 7 | KB folder structure assumed in spec doesn't match Jey's actual folder | Med | Low | Loader is small; adjust patterns once structure is confirmed |
| 8 | Prewarmer ADFS token expires, no human at VDI | Low (demo day) | Med | Run device-code refresh manually before demo; manifest surfaces auth state in UI |
| 9 | Vector retrieval recall is poor on small KB corpus | Med | Med | Hybrid retrieval: vector + keyword overlap as tiebreaker; tune `k` per collection |
| 10 | Scope creep — spec drifts toward Domain MCP shape | High | Med | Hard line: file-system KB only. Domain MCP integration is post-demo work. |

---

## 15. Open questions Jey must resolve before Phase A

1. **Does `docviewer-be` expose an embeddings endpoint?** If yes, swap `embedder.py` to use it (faster, no model download). If no, confirm `sentence-transformers` model can be installed offline.
2. **Confirm `domain kb` folder structure** — do the conventions in §7.2 match what's actually there? If not, share the actual tree.
3. **Confirm Docker availability on VDI** — even if we default to ChromaDB, knowing whether Docker is available informs the Qdrant adapter priority.
4. **Confirm ADFS token TTL** — drives prewarmer schedule cadence and demo-day refresh procedure.
5. **Aayushi's chatbox field list** — confirm the 6 fields in §8.2.2 match what she said is needed for Splunk query formation. If she named additional fields, add them to `IncidentContext` and the structured preview.
6. **Demo date and slot length** — drives Phase A–G prioritization and Phase G rehearsal time.

---

## 16. Copilot vibe-coding playbook

When sitting down with Copilot in a session, follow this script per phase. Do not skip steps.

### Per-phase opening prompt to Copilot
> *"We're working on the AEGIS Day 8+ spec, Phase X. Open the spec at `/aegis-day8-prewarm-retrieval-spec.md` and read sections [the relevant ones]. Confirm you understand the scope of this phase and the success criteria. Then propose the file-by-file implementation order. Do not write code yet."*

### After Copilot proposes order
> *"Implement file 1. Stop after the file is written and tests pass. Do not move to file 2."*

### After each file
- Run `pytest tests/test_<file>.py -v`
- If green, commit with message: `phase X: <file> — <one-line summary>`
- If red, share the failure with Copilot and iterate. Do not move on.

### Phase exit criteria
- All planned files committed
- All planned tests green
- `python -m test_pipeline` (existing CLI green path) **still passes**
- `dev_docs/dayN.md` updated with what was built and any new learnings

### Anti-patterns to call out to Copilot
- Writing more than one file per turn
- Adding TODOs / `NotImplementedError` to "come back later"
- Modifying Day 1–7 code without explicit reason and a passing `test_pipeline` after
- Inventing new schemas instead of reusing existing ones from `models/schemas.py`

---

## 17. File-by-file deliverables checklist

Use this as the master TODO. Tick as you go.

### Backend
- [ ] `backend/vector_store/__init__.py`
- [ ] `backend/vector_store/base.py`
- [ ] `backend/vector_store/chroma_backend.py`
- [ ] `backend/vector_store/qdrant_backend.py` (stub)
- [ ] `backend/vector_store/embedder.py`
- [ ] `backend/vector_store/collections.py`
- [ ] `backend/query_patterns/__init__.py`
- [ ] `backend/query_patterns/loader.py`
- [ ] `backend/query_patterns/matcher.py`
- [ ] `backend/query_patterns/renderer.py`
- [ ] `backend/query_patterns/schemas.py`
- [ ] `backend/query_patterns/patterns/upload_failure_500.yaml`
- [ ] `backend/query_patterns/patterns/upload_latency_p99_breach.yaml`
- [ ] `backend/runbook_lib/__init__.py`
- [ ] `backend/runbook_lib/loader.py`
- [ ] `backend/runbook_lib/retriever.py`
- [ ] `backend/runbook_lib/injector.py`
- [ ] `backend/runbook_lib/schemas.py`
- [ ] `backend/intake/__init__.py`
- [ ] `backend/intake/extractor.py`
- [ ] `backend/intake/prompt.md`
- [ ] `backend/models/retrieval_schemas.py`
- [ ] `backend/main.py` — add `/api/intake/parse` and `/api/health` enrichment
- [ ] `backend/engine/strategist.py` — pattern + cache + runbook integration
- [ ] `backend/config.py` — new settings (`vector_store_path`, `embedding_model_name`, `embedding_model_path`, `replay_mode`, `domain_kb_path`)

### Prewarmer
- [ ] `prewarmer/__init__.py`
- [ ] `prewarmer/runner.py`
- [ ] `prewarmer/jobs.py`
- [ ] `prewarmer/manifest.py`
- [ ] `prewarmer/schedule.bat`
- [ ] `prewarmer/README.md`

### Domain KB (created by Jey, not Copilot)
- [ ] `domain_kb/apps/*.md` — at least cds-store and foundation
- [ ] `domain_kb/failure_modes/upload_failure.md`
- [ ] `domain_kb/failure_modes/upload_latency.md`
- [ ] `domain_kb/runbooks/RB-CDS-001-upload-500.yaml`
- [ ] `domain_kb/runbooks/RB-CDS-002-upload-latency.yaml`

### Frontend
- [ ] `frontend/index.html` — chatbox + structured preview + retrieval pills + freshness footer + new SSE handlers

### Tests
- [ ] `tests/test_vector_store.py`
- [ ] `tests/test_embedder.py`
- [ ] `tests/test_pattern_matcher.py`
- [ ] `tests/test_pattern_renderer.py`
- [ ] `tests/test_pattern_loader.py`
- [ ] `tests/test_runbook_loader.py`
- [ ] `tests/test_runbook_retriever.py`
- [ ] `tests/test_runbook_injector.py`
- [ ] `tests/test_domain_kb_loader.py`
- [ ] `tests/test_intake_extractor.py`
- [ ] `tests/test_prewarmer_jobs.py`
- [ ] `tests/test_prewarmer_manifest.py`
- [ ] `tests/test_concurrent_access.py`
- [ ] `tests/test_pipeline.py` — existing, must still pass

### Docs
- [ ] `dev_docs/day8.md`
- [ ] `dev_docs/day9.md`
- [ ] `dev_docs/day10.md`
- [ ] `dev_docs/day11.md`
- [ ] `dev_docs/day12.md`
- [ ] `dev_docs/day13.md`
- [ ] `dev_docs/learnings.md` — appended throughout

---

*End of spec. Total ~6 days of work assuming uninterrupted Copilot sessions. Cut Phase F and G first if time runs short. Resolve §15 open questions before Phase A starts.*
