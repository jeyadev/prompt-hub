# AEGIS Execution Control & Provenance — Architectural Specification

**Version:** 1.0
**Status:** Draft, pending pre-validation
**Owner:** Jey
**Scope:** Phases 0–3 (Phase 4 deferred to follow-on spec)
**Predecessor:** AEGIS MVP (Phase F shipped — commit `2016ff7`)

---

## 1. Executive Summary

This spec lands four capabilities on top of AEGIS MVP, sequenced as five phases:

| Phase | Capability | Architectural weight |
|---|---|---|
| **0** | Minimal frontend modularity (Alpine.data() component split) | Foundation |
| **1** | Execution control plane: manual play, pause, resume | Paradigm shift — operator becomes co-pilot |
| **2** | Provenance layer: query → tool-call → output visibility | Observability + audit |
| **3** | T1 runbook modification: skip / abort / edit-params | Builds on Phase 1 |
| **4** *(deferred)* | T2 runbook modification: add / remove / reorder steps | Future work |

The most important architectural change is **Phase 1**, which converts AEGIS from "autonomous agent with approval gates" to "operator-controlled execution with approval gates." Phase 1 must land before Phase 3, because skip / abort / edit-params are pause-state operations.

Each phase opens with a mandatory **pre-development validation gate**. The vibe coding guide enforces that no implementation begins until validation answers are written into the spec in the designated `[VALIDATION-PENDING]` blocks.

---

## 2. Context

### 2.1 Current state (post-Phase F)

AEGIS today executes incidents end-to-end:

```
POST /api/trigger
  → IntakeExtractor (LLM)
  → gather_intelligence (parallel MCP fan-out: Splunk Prod, Splunk UAT, Dynatrace, Confluence, Jira)
  → generate_runbook (LLM planning)
  → Executor.run() (sequential step loop, autonomous)
     ├── DiagnosticAgent (read-only, WRITE_TOOLS check enforced)
     ├── DecisionAgent (LLM branching)
     └── RemediationAgent (write, demo-mocked, human approval gate)
  → run_completed
```

The executor runs autonomously once planning completes. Approval gates pause execution only at write steps. The operator has no ability to inspect or interrupt the run before write boundaries.

### 2.2 What Jey asked for

1. Ability to modify the runbook while executing (user prompt or structured edit).
2. Intelligence-gathering UI: show the queries used, not just outputs.
3. Step UI: show the tool calls (input + output), not just step result.
4. Modularity in the frontend (currently 1,366-line `index.html`).

### 2.3 Reframing applied during design review

- **Item 1** is not one feature — it's a tier (T1 → T2 → T3). v1 ships T1 only.
- **A new Item 0** was surfaced: manual play / pause / resume on the runbook itself. T1 cannot be built on top of an autonomous executor. Execution control is its own architectural change.
- **Items 2 + 3** are siblings sharing infrastructure (SSE event enrichment + reusable tool detail modal). They ship together.
- **Item 4** is foundational. Done before the others, it is cheap. Done after, it doubles in cost.

### 2.4 What this spec deliberately defers

- T2 modification (add / remove / reorder steps) — Phase 4, separate spec.
- T3 modification (free-form prompt-driven re-planning) — not on roadmap.
- SQLite persistence — separate workstream, but cross-references included where state shape changes.
- AEGIS Knowledge Graph integration — separate workstream.

---

## 3. Pre-existing findings (carried forward from architecture diagram review, 2026-04-28)

These are **not in scope for this spec** but are known unknowns that interact with the work below. Each has a [VALIDATION-PENDING] block in the relevant phase to confirm the work here does not make them worse.

| # | Finding | Risk | Phase that may interact |
|---|---|---|---|
| **F1** | Asymmetric WRITE_TOOLS enforcement: diagnostic path has the deterministic check, remediation path appears to rely on human approval only | Audit / governance violation if RemediationAgent calls a non-WRITE tool by mistake | Phase 3 (edit-params re-validates WRITE_TOOLS, must land symmetrically on both paths) |
| **F2** | SSE event count: audit lists 15 events, diagram shows 14 (`planning_started` not visible) | Frontend may listen for an event that never fires | Phase 1 (adds new SSE events, opportunity to audit existing ones) |
| **F3** | Graceful degradation drawn for M1 (Splunk Prod) and M3 (Dynatrace) only. M2 (Splunk UAT), M4 (Confluence), M5 (Jira) have no degraded path on diagram | Crash on Confluence/Jira outage instead of degraded continuation | Phase 2 (provenance must show degradation status per tool call) |
| **F4** | DT_TS (Dynatrace timestamp anchor) is a sequential dependency before parallel fan-out, with no fallback shown | Single point of failure for incident triage timing | Phase 2 (provenance UI exposes this — operator visibility may surface the risk) |
| **F5** | `S4 (runbook_retrieved)` emits SSE; `S3 (domain_kb retrieve)` does not | Asymmetric retrieval observability | Phase 2 (fix while enriching events) |

**Pre-validation requirement (cross-phase):** Before Phase 2 begins, confirm whether F1, F3, F5 are diagram omissions or genuine code gaps. If genuine, file as separate corrective work — do not silently fix as part of this spec.

---

## 4. Pre-Development Validation Framework

Every phase opens with three categories of validation. **No phase implementation begins until all three categories are answered.**

### 4.1 Category structure

| Category | Definition | Method | Owner |
|---|---|---|---|
| **CV** — Code-Verifiable | Answer is in the codebase. Read it. | Vibe guide Session 1 reads files. | IDE (Copilot) |
| **BV** — Behavior-Verifiable | Answer requires running code or probing system | Vibe guide Session 1 includes a probe script | IDE + Jey reviews output |
| **SV** — Stakeholder-Verifiable | Only a human can answer (Jey, Aayushi, Kunal) | Asked in chat, answer pasted into spec | Jey |

