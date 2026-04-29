# AEGIS Execution Control & Provenance — Vibe Coding Guide

**Companion to:** `AEGIS-EXECUTION-CONTROL-SPEC.md`
**For:** GitHub Copilot (VS Code, Opus 4.6) or equivalent IDE agent
**Reading order:** Read the spec first. Read this second. Do not skip the spec.

---

## How to use this guide

Each phase is a sequence of **sessions**. Each session has:

- **Preconditions** — what must be true before starting
- **Goal** — single-sentence outcome
- **Prompt pattern** — what to feed the IDE
- **Quality gates** — automated and manual checks before exit
- **Red flags** — signs to stop
- **Verification commands** — bash to run

**Session 1 of every phase is mandatory pre-validation.** Implementation does not begin until every `[VALIDATION-PENDING]` block in the spec for that phase has been replaced with an answer or an explicit `DEFERRED` / `BLOCKED` note.

If the IDE proposes to skip pre-validation, that is the single biggest red flag in this entire guide. Stop. Restart. Re-feed the spec.

---

## Phase 0 — Minimal Modularity

**Estimated:** 2–3 sessions. **Calendar:** 2–3 days.

### Session 0.1 — Pre-validation

**Preconditions:**
- Spec file open in workspace.
- Working AEGIS branch checked out.

**Goal:** Fill every `[VALIDATION-PENDING]` block in spec sections 6.3 with an answer.

**Prompt pattern:**

```
You are validating assumptions in AEGIS-EXECUTION-CONTROL-SPEC.md
Section 6.3 (Phase 0 Pre-Development Validation).

For each [VALIDATION-PENDING] block:
1. Read the relevant code or run the probe described.
2. Replace [VALIDATION-PENDING] with [ANSWER: <evidence>].
3. If you cannot answer, replace with [BLOCKED — reason].

Do NOT begin implementation. Your only deliverable is updated
spec validation blocks. Show me the diff before saving.

Start with CV-0.1 and proceed in order.
```

**Quality gates:**
- Every Phase 0 `[VALIDATION-PENDING]` is resolved.
- Every answer cites evidence (file path + line, or probe output).
- SV-0.1 (single-file deployment constraint) has been confirmed by Jey in writing.

**Red flags:**
- IDE replaces a pending block with "assumed correct" or "appears to be" — reject. Make it cite.
- IDE proposes to "validate during implementation." No. Halt.

**Exit criterion:** Diff applied to spec, validation answers committed.

---

### Session 0.2 — Extract pure-function libraries

**Preconditions:** Session 0.1 complete.

**Goal:** Extract `lib/format-utils.js` and `lib/api.js` from `index.html`.

**Prompt pattern:**

```
Read AEGIS-EXECUTION-CONTROL-SPEC.md section 6.4 (architecture)
and section 6.7 (implementation order).

Step 1: Identify all pure utility functions in frontend/index.html
(formatToolOutputRich, highlightYaml, formatDecisionAnalysis, etc.).
List them with their dependencies.

Step 2: Identify all fetch() calls. List endpoints and their callers.

Step 3: Show me the plan before extracting. I will approve it.

Step 4 (after approval): Create frontend/lib/format-utils.js and
frontend/lib/api.js. Move the identified functions. Add <script>
tags to index.html in the correct load order.

Constraints:
- No build step.
- ES module syntax only if it works without a bundler in modern browsers
  for our setup; otherwise use IIFE / global namespace.
- Confirm BV-0.1 result before choosing.
- Run the existing demo path manually after — no functional change.
```

**Quality gates:**
- `wc -l frontend/index.html` shows reduction proportional to extracted code.
- All functions in extracted files are pure (no Alpine state access, no DOM mutations except via inputs).
- Existing demo flow runs without console errors.
- Network tab in browser shows new JS files loading.

**Verification commands:**
```bash
# Line count reduction
wc -l frontend/index.html frontend/lib/*.js

# Search for orphaned references
grep -r "formatToolOutputRich\|highlightYaml" frontend/

# Run smoke test
python -m backend.main &
curl -s http://localhost:8000/ | grep -c "format-utils.js"
```

**Red flags:**
- Function moved breaks because it relied on Alpine's `this.something`. Revert; extract differently.
- IDE suggests adding npm / package.json. Reject.

