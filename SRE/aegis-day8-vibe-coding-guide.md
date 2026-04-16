# AEGIS Day 8+ — Vibe Coding Guide

> **Purpose**: How to integrate the Day 8+ spec (`aegis-day8-prewarm-retrieval-spec.md`) with your existing master spec and drive development in the GitHub Copilot IDE without losing momentum, context, or your mind.
>
> **Audience**: You (Jey), in an IDE session, probably late evening, probably slightly tired.
>
> **Predecessor**: This builds on the Day 1–7 playbook that got you to a working CLI green path.

---

## 0. The one rule above all others

**One file at a time. Tests green before the next file. No exceptions.**

Vibe coding with Copilot breaks down when you let it generate three files at once and you merge them all before running tests. Copilot is very good at confidently writing code that doesn't work together. The discipline that made Day 1–7 ship is the same discipline that will make Day 8+ ship: vertical slices, testable artifacts, commit-as-you-go.

If you remember nothing else from this guide, remember that.

---

## 1. Repo setup — get your workspace in the right state first

Before you open Copilot, do this 10-minute prep:

### 1.1 Place the spec files in the repo

```
sre-aegis/
├── specs/                                  # NEW
│   ├── README.md                           # Index of all specs
│   ├── 00-aegis-mvp-spec.md                # Your existing 8-day MVP spec (master)
│   ├── 01-aegis-day8-prewarm-retrieval.md  # The new spec
│   └── vibe-coding-guide.md                # This file
├── backend/
├── frontend/
├── dev_docs/
└── ...
```

**Why a `specs/` folder**: Copilot's workspace context uses file proximity as a signal. Specs alongside code = Copilot reads them during autocomplete. Specs buried in Confluence = Copilot invents.

### 1.2 Create `specs/README.md`

This is the index Copilot reads first. Short, navigational:

```markdown
# AEGIS Specs

## Active development
- [01-aegis-day8-prewarm-retrieval.md](./01-aegis-day8-prewarm-retrieval.md)
  Day 8+ additive spec: vector store, prewarmer, query patterns,
  runbook retrieval, chatbox UI. Phases A–G.

## Reference (do not modify)
- [00-aegis-mvp-spec.md](./00-aegis-mvp-spec.md)
  Original 8-day MVP spec. Days 1–7 shipped. Day 8 was "integration +
  demo rehearsal" — superseded by spec 01.

## Rules
- Specs are source of truth. Code follows specs, not memory.
- Do not start a new phase until the previous phase is green.
- Update the relevant `dev_docs/dayN.md` at the end of every session.
```

### 1.3 Link the two specs