### 4.2 Validation gate enforcement

Each phase has a `[VALIDATION-PENDING]` block per assumption. The block is the only legal place to record an answer. Vibe guide Session 1 of each phase **must not exit** until every block is filled with one of:

- A direct answer + evidence (file path + line number, or probe output).
- An explicit `DEFERRED — proceeding with assumption X, will revisit if Y`.
- A `BLOCKED — cannot proceed until Z` (halt the phase).

### 4.3 Why this matters

The IDE will fabricate answers if you let it. Pre-validation prevents the spec from drifting into "Copilot decided" territory. It also creates an audit trail: if a phase fails review, the validation answers are the first thing to inspect.

---

## 5. Phase Plan

| Phase | Effort | Demo-visible? | Depends on |
|---|---|---|---|
| **0 — Modularity** | 2–3 days | No (refactor) | Nothing |
| **1 — Execution control** | 1.5–2 weeks | Yes (play button is the most visible change) | Phase 0 |
| **2 — Provenance** | 1 week | Yes (clickable depth in tool calls) | Phase 0, ideally Phase 1 |
| **3 — T1 modification** | 1.5 weeks | Yes (skip / abort / edit during pause) | Phase 1, Phase 2 |

**Total:** ~5 weeks engineering on top of the existing AEGIS MVP. Fits within the 7-week MVP demo target with ~2 weeks of buffer for integration, demo rehearsal, and Aayushi's scenario validation.

---

## 6. Phase 0 — Minimal Frontend Modularity

### 6.1 Goal

Split the 1,366-line `frontend/index.html` into Alpine.data() components in separate JS files, **without introducing a build step**. This is foundation work for Phases 1–3, which add a runbook editor component, a richer tool detail modal, and execution control widgets.

### 6.2 Non-goals

- No Vue / React / Svelte migration.
- No build step. No npm. No Vite.
- No CSS framework change. Tailwind via CDN stays.
- No restructuring of HTML markup beyond what's needed to host components.

### 6.3 Pre-development validation

#### CV-0.1 — Existing frontend structure
**Claim:** `frontend/` contains only `index.html` (single file, no module structure today).
**Risk if wrong:** May overwrite existing modular structure.
**Method:** `ls frontend/` and `wc -l frontend/index.html`.
**[VALIDATION-PENDING]**

#### CV-0.2 — Alpine.data() partitioning
**Claim:** All ~40 functions live in a single `sre()` Alpine component, not already partitioned.
**Risk if wrong:** Effort estimate is wrong.
**Method:** Open `index.html`, count `Alpine.data(...)` registrations.
**[VALIDATION-PENDING]**

#### CV-0.3 — Tool detail modal location
**Claim:** Tool detail modal is inline in `index.html`, not already a separate template/include.
**Risk if wrong:** Phase 2 plan changes.
**Method:** Search `index.html` for "tool detail" / modal rendering code.
**[VALIDATION-PENDING]**

#### BV-0.1 — Multiple JS file serving
**Claim:** FastAPI in `main.py` serves `frontend/` as static files via a route that supports arbitrary subpaths.
**Risk if wrong:** Need to add static-file routes for new JS files.
**Method:** Check `main.py` for `StaticFiles` mount or equivalent. Probe: `curl http://localhost:PORT/frontend/test.js` after dropping a test file.
**[VALIDATION-PENDING]**

#### SV-0.1 — Single-file deployment constraint
**Claim:** JPMC Private Cloud serving model accepts multiple JS files (no all-in-one deployment requirement).
**Owner:** Jey.
**[VALIDATION-PENDING]**

### 6.4 Architecture

```
frontend/
├── index.html                    (shell: layout, mounts, top-level Alpine x-data)
├── components/
│   ├── incident-trigger.js       (left chatbox, parse-intent, datasource picker)
│   ├── runbook-view.js           (Phase 1 will extend this for play/pause)
│   ├── step-cards.js             (Phase 2 will extend for tool call display)
│   ├── tool-detail-modal.js      (extracted from inline; reused in Phases 2-3)
│   ├── approval-modal.js
│   ├── decision-modal.js
│   └── sse-client.js             (centralized EventSource handling, dispatch to components)
├── lib/
│   ├── format-utils.js           (formatToolOutputRich, highlightYaml, etc.)
│   └── api.js                    (fetch wrappers for /api/* endpoints)
└── (no node_modules, no build artifacts)
```

### 6.5 Component contract

Each component file exports an Alpine.data() registration:

```javascript
// components/runbook-view.js
document.addEventListener('alpine:init', () => {
    Alpine.data('runbookView', () => ({
        runbook: null,
        // ...
        init() { /* listeners */ },
        // ...methods
    }));
});
```

`index.html` mounts components by name:

```html
<div x-data="runbookView" x-init="init()"></div>
```

### 6.6 SSE dispatch refactor

Today: a single connectSSE() registers handlers for 15 events directly on Alpine state.
After Phase 0: `components/sse-client.js` registers a single EventSource, dispatches typed CustomEvents (`aegis:tool_started`, `aegis:step_completed`, etc.) on `window`. Components subscribe to the events relevant to them.

This is the change that makes Phase 1 (play/pause) and Phase 3 (modify-during-pause) clean. Without it, every component that needs SSE state has to know about the central Alpine store. With it, components are decoupled.

### 6.7 Implementation order

1. Extract `lib/format-utils.js` (zero risk — pure functions).
2. Extract `lib/api.js` (low risk — fetch wrappers).
3. Build `components/sse-client.js` with the CustomEvent dispatch contract.
4. Migrate one component end-to-end (recommend: `tool-detail-modal.js`) to validate the pattern.
5. Migrate remaining components.
6. Remove dead code from `index.html`.

### 6.8 Acceptance criteria