---

### Session 0.3 — SSE client + component extraction

**Preconditions:** Session 0.2 complete.

**Goal:** Build `components/sse-client.js` with CustomEvent dispatch. Migrate `tool-detail-modal.js` end-to-end as the pattern validator. Migrate remaining components.

**Prompt pattern:**

```
Read spec sections 6.4, 6.5, 6.6, 6.7.

Step 1: Build frontend/components/sse-client.js. It should:
- Open one EventSource per run_id.
- For every SSE event type (15 currently), dispatch a CustomEvent
  on window with name `aegis:${event_type}` and the payload as detail.
- Expose a small API: connectSSE(runId), disconnectSSE().
- Replace the connectSSE() function in index.html by calling this module.

Step 2: Verify all 15 events fire by triggering a demo run and logging
window-level CustomEvents.

Step 3: Extract tool-detail-modal.js as Alpine.data('toolDetailModal',
() => ({...})). Subscribe to relevant CustomEvents in its init().

Step 4: After tool-detail-modal works, extract in this order:
incident-trigger, runbook-view, step-cards, approval-modal,
decision-modal.

For each component:
- Keep under 200 lines.
- Pure Alpine.data registration. No imports from other components.
- Components communicate ONLY via CustomEvents on window.

Show me the diff per component. Do not bulk-migrate.
```

**Quality gates:**
- `frontend/index.html` is < 400 lines.
- Each component file is < 200 lines.
- Browser DevTools shows `aegis:*` CustomEvents firing during a demo run.
- All 15 SSE events are dispatched (validated against spec finding F2).
- Demo runs end-to-end with zero console errors.

**Verification commands:**
```bash
# Component file sizes
wc -l frontend/components/*.js

# Find any direct cross-component imports (should be none)
grep -r "from '\\.\\./" frontend/components/

# Confirm SSE event count matches spec
grep -c "dispatchEvent" frontend/components/sse-client.js
# Expected: 15+ (or whatever current count is, document if mismatched)
```

**Red flags:**
- A component imports another component directly. CustomEvents only.
- IDE refactors business logic during extraction. Reject — extraction is a move, not a rewrite.
- A component exceeds 200 lines. Re-split.

**Exit criterion:** Phase 0 acceptance criteria (spec section 6.8) all pass. Commit. Tag.

---

## Phase 1 — Execution Control Plane

**Estimated:** 6–7 sessions. **Calendar:** 1.5–2 weeks.

### Session 1.1 — Pre-validation

**Preconditions:** Phase 0 tagged.

**Goal:** Resolve every `[VALIDATION-PENDING]` in spec section 7.3.

**Prompt pattern:**

```
Validate spec section 7.3. Same rules as Session 0.1:
- Every block resolved with evidence.
- BV-1.1 and BV-1.2 require running probes — write the probe scripts,
  run them, paste output into the spec.
- SV-1.2 and SV-1.4 — flag for Jey, do not invent answers.

Special attention:
- CV-1.3 (approval gate mechanism). The pause/resume implementation
  in later sessions copies this pattern. If we get this wrong here,
  Phase 1 implementation is wrong.
- BV-1.1 (mid-step pause behavior). If the run aborts on SSE
  disconnect today, we have a different problem than this spec
  assumes. Halt and reconvene with Jey if so.

Halt before implementing anything.
```

**Quality gates:**
- All Phase 1 validation blocks resolved.
- BV-1.1 probe output captured and inspected.
- BV-1.2 step duration distribution recorded (informs UI feedback design).

**Red flags:**
- BV-1.1 reveals run dies on SSE disconnect — this is a separate fix; halt.
- CV-1.3 reveals approval uses polling, not asyncio.Event — flag as design implication.

---

### Session 1.2 — Pydantic schema additions

**Preconditions:** Session 1.1 complete.

**Goal:** Add new `RunStatus` values, `PauseReason`, new `RunState` fields. Tests pass.

**Prompt pattern:**

