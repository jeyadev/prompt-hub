# Apply prompt — use the sdlc layer to actually improve a repo's own AI-instruction surface

`COPILOT_IMPORT_PROMPT.md` only teaches the assistant the process. This prompt
gives it a concrete job: audit and enhance the *target repo's own* AI-facing
surface — Copilot/Cline custom instructions, prompt files, and hooks — using
that process, and ship real edits instead of a mapping report.

Paste `COPILOT_IMPORT_PROMPT.md` first (or together with this one in the same
message) so the assistant has loaded the entry point and escalation rule
before it starts.

---

```
Task: using the sdlc-* discipline you just loaded, audit and enhance this
repo's own AI-instruction surface. Concretely, that means whichever of these
exist here:

- .github/copilot-instructions.md (repo-wide custom instructions)
- .github/instructions/*.instructions.md (path-scoped instructions)
- .github/prompts/*.prompt.md (reusable prompt files)
- .clinerules/ or equivalent rules files for other AI tools in this repo
- Claude Code hooks / settings.json hook definitions, if present
- Any equivalent "operating contract" layer this repo already has (you may
  have just mapped these during import — reuse that mapping, don't redo it)

Do not stop at producing a mapping/context report — that's input, not the
deliverable. Run the actual process:

1. sdlc-problem-framing: state what "better" means here before touching
   anything — e.g. fewer contradictions with the repo's real conventions,
   clearer trigger conditions, no dangling references, no duplicated facts
   across files. Write this down before editing.
2. sdlc-spec-and-requirements-reasoning: list the specific defects you'll
   fix (contradictions, staleness vs. current code, missing when-NOT-to-use
   guidance, vague/untestable instructions) as a checklist BEFORE editing.
3. sdlc-implementation-discipline: make the edits. Match the existing file's
   voice and structure; don't introduce a new convention when the repo has
   one already. Small, reviewable diffs over one giant rewrite.
4. sdlc-code-review-reasoning: review your own diff as if you were reviewing
   someone else's — check for now-contradictory statements between files you
   touched and ones you didn't.
5. sdlc-quality-bar-and-taste: last pass, only after correctness — is it
   actually clearer, not just longer?
6. sdlc-verification-discipline: state what you expect to be true after the
   edit (e.g. "no file references a path that doesn't exist", "no two files
   claim the same fact differently") and actually check it before calling
   this done.

If this repo has its own change-control/review gate for editing its
instruction surface (you may have identified one during import — e.g. a
CODEOWNERS rule, a governance doc, a required review step), route through it
rather than committing directly. If you're unsure whether one applies, say so
and ask rather than assuming either way.

Deliverable: the actual file edits (or a diff/PR if that's this repo's
convention), plus a short list of what changed and why — not a standalone
analysis document.
```