- `frontend/index.html` is under 400 lines.
- All component files are independently loadable (no circular deps).
- Existing demo scenarios pass with no functional change.
- SSE dispatch goes through CustomEvents, not direct Alpine store mutations from connectSSE().
- A new component can be added in under 50 lines + a single `<script>` tag in `index.html`.

---

## 7. Phase 1 — Execution Control Plane

### 7.1 Goal

Convert AEGIS from autonomous executor with approval gates to operator-controlled executor with approval gates. The operator must explicitly press **Play** to start step execution after planning completes. The operator can **Pause** between steps and **Resume** at will.

### 7.2 The architectural shift

| | Before | After |
|---|---|---|
| Post-planning state | Auto-execute first step | `AWAITING_PLAY` — wait for operator |
| Step transitions | Autonomous | Operator-gated (pause-aware) |
| Approval gates | Sole pause mechanism | Co-exist with operator pause |
| Operator role | Reviewer | Co-pilot |
| Default mode for v1 | (none) | Manual-only. No auto-play option. |

### 7.3 Pre-development validation

#### CV-1.1 — Executor structure
**Claim:** `Executor.run()` is async, iterates a step list, and has a single point where each step transition occurs.
**Risk if wrong:** Pause checkpoint placement is wrong.
**Method:** Read `backend/engine/executor.py`. Map the loop. Identify step-boundary points.
**[VALIDATION-PENDING]**

#### CV-1.2 — RunState representation
**Claim:** `RunState` is a Pydantic model in `backend/models/schemas.py` with a `status` field and a `runbook` field.
**Risk if wrong:** Adding new statuses / fields requires different patterns.
**Method:** Read schemas.py. Identify model class and current statuses.
**[VALIDATION-PENDING]**

#### CV-1.3 — Approval gate mechanism
**Claim:** Existing approval gate pauses execution via `asyncio.Event` or equivalent (not polling).
**Risk if wrong:** Pause/resume needs a different pattern; can't reuse approval primitives.
**Method:** Read `_wait_for_approval()` in executor.py. Identify the pause primitive.
**[VALIDATION-PENDING]**

#### CV-1.4 — SSE emission point
**Claim:** All SSE events emit through a single `_emit()` function or a small set of points.
**Risk if wrong:** Adding events requires editing scattered locations.
**Method:** Search executor.py / strategist.py / agents.py for `emit` calls.
**[VALIDATION-PENDING]**

#### CV-1.5 — API endpoint pattern
**Claim:** `main.py` has a pattern for `POST /api/run/{id}/<action>` (e.g., `/api/approve/{run_id}`).
**Risk if wrong:** New endpoints need to follow a new pattern.
**Method:** Read main.py route definitions.
**[VALIDATION-PENDING]**

#### BV-1.1 — Mid-step pause behavior
**Claim:** If an SSE connection drops mid-step, the run continues to completion (server-side run is independent of SSE clients).
**Risk if wrong:** Pause semantics are confused with disconnect semantics.
**Method:** Trigger a run, kill the SSE connection mid-step, observe server logs.
**[VALIDATION-PENDING]**

#### BV-1.2 — Typical step duration
**Claim:** Splunk MCP calls complete in under 30 seconds in the current setup.
**Risk if wrong:** Pause-after-current-step UX needs visible progress feedback.
**Method:** Add timing logs to one MCP call. Run 5 incidents. Capture distribution.
**[VALIDATION-PENDING]**

#### SV-1.1 — Default mode
**Claim:** v1 default is `AWAITING_PLAY` (manual only). No auto-play option.
**Owner:** Jey. **(Confirmed in chat 2026-04-29.)**
**[ANSWER:** Manual-only. No auto-play in v1.**]**

#### SV-1.2 — Pause timeout
**Claim:** v1 has no timeout on PAUSED runs. They sit indefinitely until operator resumes or aborts.
**Owner:** Jey.
**[VALIDATION-PENDING]**

#### SV-1.3 — Step granularity
**Claim:** Pause takes effect at next step boundary. No mid-step cancellation.
**Owner:** Jey. **(Confirmed via recommendation acceptance.)**
**[ANSWER:** Pause at step boundary only. Document as contract in operator UI.**]**

#### SV-1.4 — Demo scenario impact
**Claim:** Aayushi's incident scenarios will incorporate the play button as part of the demo flow (not bypass it).
**Owner:** Jey + Aayushi.
**[VALIDATION-PENDING]**

### 7.4 State machine

```
                          ┌─────────────────────┐
                          │  GATHERING          │
                          └──────────┬──────────┘
                                     │
                          ┌──────────▼──────────┐
                          │  PLANNING           │
                          └──────────┬──────────┘
                                     │ (planning_completed)
                          ┌──────────▼──────────┐
            ┌─────────────│  AWAITING_PLAY      │ ◄── NEW
            │             └──────────┬──────────┘
   abort    │                        │ play
            │             ┌──────────▼──────────┐
            ├─────────────│  RUNNING            │
            │             └──────────┬──────────┘
   abort    │                        │ pause requested
            │             ┌──────────▼──────────┐
            ├─────────────│  PAUSED             │ ◄── NEW
            │             └──────────┬──────────┘
            │                        │ resume / step_forward
            │                        │
            │             ┌──────────▼──────────┐
            ├─────────────│  AWAIT_APPROVAL     │ (existing)
            │             └──────────┬──────────┘
            │                        │ approved
            │             ┌──────────▼──────────┐
            ├─────────────│  AWAIT_DECISION     │ (existing)
            │             └──────────┬──────────┘
            │                        │
            │             ┌──────────▼──────────┐
            └────────────►│  RESOLVED / ESCALATED / ABORTED │
                          └─────────────────────┘
```