```
Read spec section 7.5.

Update backend/models/schemas.py:
1. Add RunStatus.AWAITING_PLAY, .PAUSED, .ABORTED.
2. Add PauseReason enum.
3. Add RunState fields: pause_requested, pause_reason,
   current_step_index, play_event, abort_event.

asyncio.Event cannot be a Pydantic field directly — wrap with
arbitrary_types_allowed=True or use a private attribute initialized
in a model_validator. Choose the pattern that matches existing
RunState construction.

Run existing tests (test_agents, test_executor, test_schemas).
All must still pass — this change is purely additive.

Show me the diff before applying.
```

**Quality gates:**
- All existing tests pass.
- New enum values are reachable from existing code.
- `RunState()` constructs without error.

**Verification commands:**
```bash
cd backend
pytest tests/test_schemas.py tests/test_agents.py tests/test_executor.py -v
python -c "from backend.models.schemas import RunStatus, PauseReason, RunState; print(list(RunStatus))"
```

**Red flags:**
- Existing test fails — additive change broke something. Investigate before proceeding.
- IDE renames existing enum values. Reject.

---

### Session 1.3 — Executor refactor

**Preconditions:** Session 1.2 complete. `[BREAKING CHANGE WARNING]` — this session changes the executor entry point.

**Goal:** Refactor `Executor.run()` into `Executor.start()` + background `_execution_loop`. Add `play()`, `pause()`, `abort()` methods.

**Prompt pattern:**

```
Read spec section 7.6 carefully. The pseudocode there is the
target shape, not literal copy-paste.

Current state: Executor.run() is a synchronous-style async loop.
Target state: Executor.start() launches a background task that
waits on play_event between steps.

Implementation steps:
1. Rename run() to _execution_loop(). Keep its current behavior.
2. Add start() that creates background task and sets AWAITING_PLAY.
3. Insert pause checkpoint at top of step loop:
     if run_state.pause_requested: enter_pause; await play_event.
4. Add play(), pause(), abort() public methods.
5. Update main.py /api/trigger to call start() not run().

Critical:
- DO NOT touch the existing approval gate logic. Pause/resume coexists.
- Pause takes effect at step boundary only — verify no checkpoint
  is added INSIDE _execute_step.
- abort_event setting must unblock any waiter (set play_event too).

Show me the full executor.py diff before applying.
After applying, run the existing demo end-to-end. It should now
hang at AWAITING_PLAY (no play API yet — that's Session 1.4).
This is expected. Confirm by checking RunState.status in logs.
```

**Quality gates:**
- Existing executor tests pass (with adjusted expectations for non-blocking start()).
- Demo run reaches AWAITING_PLAY and stops there.
- No mid-step pause checkpoint exists.
- abort_event correctly unblocks waiters.

**Verification commands:**
```bash
pytest backend/tests/test_executor.py -v
# Trigger a run, observe state transitions
curl -X POST http://localhost:8000/api/trigger -d '{...}'
# Wait, then check run state
curl http://localhost:8000/api/run/<id>
# Should show status: "awaiting_play"
```

**Red flags:**
- IDE adds a pause check inside `_execute_step` — reject.
- IDE changes the approval gate's `_wait_for_approval` — reject.
- Existing tests broken in non-trivial way — investigate before proceeding.

---

### Session 1.4 — API endpoints

**Preconditions:** Session 1.3 complete.

**Goal:** Add `POST /api/run/{id}/play`, `/pause`, `/abort`. SSE events fire correctly.

**Prompt pattern:**

```
Read spec section 7.8 and 7.7.

Add three FastAPI route handlers to backend/main.py:
- POST /api/run/{run_id}/play  → executor.play(run_state)
- POST /api/run/{run_id}/pause → executor.pause(run_state)
- POST /api/run/{run_id}/abort → executor.abort(run_state)

Each endpoint:
- Validates run_id exists.
- Validates current state allows the transition.
- Returns 409 Conflict on invalid transition with informative body.
- Returns current RunState snapshot on success.

Add SSE event emission in the executor at the points specified:
- runbook_ready_for_execution: when start() sets AWAITING_PLAY.
- execution_paused: in _enter_pause().
- execution_resumed: in play().
- run_aborted: when abort_event triggers status = ABORTED.

While here, also audit existing SSE events for finding F2:
does planning_started fire? If not, add it at the start of
generate_runbook(). Document the result.

Show me each handler diff individually.
```

**Quality gates:**
- All three endpoints return correct snapshot on happy path.
- All three return 409 with body on invalid transition.
- All four new SSE events fire in correct order during a demo run.
- F2 finding resolved (planning_started either fires or is documented as not-needed).

