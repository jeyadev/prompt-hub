Before writing anything, reconstruct what actually happened — don't narrate from memory.

1. EVIDENCE PASS
   - Gather ground truth: git log/diff, files created/modified, test and command outputs,
     and the session transcripts for this session and every child/sub-agent session
     (in Claude Code these are typically JSONL files under ~/.claude/projects/).
   - For each child session, record: the brief it was given, what it returned, and what
     it actually changed. Flag any gap between "reported" and "verifiable".
   - Tag every claim: ✅ verified (point to the evidence) / ◐ reported only / ? inferred.

2. NARRATIVE (plain language, no jargon without a one-line gloss)
   - Goal as originally stated → goal as it ended up (did it drift? why?)
   - 4–7 phases, each: what was attempted, by which session, why, outcome.
   - Dead ends and pivots are mandatory — show what failed and what that revealed.
   - Final state: what works (with proof), what's unverified, open risks, what
     you would do next.

3. UX REPRESENTATION — one self-contained HTML file:
   - Top banner: Goal → Outcome, with an overall status and confidence.
   - A horizontal "route map": the main line runs left to right through the phases;
     dead ends branch off and terminate visibly; pivots bend the line.
   - Swimlanes underneath: one row per session (parent + each child), so I can see
     who did what, in parallel or in sequence, and where handoffs happened.
   - Each node is clickable → expands to: what / why / evidence link / verification tag.
   - A sidebar showing "Success criteria → met / unmet / unverified".
   - Overview first at a glance; details only on click.

Rules: don't hide failures, don't upgrade ◐ or ? claims to ✅, and if a child session's
work can't be reconstructed, say so explicitly rather than filling it in.