**Key invariants:**
- Only one of `RUNNING`, `PAUSED`, `AWAIT_APPROVAL`, `AWAIT_DECISION` is active at a time.
- Approval / decision gates take precedence over pause: if a step ends and lands on an approval gate, the run enters `AWAIT_APPROVAL` regardless of `pause_requested`. The operator must approve/reject before the pause can take effect.
- `AWAITING_PLAY` is reached exactly once per run (post-planning). If the operator aborts from `AWAITING_PLAY`, no steps execute.

### 7.5 Pydantic schema changes

```python
# backend/models/schemas.py

class RunStatus(str, Enum):
    GATHERING = "gathering"
    PLANNING = "planning"
    AWAITING_PLAY = "awaiting_play"     # NEW
    RUNNING = "running"
    PAUSED = "paused"                    # NEW
    AWAIT_APPROVAL = "await_approval"
    AWAIT_DECISION = "await_decision"
    RESOLVED = "resolved"
    ESCALATED = "escalated"
    ABORTED = "aborted"                  # NEW

class PauseReason(str, Enum):
    USER_REQUESTED = "user_requested"
    APPROVAL_GATE = "approval_gate"
    DECISION_GATE = "decision_gate"

class RunState(BaseModel):
    # ...existing fields...
    status: RunStatus
    pause_requested: bool = False        # NEW — set by API, cleared on resume
    pause_reason: Optional[PauseReason] = None  # NEW
    current_step_index: int = 0          # NEW — explicit pointer, do not infer
    play_event: asyncio.Event = ...      # NEW — set on play / resume, cleared on pause
    abort_event: asyncio.Event = ...     # NEW — set on abort
```

### 7.6 Executor changes

The current `Executor.run()` is a tight async loop. The new shape is event-driven:

```python
class Executor:
    async def start(self, run_state: RunState):
        """
        Initialize execution. Sets state to AWAITING_PLAY.
        Does NOT begin step execution. Returns immediately.
        """
        run_state.status = RunStatus.AWAITING_PLAY
        await self._emit("status_change", run_state)
        # Begin background watcher task
        asyncio.create_task(self._execution_loop(run_state))

    async def _execution_loop(self, run_state: RunState):
        """
        Background loop. Waits for play_event. Executes steps until done,
        respecting pause and abort signals at every step boundary.
        """
        # Wait for first play
        await self._wait_for_play_or_abort(run_state)
        if run_state.status == RunStatus.ABORTED:
            return

        run_state.status = RunStatus.RUNNING
        await self._emit("status_change", run_state)

        while run_state.current_step_index < len(run_state.runbook.steps):
            # Check pause at each step boundary
            if run_state.pause_requested:
                await self._enter_pause(run_state, PauseReason.USER_REQUESTED)
                await self._wait_for_play_or_abort(run_state)
                if run_state.status == RunStatus.ABORTED:
                    return

            step = run_state.runbook.steps[run_state.current_step_index]
            await self._execute_step(step, run_state)
            run_state.current_step_index += 1

        # Done
        run_state.status = RunStatus.RESOLVED
        await self._emit("run_completed", run_state)

    async def _enter_pause(self, run_state, reason: PauseReason):
        run_state.status = RunStatus.PAUSED
        run_state.pause_reason = reason
        run_state.play_event.clear()
        await self._emit("execution_paused", run_state)

    async def _wait_for_play_or_abort(self, run_state):
        play = asyncio.create_task(run_state.play_event.wait())
        abort = asyncio.create_task(run_state.abort_event.wait())
        done, pending = await asyncio.wait(
            [play, abort], return_when=asyncio.FIRST_COMPLETED
        )
        for t in pending:
            t.cancel()
        if run_state.abort_event.is_set():
            run_state.status = RunStatus.ABORTED
            await self._emit("run_aborted", run_state)

    # External API methods
    async def play(self, run_state: RunState):
        if run_state.status not in (RunStatus.AWAITING_PLAY, RunStatus.PAUSED):
            raise InvalidTransition(...)
        run_state.pause_requested = False
        run_state.play_event.set()
        run_state.status = RunStatus.RUNNING
        await self._emit("execution_resumed", run_state)

    async def pause(self, run_state: RunState):
        if run_state.status != RunStatus.RUNNING:
            raise InvalidTransition(...)
        run_state.pause_requested = True
        # Pause takes effect at next step boundary; no event emitted here

    async def abort(self, run_state: RunState):
        run_state.abort_event.set()
        run_state.play_event.set()  # unblock any waiter
```

### 7.7 SSE event additions

| Event | When fired | Payload |
|---|---|---|
| `runbook_ready_for_execution` | After `planning_completed`, when state transitions to `AWAITING_PLAY` | `{ run_id, runbook_summary }` |
| `execution_paused` | When state enters `PAUSED` | `{ run_id, reason: PauseReason, current_step_index }` |
| `execution_resumed` | When `play()` is called from `AWAITING_PLAY` or `PAUSED` | `{ run_id, current_step_index }` |
| `run_aborted` | When `abort()` is called | `{ run_id, aborted_at_step_index }` |

**Cross-reference to Finding F2:** While adding these events, audit the existing 14 vs 15 mismatch. Fix `planning_started` if missing.

### 7.8 API additions

| Method | Path | Body | Effect |
|---|---|---|---|
| POST | `/api/run/{run_id}/play` | `{}` | Transition `AWAITING_PLAY` or `PAUSED` → `RUNNING` |
| POST | `/api/run/{run_id}/pause` | `{}` | Set `pause_requested = true`. Effective at next step boundary. |
| POST | `/api/run/{run_id}/abort` | `{ reason?: string }` | Transition any active state → `ABORTED` |

All three endpoints return current `RunState` snapshot.

### 7.9 Frontend additions

#### Runbook view extensions
- After `runbook_ready_for_execution`: render runbook with prominent **▶ Play** button.
- After `execution_resumed`: render with **⏸ Pause** button (visible during RUNNING).
- After `execution_paused`: render with **▶ Resume** + **⏭ Step Forward** + **⏹ Abort** buttons.
- Status banner: "Awaiting your play" / "Running step X of Y" / "Paused after step X" / "Aborted".