**Verification commands:**
```bash
# Trigger run, then exercise control plane
RUN_ID=$(curl -s -X POST http://localhost:8000/api/trigger ... | jq -r .run_id)
sleep 5  # let planning complete
curl -X POST http://localhost:8000/api/run/$RUN_ID/play
# Monitor SSE in another terminal
curl -N http://localhost:8000/api/stream/$RUN_ID
```

**Red flags:**
- Endpoint returns 200 on invalid transition — reject.
- IDE adds idempotency tokens or rate limiting — out of scope, reject.

---

### Session 1.5 — Frontend control plane

**Preconditions:** Session 1.4 complete. Phase 0 modular frontend in place.

**Goal:** Update `runbook-view.js` to render play/pause/resume/abort buttons. Subscribe to new SSE events.

**Prompt pattern:**

```
Read spec section 7.9.

Update frontend/components/runbook-view.js:
1. Subscribe to aegis:runbook_ready_for_execution → render Play button.
2. Subscribe to aegis:execution_resumed → swap to Pause button.
3. Subscribe to aegis:execution_paused → swap to Resume + Step Forward + Abort.
4. Subscribe to aegis:run_aborted → render aborted state.

Buttons hit the new API endpoints via lib/api.js helpers.

Status banner above runbook reflects current state literally:
"Awaiting your play" / "Running step X of Y" / "Paused after step X" /
"Aborted by operator".

While Pause is in transit ("Pausing after current step…"), disable
the button and show a spinner.

Disable Pause button when an approval modal is open (per ADR-03).

Step cards show position (Step X of Y) and dim pending steps.

Test manually:
- Trigger run → reaches Awaiting your play.
- Click Play → step 1 runs, then 2, etc.
- Click Pause during step 3 → banner shows Pausing…, then Paused.
- Click Resume → step 4 runs.
- Click Abort from Paused → run ends.
```

**Quality gates:**
- All user flows from Phase 1 acceptance criteria (spec 7.11) pass manually.
- No console errors.
- Multi-tab test: open run in two tabs, both reflect state changes.

**Red flags:**
- Button state flickers — likely a race between optimistic UI update and SSE event.
- Pause button enabled while approval modal open — fix immediately.

---

### Session 1.6 — Failure mode hardening

**Preconditions:** Session 1.5 complete.

**Goal:** Implement the mitigations from spec section 7.10. Add tests.

**Prompt pattern:**

```
Read spec section 7.10.

For each failure mode listed, write a test or implement the
mitigation if not already present. Specifically:

1. Test: Pause requested during step → step completes, then pause.
2. Test: Pause + approval gate race → approval wins; pause queues.
3. Test: Play twice rapidly → idempotent, single execution.
4. Test: Abort from any active state → run ends.
5. UI: Disable Pause button while approval modal open.
6. UI: Show "Pausing after current step…" with spinner during transit.

Add these to backend/tests/test_executor_control.py (new file).

Show test list before writing.
```

**Quality gates:**
- Six tests pass.
- Manual UI test of each scenario.

**Verification commands:**
```bash
pytest backend/tests/test_executor_control.py -v
```

---

### Session 1.7 — Phase 1 integration & demo rehearsal

**Preconditions:** Sessions 1.1–1.6 complete.

**Goal:** End-to-end demo rehearsal of Phase 1.

**Prompt pattern:**

```
Run all of these scenarios end-to-end. Document results.

Scenario A — Happy path manual control:
1. Trigger document upload failure incident.
2. Wait for AWAITING_PLAY.
3. Press Play. Verify step 1 runs.
4. Press Pause during step 2. Verify pauses after step 2.
5. Press Resume. Verify step 3 runs.
6. Run completes with RESOLVED.

Scenario B — Abort path:
1. Trigger incident.
2. Pause after step 2.
3. Press Abort. Verify run ends with ABORTED.

Scenario C — Approval coexistence:
1. Trigger incident with a write step.
2. Pause is requested during the write step's gather phase.
3. Approval modal appears.
4. Pause button disabled.
5. Approve. Step executes. Then pause takes effect.

Capture screenshots of UI state at each phase. Save to dev_docs/.

If any scenario fails, halt and report.
```

