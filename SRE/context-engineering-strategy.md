# Context Engineering Strategy — Cline / Roo Code

**Audience:** Cline / Roo Code, to be absorbed into a workspace's mode personas and rules.
**Goal:** Stop exhausting the context window at session start; make condensing safe; reach top-1% context discipline.
**Scope:** Workspace-agnostic. Drop into any project. Replace `.context/` with your preferred directory if needed; nothing else is project-specific.

---

## 0. The one reframe (read this first)

The conversation context is **working memory** — volatile, expensive, and lossy under compaction. Disk is **long-term memory** — durable, cheap, lossless. The common failure is that extraction and doc-reading land in working memory at session start, where they sit whether or not they're needed downstream, and then get mangled when condensing fires.

**Rule that governs everything below:** *nothing lives in the conversation that a file could hold instead.* The live thread should carry only the active task, a checklist, and pointers to disk. Everything else is retrieved just-in-time.

---

## 1. Diagnosis — the three things that go wrong

- **Front-loaded extraction.** Large docs/schemas/logs are read into the *main* thread before real work begins. This is the single biggest leak.
- **Compaction used as a safety net.** Condensing is lossy and fires on a threshold you don't fully control; it can fire mid-operation and leave the agent "lost" (a documented failure mode in both Cline and Roo). Relying on it to preserve detail is fragile by construction.
- **No retrieval discipline.** Once something is in context it stays. There is no rule that says "read the digest, not the source" or "grep the 12 lines you need, don't full-read the 800-line file."

The "write temp files on condense" instinct targets symptom #2 only, and reactively. The architecture below removes the cause of all three.

---

## 2. Stress-test of the reactive approach

**Common proposal:** write critical data to temp files when condensing is about to happen.

**Why the weak version fails:**
1. **Timing isn't yours.** You can't reliably hook "just before condense." It can fire mid-operation.
2. **You're already too late.** By the time you react, the summarizer has often already decided what to drop.
3. **It competes for the same window.** The save action itself consumes the budget you're trying to protect.

**The strong inversion:** don't save *because* you're condensing — condense freely *because* you already saved. Write to disk **continuously and proactively** so ground truth is always current. Then a compaction event has nothing to lose. This is the antifragile form: the more condensing happens, the cheaper and safer it gets, because the conversation was never the source of truth.

---

## 3. The architecture — four layers

### Layer 1 — Extraction isolation (highest leverage)

Never read large/numerous sources into the main thread. Delegate extraction to a **disposable sub-context** via orchestrator/boomerang. A subtask gets its *own* full window: it can burn most of it reading docs, write a small digest to disk, and return only a pointer summary to the parent. The parent's window never ingests the raw material.

This is the structural fix that makes most of the problem disappear. See the `context-steward` mode in §4.

### Layer 2 — Durable scratch memory on disk

A single canonical home for session memory:

```
.context/
  _index.md          # single entrypoint — the ONLY file auto-read at task start
  state.md           # active task + live checklist (the working-state file)
  decisions.md       # append-only ADR-lite log
  handoff.md         # session-continuity snapshot for /newtask
  digests/           # distilled extractions (output of context-steward)
    <source>.md
```

The steward and work modes write here **as they go**, not at the end. A compaction or a fresh `/newtask` is survivable because `_index.md` + `state.md` reconstitute everything.

### Layer 3 — A condensing prompt that compresses to *pointers*, not prose

This is the trick most setups miss. The condensed summary's job is to become a **table of contents to disk**, not a lossy copy of the work. Customize the condensing prompt (§4) so it preserves the active task, file paths, open decisions, and digest pointers — and replaces any file-derived detail with `See <path> (lines X–Y)`. Detail is recoverable; the summary only has to remember *where*.

### Layer 4 — Lazy, metadata-first retrieval baked into the persona

Discipline that keeps the window lean during work:
- Need info from a source? Check `digests/` first. If a fresh digest exists, use it — do **not** open the source.
- Must touch a source? `grep`/`head`/range-read for the specific lines. Never full-read a file over ~400 lines into context.
- Produced a decision? Append it to `decisions.md` immediately; don't carry it in your head.
- Before any long/risky operation, flush current state to `state.md`.

---

## 4. Paste-ready artifacts

### 4a. `context-steward` mode (add to `.roomodes`)