#### Step card extensions
- Each step card shows position (`Step 3 of 7`), state (pending / running / completed / skipped / aborted).
- Pending steps display dimmed.

### 7.10 Failure modes & mitigations

| Failure mode | Mitigation |
|---|---|
| Pause-during-MCP-call | Pause takes effect after current step completes. Document as UI contract. UI shows "Pausing after current step…" |
| Pause-then-abandoned | v1: no timeout. Document as known limitation. Production: add timeout in follow-on work. |
| Race between pause and approval gate | Approval gate wins. UI disables pause button while approval modal open. |
| Resume from stale state (Phase 3 dependency) | Resume validates runbook version unchanged. If changed, force re-acknowledgment. (Implemented in Phase 3.) |
| Multi-tab divergence | SSE broadcasts. Last-writer-wins on API. Both tabs reflect current state. Acceptable for v1. |
| Server restart during PAUSED | v1: in-memory state, run lost. Acceptable until SQLite persistence lands. Document. |
| Operator presses play twice rapidly | API is idempotent on `play_event.set()`. No double-execution. |

### 7.11 Acceptance criteria

- A new run reaches `AWAITING_PLAY` after planning completes; no step executes.
- Pressing Play transitions to `RUNNING` and executes step 1.
- Pressing Pause during step 3 results in pause taking effect after step 3 completes; UI shows "Pausing…" until then.
- Pressing Resume from `PAUSED` continues with step 4.
- Pressing Abort from any active state ends the run, no further steps execute.
- Approval gate during step N still works as before; pause requested during AWAIT_APPROVAL is queued, takes effect after approve/reject.
- All four new SSE events fire and are observable in browser DevTools.
- Server logs show clear state transitions for all flows.

---

## 8. Phase 2 — Provenance Layer

### 8.1 Goal

Make visible what AEGIS is actually doing:
- For `gather_intelligence`: show which pattern matched, the rendered query, and the tool output for each MCP call.
- For each step in execution: show every tool call the agent made (input + output), not just the step result.

### 8.2 Why this matters beyond UX

- **Pattern matcher errors become visible.** Today, if `pattern_matcher.match` picks the wrong pattern, you only see it in logs.
- **JPMC audit posture improves.** Per-tool-call records are the answer to "what did the system do?" without grepping Splunk.
- **Operator trust calibration.** Showing the query exposes the deterministic logic; trust shifts from "the LLM said so" to "the matcher rendered this query, here's the result."

### 8.3 Pre-development validation

#### CV-2.1 — Pattern matcher metadata
**Claim:** `pattern_matcher.match()` returns enough metadata (pattern_id, confidence, app filter) to surface in UI.
**Risk if wrong:** Need to thread metadata through; not just an SSE enrichment.
**Method:** Read `backend/query_patterns/matcher.py` return type. Inspect what's stored in matcher result objects.
**[VALIDATION-PENDING]**

#### CV-2.2 — Pattern renderer output
**Claim:** `pattern_renderer.render()` returns the rendered SPL string in a form that can be serialized to JSON.
**Risk if wrong:** Frontend can't display the query.
**Method:** Read renderer.py.
**[VALIDATION-PENDING]**

#### CV-2.3 — Agent tool call interface
**Claim:** All three agents (Diagnostic, Decision, Remediation) make tool calls via `MCPClientPool.call_tool()` (single chokepoint).
**Risk if wrong:** Per-tool-call event emission requires editing each agent.
**Method:** Read `backend/engine/agents.py`. Search for `call_tool` invocations.
**[VALIDATION-PENDING]**

#### CV-2.4 — Tool detail modal current state
**Claim:** Tool detail modal already renders Splunk tabular results.
**Risk if wrong:** Modal needs net-new rendering logic for some output types.
**Method:** Read modal code in index.html (or, post-Phase 0, in `components/tool-detail-modal.js`).
**[VALIDATION-PENDING]**

#### CV-2.5 — F1 verification (cross-finding)
**Claim:** Remediation path enforces WRITE_TOOLS check in code, not only via human approval.
**Method:** Read agents.py `RemediationAgent.run()`. Check for WRITE_TOOLS validation before tool call.
**[VALIDATION-PENDING — if FALSE, file separate corrective work and do not silently fix here.]**

#### CV-2.6 — F3 verification
**Claim:** M2, M4, M5 MCP calls handle errors with structured degradation (return error dict, not raise).
**Method:** Read MCP wrapper code for each.
**[VALIDATION-PENDING]**

#### BV-2.1 — Tool output size distribution
**Claim:** Typical Splunk MCP tool output is under 50 KB JSON.
**Risk if wrong:** SSE payload bloat; need streaming or on-demand fetch.
**Method:** Log `len(json.dumps(tool_output))` for one run. Capture min/median/max.
**[VALIDATION-PENDING]**

#### BV-2.2 — PII / sensitive data in outputs
**Claim:** Splunk results in current demo scenarios do not contain raw PII or auth tokens.
**Risk if wrong:** Surfacing them in UI is a leak.
**Method:** Sample 3 demo run outputs. Manual inspection.
**[VALIDATION-PENDING — if PII found, scope a redaction layer before this phase ships.]**

#### SV-2.1 — Display retention policy
**Claim:** Tool call records are held in run state for the lifetime of the run, no expiry.
**Owner:** Jey.
**[VALIDATION-PENDING]**

### 8.4 Backend changes

#### 8.4.1 Pattern provenance enrichment
Enrich the `pattern_matched` SSE event:

```json
{
  "event": "pattern_matched",
  "data": {
    "tool_name": "splunk_search",
    "pattern_id": "doc_upload_failure_v3",
    "pattern_confidence": 0.87,
    "app_filter": "doc-service",
    "rendered_query": "index=app sourcetype=doc-service ERROR ...",
    "fallback_used": false
  }
}
```

#### 8.4.2 Per-tool-call events
Wrap `MCPClientPool.call_tool()` to auto-emit events:

```python
class MCPClientPool:
    async def call_tool(self, server, name, params, *, run_state=None, agent_name=None):
        call_id = str(uuid4())
        if run_state:
            await self._emit("agent_tool_started", {
                "call_id": call_id,
                "step_index": run_state.current_step_index,
                "agent": agent_name,
                "server": server,
                "tool_name": name,
                "params": params,
            })
        result = await self._dispatch(server, name, params)
        if run_state:
            await self._emit("agent_tool_completed", {
                "call_id": call_id,
                "step_index": run_state.current_step_index,
                "summary": _summarize(result),  # truncated for SSE
                "size_bytes": len(json.dumps(result)),
                "degraded": result.get("degraded", False),
            })
        # Store full result keyed by call_id for on-demand fetch
        run_state.tool_calls[call_id] = {
            "started_at": ...,
            "params": params,
            "result": result,
        }
        return result
```

#### 8.4.3 On-demand full output endpoint
```
GET /api/run/{run_id}/tool_call/{call_id}
→ { params, result, started_at, completed_at, agent, server, tool_name }
```

Frontend opens modal, fetches full output on click. Solves Splunk payload bloat.

#### 8.4.4 SSE event additions

| Event | Payload |
|---|---|
| `agent_tool_started` | `{ call_id, step_index, agent, server, tool_name, params_summary }` |
| `agent_tool_completed` | `{ call_id, step_index, summary, size_bytes, degraded }` |

### 8.5 Frontend changes

#### 8.5.1 Tool detail modal extension
The reusable modal (extracted in Phase 0) now serves three contexts:
1. Gather phase tool call (existing).
2. Pattern provenance display (new — pattern_id, rendered query, output).
3. Agent tool call within step (new — params, output, agent name).

A `mode` prop drives rendering.

#### 8.5.2 Step card extension
Each step card gains a collapsible **Tool Calls** sub-section:

```
┌─ Step 3: Investigate failed uploads ──────────────────────┐
│ Status: completed                                         │
│ Result: 47 failures in last hour                          │
│                                                           │
│ ▼ Tool Calls (3)                                          │
│   ├ splunk_search       success    312 ms    [view ▾]   │
│   ├ dynatrace_metrics   degraded   1.2 s     [view ▾]   │
│   └ confluence_search   success    480 ms    [view ▾]   │
└───────────────────────────────────────────────────────────┘
```

Click `[view ▾]` → fetches full output via on-demand endpoint → opens tool detail modal.

#### 8.5.3 Gather panel extension
Each gather tool result gets a "Show query" affordance opening the modal in **Pattern provenance mode**:

```
Pattern: doc_upload_failure_v3 (confidence 0.87)
App filter: doc-service
Rendered query:
   index=app sourcetype=doc-service ERROR uploadId=*
   | stats count by error_code
Output: [view full output]
```

### 8.6 Payload size handling

- SSE events carry **summaries only**: first 500 chars of output, count of rows, key/value count.
- Full output retrieved on demand via `GET /api/run/{run_id}/tool_call/{call_id}`.
- If size > 1 MB, frontend warns and offers download instead of inline render.

### 8.7 Acceptance criteria

- Gather panel shows the rendered query for every tool call, alongside the output.
- Pattern provenance is visible: which pattern matched, with confidence.
- Each step card lists every tool call the agent made, with timing.
- Clicking a tool call opens the modal with full input + output.
- Degraded calls visually distinct (e.g. amber dot).
- SSE event size for `agent_tool_completed` stays under 5 KB regardless of underlying output size.
- F1, F3, F5 verified — separate corrective tickets filed if true gaps found.

---

## 9. Phase 3 — T1 Runbook Modification

### 9.1 Goal

While paused, the operator can:
- **Skip** the next pending step.
- **Abort** the entire run (already shipped in Phase 1, hardened here with audit).
- **Edit params** of the next pending step before resuming.

### 9.2 Pre-development validation

#### CV-3.1 — Step parameter shape
**Claim:** Each step in the runbook has a structured `params` field (Pydantic) that can be edited generically.
**Risk if wrong:** Edit-params UI requires per-step-type custom forms.
**Method:** Read RunbookPlanStep / ExecutionStep schemas.
**[VALIDATION-PENDING]**

#### CV-3.2 — WRITE_TOOLS list location
**Claim:** WRITE_TOOLS is a constant in schemas.py or constants.py, accessible from Executor.
**Risk if wrong:** Re-validation on edit needs different access pattern.
**Method:** Search for WRITE_TOOLS definition.
**[VALIDATION-PENDING]**

#### CV-3.3 — Runbook mutability semantics
**Claim:** `RunState.runbook` is mutable in-place; mutating it does not require re-issuing through SSE.
**Risk if wrong:** Edit may not propagate to frontend.
**Method:** Read schemas.py model config.
**[VALIDATION-PENDING]**

#### SV-3.1 — Audit log persistence
**Claim:** Modification audit log goes to in-memory `RunState.modification_log: List[ModificationRecord]` for v1; persists to SQLite when persistence lands.
**Owner:** Jey.
**[VALIDATION-PENDING]**

#### SV-3.2 — Approval re-trigger on edit
**Claim:** A step whose params are edited requires fresh approval if it's a write step, even if it was previously approved.
**Owner:** Jey. *(Recommendation: yes, this is the safer default.)*
**[VALIDATION-PENDING]**

### 9.3 Pydantic additions