**Exit criterion:** Phase 1 acceptance criteria (spec 7.11) all pass. Commit. Tag.

---

## Phase 2 — Provenance Layer

**Estimated:** 4–5 sessions. **Calendar:** ~1 week.

### Session 2.1 — Pre-validation

**Preconditions:** Phase 1 tagged.

**Goal:** Resolve `[VALIDATION-PENDING]` in spec section 8.3. **Critically: validate F1, F3, F5.**

**Prompt pattern:**

```
Validate spec section 8.3.

Special attention to F1 (CV-2.5), F3 (CV-2.6), F5:
- F1: Read backend/engine/agents.py RemediationAgent.run().
  Confirm WRITE_TOOLS check is present BEFORE the tool call,
  not just after human approval. If absent: file a separate
  corrective ticket. Do not silently fix here. Halt Phase 2
  until decided with Jey.
- F3: Read each MCP wrapper. Confirm error → structured dict,
  not raise. Document any that raise.
- F5: Confirm domain_kb retrieval has no SSE event today.

BV-2.1: instrument one demo run, capture max tool output size.
BV-2.2: sample 3 outputs for PII. If any found, halt for
redaction scope discussion with Jey.
```

**Quality gates:** Same as before. Halt on F1 absent or PII present.

---

### Session 2.2 — Backend provenance enrichment

**Preconditions:** Session 2.1 complete.

**Goal:** Enrich `pattern_matched` event. Wrap `MCPClientPool.call_tool()` for per-call events. Add on-demand fetch endpoint.

**Prompt pattern:**

```
Read spec sections 8.4.1, 8.4.2, 8.4.3, 8.4.4.

Step 1: Enrich pattern_matched event. Update wherever it's emitted
to include pattern_id, pattern_confidence, app_filter, rendered_query.

Step 2: Wrap MCPClientPool.call_tool() to emit agent_tool_started
and agent_tool_completed. The wrapper signature accepts run_state
and agent_name kwargs. Existing callers pass these.

Update DiagnosticAgent.run() and RemediationAgent.run() to pass
run_state and self.__class__.__name__ when invoking call_tool.

Step 3: Add storage:
- run_state.tool_calls: Dict[str, ToolCallRecord] = {}

ToolCallRecord schema in models/schemas.py:
- call_id, started_at, completed_at, agent, server, tool_name,
  params, result (full), degraded.

Step 4: Add GET /api/run/{run_id}/tool_call/{call_id} endpoint.
Returns ToolCallRecord. 404 if not found.

Step 5: SSE summaries: agent_tool_completed payload contains
summary (truncated stringified result, max 500 chars), size_bytes,
degraded boolean. NOT the full result.

Tests:
- A demo run produces N agent_tool_started and N agent_tool_completed.
- The on-demand endpoint returns full data.
- SSE payload size for agent_tool_completed < 5KB.
```

**Quality gates:**
- SSE event size validated (< 5 KB regardless of underlying output).
- Full output retrieval works.
- Pattern provenance present in pattern_matched event.

---

### Session 2.3 — Frontend modal extension

**Preconditions:** Session 2.2 complete.

**Goal:** Extend `tool-detail-modal.js` with three modes. Build step-card sub-section.

**Prompt pattern:**

```
Read spec sections 8.5.1, 8.5.2, 8.5.3.

Step 1: Update components/tool-detail-modal.js to accept a `mode`:
- 'gather'        — existing behavior
- 'pattern'       — show pattern_id, confidence, app_filter, rendered_query, output
- 'agent_tool'    — show agent name, server, tool, params, output, timing

Output rendering is the same engine across modes (existing
formatToolOutputRich).

Step 2: Step card extension. components/step-cards.js:
- Each step shows a collapsed Tool Calls section.
- On expand, lists each tool call from run_state.tool_calls
  filtered by step_index.
- Each item shows: tool name, status (success/degraded), duration.
- "view" button fetches full record from on-demand endpoint and
  opens modal in 'agent_tool' mode.

Step 3: Gather panel. components/runbook-view.js (or wherever
gather UI lives):
- For each pattern-matched tool call, add "Show query" button.
- Opens modal in 'pattern' mode.

Test:
- Run incident. Expand step 2's tool calls. Click view on one.
  Modal shows full input + output.
- In gather panel, click Show query on Splunk result.
  Modal shows pattern_id, confidence, rendered query.
```