At the top of `01-aegis-day8-prewarm-retrieval.md`, add this header (if you haven't already — your spec already references the predecessor, just make sure the path is right):

```markdown
> **Predecessor**: [00-aegis-mvp-spec.md](./00-aegis-mvp-spec.md)
> **Status**: Additive. Does not break Day 1–7 work.
> **Gate**: `test_pipeline.py` (CLI green path) must stay passing throughout.
```

At the bottom of `00-aegis-mvp-spec.md` (your master), add ONE line:

```markdown
> **Continuation**: See [01-aegis-day8-prewarm-retrieval.md](./01-aegis-day8-prewarm-retrieval.md) for Day 8+.
```

That's it. Don't rewrite the master. The master described the build; it did its job.

### 1.4 Freeze the master spec

In `00-aegis-mvp-spec.md`, add a note at the top:

```markdown
> **FROZEN as of end of Day 7.** No edits. Continuation lives in spec 01.
```

This matters because Copilot will occasionally try to "help" by editing the master. Don't let it. Historical specs are historical record — they're how future-you reconstructs why past-you made decisions.

### 1.5 Confirm `.vscode/settings.json` or Copilot settings include specs folder

If you use workspace-level Copilot context (Copilot Chat's `@workspace`), confirm the specs folder is not in any exclusion list. Check `.gitignore`, `.copilotignore` if present, and `files.exclude` in VS Code settings.

---

## 2. Session 0 — resolve the 6 open questions (do not skip)

The spec has §15 — "Open questions Jey must resolve before Phase A." These are gates. If you start Phase A without resolving them, you will rebuild later.

Run this as a **single 30–60 minute session, not with Copilot**. Use Claude, use a notebook, use whatever — but these answers are YOURS, not an LLM's:

| # | Question | How to resolve |
|---|---|---|
| 1 | Does `docviewer-be` expose embeddings? | Hit the endpoint. Read the API docs. If unclear, ask the docviewer-be team directly. |
| 2 | `domain kb` folder structure | `tree` it. Screenshot. Paste into Copilot later when editing `loader.py`. |
| 3 | Docker on VDI | Try `docker --version` on the VDI. Fails → ChromaDB only. Works → Qdrant stays optional. |
| 4 | ADFS token TTL | Check `.token_cache.json` contents. Note the `expires_in`. |
| 5 | Aayushi's chatbox field list | Re-read her message. Confirm the 6 fields in §8.2.2 match. If she mentioned more, list them. |
| 6 | Demo date + slot length | Confirm with Kunal. Drives cut order. |

**Output of this session**: a short markdown file `specs/open-questions-resolved.md` with each answer and date. Copilot reads this when implementing embedder, loader, etc.

Example:

```markdown
# Open Questions — Resolved

Resolved: 2026-04-17

| # | Question | Answer |
|---|---|---|
| 1 | docviewer-be embeddings? | No. Use local sentence-transformers. Model pre-downloaded to `C:\models\all-MiniLM-L6-v2`. |
| 2 | domain kb structure | See `specs/domain-kb-tree.md`. Top level: `docs/`, `schemas/`. No `apps/` or `runbooks/` subfolders yet — will create. |
| 3 | Docker on VDI | Not available. ChromaDB embedded only. Qdrant adapter remains stub. |
| 4 | ADFS token TTL | 8 hours. Prewarmer every 10 min is fine; token refresh every morning. |
| 5 | Aayushi's fields | 6 fields confirmed. Added `fid` and `doc_code` as optional — update §8.2.2. |
| 6 | Demo | 2026-05-02, 15-min slot, governance forum audience. → Prioritize phases A-B-C-G. |
```

If your answers diverge from the spec's defaults, **edit the spec** before Phase A. Do not let implementation drift from spec silently.

---

## 3. Session structure — the anatomy of a Copilot working session

Every session follows the same pattern. Don't improvise this. The structure is what keeps Copilot on rails.

### 3.1 Pre-session checklist (2 min)

- [ ] Last session's tests green? (`pytest`) — if not, fix before starting anything new
- [ ] `test_pipeline.py` passes? — non-negotiable
- [ ] Last session's `dev_docs/dayN.md` written?
- [ ] You know which **phase** and which **file within the phase** you're starting

If any of these is no, do not start the new thing. Close the loop first.

### 3.2 Opening prompt to Copilot Chat

Paste this verbatim (fill in the brackets):

```
We're working on AEGIS Day 8+, Phase [A/B/C/...].

Read these files, in order:
1. specs/01-aegis-day8-prewarm-retrieval.md — sections [list sections]
2. specs/open-questions-resolved.md
3. dev_docs/day[N-1].md (previous session)

Do NOT write code yet. Respond with:
(a) Your understanding of this phase's scope and success criteria
(b) The file-by-file implementation order
(c) Any contradictions you found between the spec and existing code
(d) Any assumption you'd make that I should confirm

Wait for my confirmation before writing code.
```

This forces Copilot to load context into its own window before it starts generating. Skip this, and it will hallucinate file contents from memory.

### 3.3 Per-file loop

For each file in the phase:

1. **Prompt**: *"Implement file 1: `<path>`. Follow the spec section [X]. Use existing schemas from `models/schemas.py` and `models/retrieval_schemas.py` — do not invent new ones. Stop after the file is written."*
2. **Review the code yourself** before accepting. Specifically check:
   - Imports only from paths that exist
   - Pydantic models match the spec exactly (field names, types, defaults)
   - No `TODO` / `NotImplementedError` / `pass`-only stubs
   - No mock data hardcoded where real data should flow
3. **Run the test** for that file: `pytest tests/test_<file>.py -v`
4. If green: **commit immediately** with message `phase X: <file> — <one-line summary>`
5. If red: paste the failure back to Copilot. Iterate. Do NOT move on.
6. Repeat.

### 3.4 Mid-phase sanity check

Every 3 files, run:

```bash
pytest                           # all tests
python -m test_pipeline          # the CLI green path — THE non-negotiable gate
```

If `test_pipeline.py` is red, **stop**. Something you just added broke the existing flow. Fix it before continuing. This is exactly what "CLI green path must stay passing throughout" in §11 means.

### 3.5 Phase exit (do this every single phase)

- [ ] All files in the phase's checklist from §17 of the spec are ticked
- [ ] All new tests green
- [ ] `test_pipeline.py` still green
- [ ] `dev_docs/dayN.md` written — what you built, what you learned, what tripped you up
- [ ] `specs/learnings.md` appended if a new learning is generalizable
- [ ] Git tag: `git tag phase-A-complete` (lets you roll back to a known-good state)

### 3.6 Session close

Before closing the IDE:

```bash
git status                 # clean? committed?
pytest                     # all green?
python -m test_pipeline    # CLI still works?
```

If you can't leave with all three clean, you haven't finished the session. Keep going or roll back — don't leave landmines for tomorrow.

---

## 4. The golden prompts

These are the specific prompts that worked for Days 1–7 and will work for Day 8+. Copy them, adapt the bracketed bits, keep the structure.

### 4.1 Start-of-phase prompt

```
Phase [X]: [phase name].

Read spec section §[N] carefully. It defines:
- Module structure
- File-by-file deliverables
- Tests required
- Success criteria

Propose the implementation order for this phase's files.
For each file, tell me:
- What it does (one sentence)
- What it depends on (other files or modules)
- What test validates it

Do NOT write code yet.
```

### 4.2 Per-file prompt

```
Implement [path/to/file.py] per spec section §[N.N].

Constraints:
- Use existing schemas from models/schemas.py and models/retrieval_schemas.py
- Do not invent new settings — use config.py as-is or add per the spec
- No TODOs, no stubs, no NotImplementedError
- Async functions where the spec says async
- Follow existing code style in backend/ (pydantic, type hints, docstrings)

Show me the file. Do not create additional files.
```

### 4.3 Test prompt

```
Write tests/[test_name].py per spec section §[N.N] "Tests".

Requirements:
- pytest style (function-based, not class-based — matches existing style)
- Use fixtures from conftest.py where they exist
- Mock MCP calls unless the test is explicitly an integration test
- At least one happy-path test and one failure-mode test
- Assertions must be specific (no `assert result` — assert the actual shape)
```

### 4.4 Debug prompt

```
The test failed with:

[paste full traceback]

Spec for this file is §[N.N]. Do not rewrite the file from scratch.
Identify the smallest change that makes the test pass while respecting
the spec. Propose the diff. Do not apply it until I confirm.
```

### 4.5 Integration prompt (for editing existing Day 1–7 code)

```
I need to integrate [new module] into [existing file, e.g. strategist.py]
per spec section §[N.N].

RULES:
- The existing test_pipeline.py MUST still pass after your change
- Make the smallest possible diff
- Preserve all existing function signatures unless the spec changes them
- If you're adding a new method, add it; if you're modifying an existing
  method, show me the full before/after

Show me the diff first. Do not apply.
```

### 4.6 "I'm stuck" prompt

```
I've been on this for [X] minutes. Current state:

- What I'm trying to do: [...]
- What I've tried: [...]
- What the error says: [...]
- What the spec says should happen: [...]

Step back. Don't fix the immediate error — tell me if my approach is
even right. Is there a simpler path through the spec that I'm missing?
```

Use this one liberally. It's how you escape rabbit holes.

---

## 5. Anti-patterns — the specific things that will wreck you

These are the failure modes Copilot users hit again and again. Recognize them, kill them on sight.

### 5.1 "Let me write all the files at once"

Copilot will offer. Say no. Even if the code looks right, you lose the ability to locate failures. One file, one test, one commit.

### 5.2 "Just add a TODO and come back to it"

Your Day 7 inventory shows **zero TODOs across all backend files**. That's a rare, valuable property. Preserve it. If Copilot adds `# TODO: implement later`, reject the suggestion and re-prompt with "no stubs allowed."

### 5.3 "Let me just modify this Day 1–7 file quickly"

Any change to `strategist.py`, `executor.py`, `agents.py`, `main.py`, or `client_pool.py` is a landmine. Required for Phase C/D/E, sure — but use the integration prompt (§4.5), review the diff, and run `test_pipeline.py` immediately after.

### 5.4 "Copilot said it was done, let me move on"

Copilot's "done" and your "done" are different things. Done means: test green, existing tests green, committed. Run the pytest yourself. Read the code yourself.

### 5.5 Schema drift

Copilot will occasionally invent a new Pydantic model when one already exists (e.g., make a new `RunbookStep` when `RunbookStep` and `RunbookDocStep` already exist in spec). This is how you get two incompatible models doing similar things. Catch it in code review.

### 5.6 Silent test skipping

Sometimes Copilot writes a test, it's red, and it "fixes" it by changing the assertion to match the bug. Read the diff. If an assertion goes from `assert x == "expected"` to `assert x == "<whatever_was_produced>"`, reject.

### 5.7 Context exhaustion

After about 45 minutes in one chat session, Copilot starts forgetting earlier parts of the conversation. Start a fresh chat at phase boundaries. Re-paste the opening prompt with current context. Don't fight the context window — work with it.

### 5.8 Skipping `dev_docs/dayN.md`

The day-log is what lets you resume tomorrow. Skip it and tomorrow you stare at uncommitted files wondering what you were doing. Five minutes at session end. Every session.

---

## 6. Integration points with existing code — the landmine map

These are the files from Day 1–7 that Day 8+ touches. Each is a place where you must be extra careful.

| Existing file | What Day 8+ changes | Risk |
|---|---|---|
| `backend/config.py` | Adds `vector_store_path`, `embedding_model_name`, `embedding_model_path`, `replay_mode`, `domain_kb_path` | Low — pure addition. Don't reorder existing fields. |
| `backend/main.py` | Adds `/api/intake/parse`, enriches `/api/health`, new SSE event types | Medium — new routes + new SSE events. Existing routes unchanged. |
| `backend/engine/strategist.py` | `gather_intelligence()` and `generate_runbook()` gain retrieval steps | **High** — the heart of the pipeline. Use §4.5 prompt. Run `test_pipeline.py` after every change. |
| `backend/models/schemas.py` | Adds imports from new `retrieval_schemas.py` (or extends) | Low — spec says put new schemas in separate file. |
| `frontend/index.html` | Chatbox, structured preview, retrieval pills, 4 new SSE handlers | Medium — single file, but 1027 lines already. Use anchor comments for Copilot. |
| `backend/mcp/client_pool.py` | Shared with prewarmer — same `.token_cache.json` | Low — no code change, but document the shared cache. |

**The integration rule**: after touching any of these, run `python -m test_pipeline` before moving on. If it fails, your "improvement" just broke the demo. Revert, re-prompt, try again.

---

## 7. Frontend specifically — it's a single 1027-line file

`frontend/index.html` is the biggest integration risk because it's not broken into modules. Copilot will get lost in it. Mitigate with **anchor comments**:

Before editing, add these comments in the HTML:

```html
<!-- ================================================================ -->
<!-- AEGIS DAY 8+ — CHATBOX SECTION — begin                           -->
<!-- Spec: specs/01-aegis-day8-prewarm-retrieval.md §8.2.1            -->
<!-- ================================================================ -->

<!-- ... new chatbox HTML here ... -->

<!-- ================================================================ -->
<!-- AEGIS DAY 8+ — CHATBOX SECTION — end                             -->
<!-- ================================================================ -->
```

Then prompt Copilot:

```
In frontend/index.html, find the comment block "AEGIS DAY 8+ — CHATBOX SECTION".
Implement the chatbox + structured preview per spec §8.2.1 and §8.2.2
BETWEEN those comment markers. Do not modify any HTML outside those
markers. Show me the diff.
```

This gives Copilot a bounded edit surface. Without it, Copilot will "helpfully" refactor your existing markup and break 12 SSE event handlers in the process.

---

## 8. Prewarmer specifically — it runs out-of-process, test it that way

The prewarmer is the one module that does NOT run inside FastAPI. It's a standalone Python entry point (`python -m prewarmer.runner`) driven by Windows Task Scheduler. This has implications:

### 8.1 Do not import FastAPI in prewarmer

The prewarmer shares `config.py` and `mcp/client_pool.py` but MUST NOT import from `main.py` or anything FastAPI-specific. If Copilot adds `from fastapi import ...` anywhere in `prewarmer/`, reject.

### 8.2 Test the runner end-to-end on the VDI before scheduling

```bash
# Manual run — see if it works at all
python -m prewarmer.runner --dry-run   # no upserts, just log

# Real run, single iteration
python -m prewarmer.runner --once

# Check the manifest
cat ~/.aegis/prewarm_manifest.json
```

Only after all three work manually, run `schedule.bat`.

### 8.3 Prewarmer failures must not crash the scheduler

Every job must catch-log-continue. If Splunk MCP is down, the DT job should still run. The manifest records per-job status. Copilot will sometimes write `raise` in a job — reject and ask for graceful degradation.

---

## 9. Demo rehearsal — Phase G is not optional

Phase G in the spec is "demo replay mode + rehearsal." Treat it as a phase, not a nice-to-have. The rehearsal is where you find the bugs that live demos reveal.

### 9.1 Rehearsal protocol

1. Close Copilot. Close the IDE. Close everything except the browser and the FastAPI terminal.
2. Kill the FastAPI process. Restart fresh. (Real demo day, you reboot cleanly.)
3. Run the full demo from chatbox to run completion, exactly as §12 of the spec describes.
4. **Have someone watch you.** Ideally Aayushi or Ashish. Their "wait, what just happened?" is the bug you missed.
5. Note every "huh?" moment.
6. Fix them. Rehearse again.

### 9.2 Specific things to rehearse

- Accidental browser refresh mid-run — does the app recover or does it need a restart?
- ADFS token expired during the run — what does the UI show?
- MCP call returns 0 results — does the pipeline still produce a usable runbook?
- Pattern matcher hits the confidence floor — does the fallback feel broken or graceful?

### 9.3 Presenter notes — write them

A one-page script:

```markdown
# AEGIS Demo — Presenter Script (15 min)

## Setup (pre-demo)
- [ ] Reboot FastAPI: `uvicorn backend.main:app --host 0.0.0.0 --port 8000`
- [ ] Confirm prewarm ran within last 15 min: check footer
- [ ] Confirm ADFS token valid: check `/api/health`
- [ ] Open browser, hard refresh

## Script
1. (0:00) "This is AEGIS — incident orchestration for CDS."
2. (0:30) Open chatbox. Type Aayushi's scenario 1. Click Parse Intent.
3. (0:45) Show structured preview. "Notice it extracted the app name,
   severity, and time window. Still editable."
4. (1:00) Click Run Diagnosis. "Three things happen immediately —"
   point at the three pills as they light up.
5. ...

## Known issues (do not demo these paths)
- Do not refresh the browser mid-run
- Do not click Reject on the first approval — demo-mode doesn't handle
  branch keys cleanly yet
- ...
```

The known-issues list is real. Every demo has them. Write them down so you don't accidentally walk into one.

---

## 10. Post-phase ritual — the 10 minutes that save your next session

After the last file of a phase commits green:

### 10.1 Update `dev_docs/dayN.md`

Template:

```markdown
# Day [N] — Phase [X]: [phase name]

Date: 2026-04-[DD]
Duration: [X] hours

## What got built
- [file 1] — [one sentence]
- [file 2] — [one sentence]
- ...

## Tests added
- [test_file.py] — [what it validates]

## Integration changes
- [existing file] — [what changed, why]

## What worked
- [...]

## What tripped me up
- [...]

## New learnings (→ learnings.md if generalizable)
- [...]

## State of test_pipeline.py
Green / Red / Not run  (must be Green)

## Next session starts with
Phase [X+1], file 1: [path]. Open with prompt §4.1.
```

### 10.2 Update `specs/learnings.md`

Only generalizable learnings. "ChromaDB needs this flag on Windows" = yes. "I forgot a comma" = no.

### 10.3 Tag and push

```bash
git tag phase-[X]-complete
git push
git push --tags
```

You now have a checkpoint to roll back to. Future-you will thank you.

### 10.4 Close browser, close IDE

The phase is done. The temptation to "just start the next phase" because you're in flow — resist it. Starting fresh tomorrow is how you stay at high quality. Tired coding is how TODOs creep in.

---

## 11. The cut order under time pressure

The spec's §11 already defines this, but it bears repeating here because in the moment, under pressure, you need it written somewhere you'll actually look.

**Full plan**: A → B → C → D → E → F → G

**If 5 days remaining**: A → B → C → G (cut D, E, F — accept slower gather, no chatbox, LLM-generated SPL)

**If 3 days remaining**: A → B → G (cut runbook retrieval too — demo just shows richer diagnosis from domain KB)

**If 2 days remaining**: A → G (cut everything except vector store foundation + rehearsal. Domain KB doesn't ship. Fall back to Day 7 behavior for the demo, rehearse the hell out of what you have.)

**If less than 2 days**: Do not ship Day 8+. Rehearse Day 7 as-is. Present a roadmap slide showing Day 8+ as the forward plan.

The cut order is already optimal for demo impact per unit of effort. Do not reorder it under pressure without a specific reason.

---

## 12. Quick-reference — the commands you'll type 100 times

```bash
# Sanity
pytest                              # all tests
pytest tests/test_<file>.py -v      # one file
python -m test_pipeline             # CLI green path — THE gate

# Prewarmer
python -m prewarmer.runner --dry-run
python -m prewarmer.runner --once
cat ~/.aegis/prewarm_manifest.json

# FastAPI (dev)
uvicorn backend.main:app --reload --port 8000

# Git hygiene (end of every file)
git add -p                          # review every hunk
git commit -m "phase X: <file> — <summary>"

# Phase exit
git tag phase-X-complete
git push && git push --tags
```

Put these in a `Makefile` or `.vscode/tasks.json` if you want hotkeys. Not required, but helps under pressure.

---

## 13. If it all goes sideways

### 13.1 Rollback to last known good

```bash
git reset --hard phase-<last-good>-complete
```

You tagged for a reason. Use it.

### 13.2 Delete the vector store and restart

ChromaDB corruption, lock file weirdness, embedding drift — all fixable by:

```bash
rm -rf ~/.aegis/vector_store
python -m prewarmer.runner --once
```

Vector stores are caches. They're meant to be rebuildable.

### 13.3 Copilot is producing garbage

Fresh chat. Re-paste the §4.1 opening prompt. Re-load the spec file. Context was exhausted.

If that doesn't work, the problem is probably you, not Copilot. Take a break. Come back in 20 minutes.

### 13.4 The pipeline works but the demo feels flat

This is a framing problem, not a code problem. Phase G addresses it. If you're still feeling it post-rehearsal, the fix is not more code — it's better narration. Write the presenter script. Practice it out loud.

### 13.5 You're behind schedule

Refer to §11. Cut. Do not try to "just push through" — tired code is the #1 cause of demo-day bugs.

---

## 14. The summary, in case you skim

1. **Place both specs in `specs/`**, add a README index, freeze the master.
2. **Resolve the 6 open questions before Phase A.** Not negotiable.
3. **One file, one test, one commit.** Always.
4. **Use the golden prompts verbatim.** They work.
5. **Run `test_pipeline.py` after every change to Day 1–7 files.** Non-negotiable.
6. **Update `dev_docs/dayN.md` at the end of every session.** Non-negotiable.
7. **Tag at every phase boundary.** Rollback insurance.
8. **Rehearse the demo.** Code that's never demoed is code that will break on demo day.
9. **Cut ruthlessly if time is short.** §11 has the order.
10. **The goal is not the full spec. The goal is a demo Kunal remembers.**

Good luck. Ship it.