```python
class ModificationType(str, Enum):
    SKIP = "skip"
    EDIT_PARAMS = "edit_params"
    ABORT = "abort"

class ModificationRecord(BaseModel):
    timestamp: datetime
    type: ModificationType
    step_index: int
    actor: str = "operator"
    reason: Optional[str] = None
    diff: Optional[Dict] = None  # for edit_params: {before, after}

class RunState(BaseModel):
    # ...existing fields...
    modification_log: List[ModificationRecord] = []
```

### 9.4 API additions

| Method | Path | Body | Notes |
|---|---|---|---|
| POST | `/api/run/{run_id}/step/{step_index}/skip` | `{ reason?: string }` | Only valid in PAUSED state. Step must be pending. |
| POST | `/api/run/{run_id}/step/{step_index}/edit_params` | `{ params: {...}, reason?: string }` | Only valid in PAUSED state. Re-runs WRITE_TOOLS check. May reset approval. |
| POST | `/api/run/{run_id}/abort` | `{ reason?: string }` | (Already exists from Phase 1 — hardened here to write modification record.) |

### 9.5 WRITE_TOOLS re-validation contract

```python
def edit_step_params(run_state, step_index, new_params):
    if run_state.status != RunStatus.PAUSED:
        raise InvalidTransition()
    step = run_state.runbook.steps[step_index]
    if step.status != StepStatus.PENDING:
        raise InvalidTransition()

    # Deterministic re-check
    if step.tool_name in WRITE_TOOLS:
        # Force re-approval
        step.requires_approval = True
        step.approval_state = ApprovalState.PENDING

    diff = {"before": step.params.dict(), "after": new_params}
    step.params = step.params.__class__(**new_params)

    run_state.modification_log.append(ModificationRecord(
        timestamp=datetime.utcnow(),
        type=ModificationType.EDIT_PARAMS,
        step_index=step_index,
        diff=diff,
    ))
    await self._emit("step_modified", {...})
```

**Invariant:** edit_params on a write step ALWAYS resets approval. This is a non-negotiable contract.

### 9.6 Frontend additions

While in PAUSED state, the next pending step card shows three controls:

```
┌─ Step 4: Restart upload service (NEXT) ───────────────────┐
│ Tool: kube_restart_deployment    [requires approval]      │
│ Params:                                                   │
│    namespace: cds-prod                                    │
│    deployment: doc-uploader                               │
│                                                           │
│ [✏ Edit params] [⏭ Skip step] [⏹ Abort run]              │
└───────────────────────────────────────────────────────────┘
```

Edit form is a generic Pydantic-driven JSON editor (params field shapes drive the form). Save validates, fires API call, emits `step_modified`.

### 9.7 SSE event additions

| Event | Payload |
|---|---|
| `step_skipped` | `{ step_index, reason }` |
| `step_modified` | `{ step_index, diff, requires_reapproval }` |

### 9.8 Acceptance criteria

- During PAUSED, next-pending-step card shows skip / edit / abort controls.
- Skipping advances `current_step_index` past the skipped step; step status becomes SKIPPED.
- Editing params on a non-write step persists changes; resume executes with new params.
- Editing params on a write step persists changes AND resets approval to PENDING.
- Modification log records every action with timestamp, actor, reason, diff.
- Modification log is visible in UI (collapsible per-run history).
- Resume after edit re-validates runbook version (no stale execution).

---

## 10. Phase 4 (Deferred) — T2 Modification

Out of scope for this spec. Outline only:

- Add step (from typed template library).
- Remove step (with dependency validation).
- Reorder steps (with dependency validation).

Will require: dependency graph in runbook (today: linear), step template registry, more sophisticated WRITE_TOOLS re-validation across the modified runbook.

**Pre-condition for Phase 4 spec:** T1 has been operationally exercised in at least 5 demo runs without correctness issues, and the modification log structure has proven sufficient for audit review.

---

## 11. Cross-cutting concerns

### 11.1 Demo mode

All write operations remain demo-mocked under `settings.demo_mode = True` regardless of phase. Phase 3's `edit_params` on a write step still hits the demo-mocked RemediationAgent path; the WRITE_TOOLS re-check enforces structure even when the underlying call is mocked.

### 11.2 Persistence interaction

This spec assumes in-memory RunState. When SQLite persistence lands (separate workstream):
- New status values (`AWAITING_PLAY`, `PAUSED`, `ABORTED`) need migration support.
- `modification_log` becomes a persisted table (one-to-many to runs).
- `tool_calls` (Phase 2 storage) becomes a persisted table.
- Pause-then-server-restart becomes recoverable.

Design the schema additions in Phase 1 / 2 / 3 with persistence in mind: avoid Python-only types, prefer JSON-serializable structures throughout.

### 11.3 Audit / compliance posture

After Phase 3 ships, AEGIS provides for any incident run:
- Full runbook (original + modified versions via diff log).
- Every tool call (input + output) made by every agent.
- Every operator action (play / pause / resume / skip / edit / abort) with timestamps and reasons.
- Approval / decision trail.

This is materially stronger than the current "the system did something, check Splunk" audit posture. It is the foundation for the SR 11-7 model risk story Jey will need for production deployment.

---