**Quality gates:**
- Modal renders correctly in all three modes.
- Tool calls grouped by step correctly.
- Degraded calls visually distinct.

---

### Session 2.4 — Phase 2 integration

**Preconditions:** Sessions 2.1–2.3 complete.

**Goal:** Demo rehearsal Phase 2. Confirm provenance complete.

**Prompt pattern:**

```
End-to-end test:
1. Trigger document upload failure incident.
2. Watch gather phase. Click Show query on each MCP result.
   Confirm pattern provenance shown.
3. Press Play. Watch each step.
4. Click any tool call within any step. Confirm full input/output.
5. Force a Confluence MCP error (kill connection mid-run if possible).
   Confirm degraded badge appears, run continues.

Capture screenshots of:
- Gather phase with pattern provenance modal open.
- Step card with expanded tool calls.
- Tool detail modal in agent_tool mode.
- Degraded tool call indicator.

Save to dev_docs/.
```

**Exit:** Phase 2 acceptance (spec 8.7) passes. Commit. Tag.

---

## Phase 3 — T1 Modification

**Estimated:** 4–5 sessions. **Calendar:** ~1.5 weeks.

### Session 3.1 — Pre-validation

**Preconditions:** Phase 2 tagged.

**Goal:** Resolve `[VALIDATION-PENDING]` in spec section 9.2.

**Prompt pattern:**

```
Validate spec section 9.2.

CV-3.1 is the most important: read RunbookPlanStep, ExecutionStep
schemas. Confirm params is a structured field (not free-form
string). If free-form, the edit-params UI design changes — halt
and reconvene with Jey.

CV-3.2: locate WRITE_TOOLS exactly. If it's defined in multiple
places, fix that first as a separate change.

SV-3.2 needs Jey's confirmation: edit on a write step always
resets approval. This is a non-negotiable correctness invariant
in our recommendation. Confirm.
```

---

### Session 3.2 — Pydantic + audit log

**Preconditions:** Session 3.1 complete.

**Goal:** Add `ModificationRecord`, `ModificationType`, `RunState.modification_log`.

**Prompt pattern:**

```
Read spec section 9.3.

Update models/schemas.py per the section. Field additions are
backward-compatible (default empty list, optional fields).

Existing tests must pass.
```

---

### Session 3.3 — Skip + abort hardening

**Preconditions:** Session 3.2 complete. Abort already exists from Phase 1; this hardens it with audit logging.

**Prompt pattern:**

```
Read spec section 9.4.

Step 1: POST /api/run/{run_id}/step/{step_index}/skip
- Validate state == PAUSED.
- Validate step is pending.
- Set step.status = SKIPPED.
- Advance current_step_index past skipped step (only if it's the next pending).
- Append ModificationRecord(type=SKIP, step_index, reason).
- Emit step_skipped SSE.

Step 2: Update existing /api/run/{run_id}/abort
- Append ModificationRecord(type=ABORT, step_index=current, reason).

Step 3: Frontend. Update step-cards.js: PAUSED state shows skip
button on next pending step's card. Abort button on the run-level
control already exists from Phase 1.

Test:
- Skip step 3. Resume. Verify steps 1, 2, 4, 5 execute (3 skipped).
- Modification log shows the skip with timestamp + reason.
```

---

### Session 3.4 — Edit params (the critical session)

**Preconditions:** Session 3.3 complete.

**Goal:** Edit params with WRITE_TOOLS re-validation.

**Prompt pattern:**