```yaml
customModes:
  - slug: context-steward
    name: 🗄️ Context Steward
    roleDefinition: >-
      You are a disposable extraction specialist. Your only job is to read
      source material (docs, schemas, logs, code) in an isolated context
      window and distill it into durable, structured digests on disk. You
      never do implementation work. You optimize for exactly one thing:
      producing a digest so good that downstream modes never need to open
      the source.
    whenToUse: >-
      Delegate to this mode via an orchestrator/boomerang subtask whenever a
      task requires reading large or numerous source files. It returns only a
      pointer summary; raw material stays out of the parent context.
    groups:
      - read
      - - edit
        - fileRegex: \.context/.*\.md$
          description: Digest and index files only
    customInstructions: >-
      PROTOCOL:
      1. Read assigned sources fully — you have a throwaway window, spend it.
      2. Write one digest per logical source to .context/digests/<name>.md.
      3. Every digest opens with a provenance header:
         ---
         source: <path>
         extracted: <ISO8601>
         source_hash: <short hash if available>
         ---
      4. Digest body = only what a builder needs: contracts, signatures,
         invariants, gotchas, exact paths and line ranges. No prose padding.
      5. Update .context/_index.md with a one-line entry per digest.
      6. RETURN to the parent ONLY: the list of digest paths plus a 3-line
         summary. NEVER paste source contents back to the parent.
```

### 4b. Custom condensing prompt (paste into Context Management settings)

```
When condensing, do NOT attempt to retain the contents of files, logs, or
extracted documents. Compress them to POINTERS.

Preserve verbatim:
  1. The active task and its checklist.
  2. Every file path touched or referenced.
  3. All open decisions and unresolved questions.
  4. The path to every digest under .context/digests/ with a one-line
     description of each.

For any detail that lived in a file, replace it with: "See <path> (lines X-Y)".
The summary is a table of contents to disk, not a copy of the work. Assume all
ground truth is recoverable from .context/.

End the summary with the current contents of .context/state.md.
```

### 4c. `_index.md` template (the single auto-loaded entrypoint)

```markdown
# Context Index

> The only file auto-read at task start. Everything else is loaded on demand.

## Active state
- See: state.md

## Digests (read these instead of sources)
- digests/<name>.md — <one line> — extracted <date>

## Decisions
- See: decisions.md (append-only)

## Handoff
- See: handoff.md (last session summary)
```

### 4d. Rules block for work modes (Architect / Code — add to `.clinerules` or mode `customInstructions`)

```
CONTEXT DISCIPLINE (non-negotiable):
- At task start, read ONLY .context/_index.md and state.md.
- Before reading any source, check .context/digests/. If a fresh digest exists
  (provenance header matches the source), use it; do NOT open the source.
- If you must touch a source, grep/head/range-read the specific lines. Never
  full-read a file over ~400 lines.
- Any large or multi-file extraction MUST be delegated to context-steward as a
  subtask, never done inline. No exceptions for "small" files.
- Append every decision to decisions.md the moment it is made.
- Before any long or risky operation, write current state to state.md so a
  compaction is survivable.
```

### 4e. Session-start SOP

1. Orchestrator opens; reads `_index.md` + `state.md` only (cheap).
2. Extraction needed → boomerang to `context-steward` (isolated window).
3. Steward writes digests, returns pointers; parent stays lean.
4. Architect/Code works off digests + `state.md`.
5. At a phase boundary → `/newtask`, carrying forward `state.md` and open items via `handoff.md`.
6. If condensing fires, it compresses to pointers (4b) — a non-event.

---

## 5. Tuning knobs

- **Auto-condense ON, threshold ~70–75%.** Counterintuitive but correct: once disk is the source of truth, condense *early and often*. Early condensing is cheap and safe; late condensing is where it gets lost. (Sequence matters — only lower the threshold *after* the disk discipline below is actually in place.)
- **Assign a cheaper/faster model to the condense pass** (Haiku-class). The summary is mechanical; don't pay frontier rates for it.
- **Per-mode models.** `context-steward` = cheap long-context model. `architect`/`code` = your strongest model. The expensive window is reserved for reasoning, not reading.
- **Set a Max Requests limit** on auto-approved actions so a runaway loop can't silently burn the window before you intervene.

---

## 6. The top-1% delta (reference view)

| Tier | Extraction | Main thread holds | Condensing role | Retrieval |
|---|---|---|---|---|
| Typical | Read everything inline | Raw docs + history | Hope it saves you | Full file reads |
| Good | Memory bank + focus chain | Memory bank + work | Summarizes decisions | Re-read memory bank |
| **Top 1%** | **Isolated in disposable sub-contexts** | **Pointers + active task only** | **Compresses to a disk index** | **Lazy, metadata-first, digest-before-source** |

**The four moves that define elite, in order of leverage:**
1. Extraction is structurally isolated — raw material never enters the main thread.
2. Disk is the source of truth, written proactively and continuously.
3. Condensing compresses to pointers, not prose.
4. Retrieval is lazy and metadata-first; a digest is always preferred over its source.

**Staleness guard (the detail most people skip):** digests carry a provenance header (source path + timestamp/hash). A work mode treats a digest as authoritative only if provenance matches the current source; otherwise it re-delegates extraction. This is what keeps the externalized memory from silently drifting from ground truth.

---

## 7. One-line summary to absorb

*The window is working memory, not storage. Isolate extraction into disposable sub-contexts, keep ground truth on disk written proactively, make condensing compress to pointers, and retrieve lazily — then exhausting the window stops being possible, because the thread never holds anything heavy in the first place.*