## 12. Risk Register

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | Phase 0 introduces module-loading bug that breaks demo | Low | High | Migrate one component end-to-end first; smoke-test demo before full migration |
| R2 | Phase 1 pause semantics confuse operators ("why won't it stop now?") | Medium | Medium | UI shows "Pausing after current step…" with progress indicator; document contract |
| R3 | Phase 1 race between pause and approval | Low | Medium | Approval gate wins; pause button disabled while approval modal open |
| R4 | Phase 2 SSE payload bloat from large Splunk results | Medium | High | Summary-in-SSE + on-demand-fetch architecture (designed in 8.4.3) |
| R5 | Phase 2 leaks PII via tool output display | Medium | Critical | BV-2.2 validation + redaction layer if PII present; no shipping until validated |
| R6 | Phase 3 edit_params bypasses WRITE_TOOLS check | Low | Critical | Hard invariant in code (9.5); unit test coverage required |
| R7 | Phase 3 edit on a step with stale runbook version | Low | High | Resume revalidates version; force re-acknowledgment if changed |
| R8 | Modification log grows unbounded for long-running runs | Low | Low | Acceptable for v1; revisit when persistence lands |
| R9 | Frontend modularity drifts back toward monolith | Medium | Medium | Lint rule / convention: no component > 200 lines; review at each phase |
| R10 | F1 (asymmetric WRITE_TOOLS) is real and Phase 3 inherits the gap | Medium | Critical | CV-2.5 explicitly verifies; if gap real, fix BEFORE Phase 3 |

---

## 13. Known Unknowns

- Exact behavior of MCP servers under load when pause arrives mid-step (BV-1.1).
- Whether Aayushi's demo scenarios need re-scripting for play button (SV-1.4).
- Whether existing tool detail modal handles all output shapes Phase 2 will surface (CV-2.4).
- Whether in-memory `asyncio.Event` survives the eventual SQLite persistence transition gracefully.
- Frontend behavior under multi-tab usage with conflicting controls.

---

## 14. Architecture Decision Records

### ADR-01: Manual play, no auto-play in v1

**Context:** Post-planning state could default to AWAITING_PLAY (manual) or AUTO_PLAY (current behavior).
**Decision:** AWAITING_PLAY only. No auto-play option.
**Rationale:** The whole point of the change is to put the operator in the loop. An auto-play default reintroduces the bystander problem with worse UX.
**Consequences:** Demo flow includes a play click. Aayushi's scenarios must accommodate this.
**Status:** Accepted.

### ADR-02: Pause at step boundary, not mid-step

**Context:** Pause could attempt to cancel in-flight tool calls or wait for the current step to complete.
**Decision:** Pause takes effect at next step boundary. No mid-step cancellation.
**Rationale:** Mid-step cancellation creates half-applied state risk. Splunk queries are typically <30s; UX impact bounded.
**Consequences:** UI must communicate "Pausing after current step…" clearly.
**Status:** Accepted.

### ADR-03: T1 only in v1, T2 deferred

**Context:** Modification capability is a continuum (T1 → T2 → T3).
**Decision:** Ship T1 (skip / abort / edit_params) only. Defer T2 (add / remove / reorder). Do not pursue T3 (free-form).
**Rationale:** T1 captures the most common "deviate from runbook" operator need without bringing the LLM back into execution. T2 needs a dependency-graph runbook representation. T3 reintroduces ReAct-shaped risk.
**Consequences:** Some operator scenarios (insert new step) are unsupported in v1.
**Status:** Accepted.

### ADR-04: WRITE_TOOLS re-check on every edit_params

**Context:** Should edit_params on a step that was previously approved retain the approval?
**Decision:** No. Edit_params on a write step always resets approval to PENDING.
**Rationale:** Edited params change the blast radius of the call. Permission-laundering attack surface if approval persists across edits.
**Consequences:** Operators must re-approve after every edit; intentional friction.
**Status:** Accepted.

### ADR-05: Single-file modal reuse over per-context modals

**Context:** Tool detail modal is used in three contexts (gather, pattern provenance, agent tool call).
**Decision:** Single modal with a `mode` prop, not three separate modals.
**Rationale:** Output rendering logic is mostly shared. DRY beats per-context customization at this scale.
**Consequences:** Modal component is slightly more complex; rendering branches by mode.
**Status:** Accepted.

### ADR-06: SSE summaries + on-demand full output

**Context:** Tool outputs can be large (Splunk results in 100+ KB range).
**Decision:** SSE events carry summaries only. Full output retrieved via `GET /api/run/{id}/tool_call/{call_id}`.
**Rationale:** Avoids SSE bloat, reduces frontend memory pressure, gives natural caching point.
**Consequences:** One extra round trip per modal open. Acceptable.
**Status:** Accepted.

### ADR-07: Alpine.data() components, no build step

**Context:** Modularity could be achieved via SPA framework with build, native Web Components, or Alpine.data() split.
**Decision:** Alpine.data() components in separate JS files.
**Rationale:** Preserves no-build-step constraint, idiomatic for current stack, sufficient for project scale.
**Consequences:** Locks in Alpine for the medium term; SPA migration is a separate decision later if needed.
**Status:** Accepted.

---

## 15. Out of Scope (Explicit)

- T2 / T3 runbook modification.
- Auto-play option.
- Pause timeout / abandonment escalation.
- Mid-step cancellation.
- SQLite persistence (separate workstream; this spec designs for compatibility but does not implement).
- AEGIS Knowledge Graph integration.
- Multi-user concurrent operator scenarios (last-writer-wins is acceptable v1).
- PII redaction beyond minimum needed if BV-2.2 surfaces a real risk.
- Keyboard shortcuts for play/pause/skip.
- Mobile / tablet UX.

---

## 16. Glossary

| Term | Meaning |
|---|---|
| **AWAITING_PLAY** | Post-planning state. Runbook generated, awaiting operator play. |
| **PAUSED** | Execution paused mid-run, between steps. |
| **PauseReason** | Why execution is paused: USER_REQUESTED / APPROVAL_GATE / DECISION_GATE. |
| **WRITE_TOOLS** | Constant list of MCP tool names that perform writes. Deterministic permission gate. |
| **ModificationRecord** | Audit-log entry for any operator modification to the runbook or run state. |
| **T1 / T2 / T3** | Tiers of runbook modification capability (bounded / structured / free-form). |
| **F1–F5** | Findings from architecture diagram review (2026-04-28). |

---

*End of specification. Pending: pre-validation answers in [VALIDATION-PENDING] blocks before Phase 0 implementation begins.*