```
Read spec section 9.5 carefully. The WRITE_TOOLS re-check is a
non-negotiable invariant.

Step 1: POST /api/run/{run_id}/step/{step_index}/edit_params
- Validate state == PAUSED.
- Validate step is pending.
- Construct new params via step.params.__class__(**new_params).
  Pydantic validation enforces shape.
- If step.tool_name in WRITE_TOOLS:
    step.requires_approval = True
    step.approval_state = ApprovalState.PENDING
- Compute diff (before/after).
- Append ModificationRecord(type=EDIT_PARAMS, step_index, diff, reason).
- Emit step_modified SSE with requires_reapproval flag.

Step 2: Tests in backend/tests/test_modification.py:
- Edit non-write step → params change, no re-approval needed.
- Edit write step → params change, approval reset.
- Edit step with invalid params → 400 Bad Request, no state change.
- Edit step that's not pending → 409 Conflict.
- Edit during RUNNING (not PAUSED) → 409 Conflict.

Step 3: Frontend. components/step-cards.js: in PAUSED state,
next-pending-step card shows Edit Params control. Form generated
from Pydantic model schema (introspect via /api/run/.../step_schema
or hardcode forms per step type for v1 — flag as known limitation
if hardcoding).

Critical: confirm test "Edit write step → approval reset" passes
before declaring this session done. This is the single most
important correctness check in Phase 3.
```

**Red flags:**
- IDE suggests trusting client-side params validation only — reject.
- IDE suggests retaining approval across edits as "operator convenience" — reject. Per ADR-04.

---

### Session 3.5 — Modification log UI + Phase 3 integration

**Preconditions:** Session 3.4 complete.

**Prompt pattern:**

```
Step 1: New component components/modification-log.js. Collapsible
panel below the runbook view. Shows full modification_log with
timestamp, type, actor, reason, diff (for edits).

Step 2: End-to-end test:
- Trigger incident.
- Play 2 steps. Pause.
- Skip step 3.
- Edit params on step 4 (non-write). Resume. Verify new params used.
- Pause again at step 5 (write step).
- Edit params on step 5. Verify approval modal appears for re-approval.
- Approve. Run completes.
- Open modification log. Verify all 4 entries (skip, edit-4, edit-5,
  re-approval).

Step 3: Document a 7-step demo flow showing T1 capabilities for
Kunal. Save to dev_docs/aegis-t1-demo-script.md.
```

**Exit:** Phase 3 acceptance (spec 9.8) passes. Commit. Tag.

---

## Cross-phase quality discipline

### After every session

- Run full test suite. Must be green.
- `git diff --stat` to confirm scope.
- Commit with descriptive message referencing spec section.

### After every phase

- Demo rehearsal end-to-end.
- Update `dev_docs/dayN.md` with what shipped.
- Tag the commit (`v0.phase-N`).
- Re-read the spec section. Note any drift between spec and implementation. Update spec or fix drift.

### Universal red flags

| Symptom | Action |
|---|---|
| IDE proposes new dependency | Reject unless explicitly in spec |
| IDE proposes refactor adjacent code | Reject; out of scope |
| IDE suggests skipping a test | Halt; investigate |
| IDE answers a `[VALIDATION-PENDING]` block with hand-wave | Reject; demand evidence |
| Spec and implementation drift mid-session | Stop coding; reconcile |
| WRITE_TOOLS check is bypassed for any reason | Halt; this is the most important invariant |

### Universal verification commands

```bash
# Test suite (after every session)
cd backend && pytest -v --tb=short

# Lint (after every session)
ruff check backend/
ruff check frontend/  # if you have ruff for JS; else use eslint or skip

# Full demo dry run (after every phase)
python -m backend.main &
# Trigger via curl or UI; complete a known-good incident scenario
# end-to-end. No errors, all SSE events firing, all modals work.
```

---

## When to stop and ask Jey

The IDE should **halt and request human input** in these situations:

1. Any pre-validation block returns an answer that contradicts the spec's stated assumption.
2. F1 / F3 / F5 turn out to be real gaps — these are corrective work, not silent fixes.
3. PII detected in tool outputs (BV-2.2) — needs scope expansion for redaction.
4. A test that the spec mandates fails after implementation, and the obvious fix is to weaken the test.
5. WRITE_TOOLS re-validation behavior conflicts with any existing code path.
6. Aayushi's demo scenarios reveal a flow this spec doesn't accommodate.

In each case, the IDE should produce a clear summary of the conflict, its proposed options, and stop. Do not proceed on assumed authority.

---

## Final word

This guide is not a substitute for the spec. The spec contains the architectural reasoning — the why. This guide contains the operational order — the how. If the two ever conflict, the spec wins. Update the guide.

Every session output should leave AEGIS demonstrably better than the last. If a session ends with uncertainty about whether it broke something, that's a failed session — back out, restart.

End of guide.
