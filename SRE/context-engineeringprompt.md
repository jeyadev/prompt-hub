Read @context-engineering-strategy.md once, in full. It defines a context-engineering
setup for this workspace. Your job is to install it, not discuss it.

Execute these steps, then stop:

1. Create the scaffold:
   - .context/_index.md   → use the template in §4c, filled in for THIS repo
   - .context/state.md    → seed with: "## Active task\n(none yet)\n## Checklist\n- [ ] "
   - .context/decisions.md → seed with a header only: "# Decisions (append-only)"
   - .context/handoff.md  → seed with a header only: "# Last-session handoff"
   - .context/digests/    → create the directory with a .gitkeep; do NOT populate it
   Do not create digests — those are produced lazily by the steward at runtime.

2. Install the context-steward mode from §4a into .roomodes (create the file if
   absent, append the mode if it exists; preserve any existing customModes). Keep
   the fileRegex restriction intact.

3. Install the work-mode rules block from §4d into .clinerules (append; don't
   clobber existing rules). This governs Architect/Code modes.

4. Add .context/digests/ to .gitignore if I track generated artifacts separately;
   otherwise leave it tracked. Ask me which I want if unclear.

5. STOP and report back as a checklist the items you CANNOT do yourself because
   they are GUI/settings, not files — I will do these manually:
   - Paste the §4b condensing prompt into Context Management settings
   - Set auto-condense threshold to ~70–75%
   - Assign a cheaper model to the condense pass
   - Set per-mode models (steward = cheap long-context; architect/code = strongest)
   - Set a Max Requests limit on auto-approved actions

Constraints: read the strategy file exactly once. Do not read other repo files
unless a step requires it. Do not summarize the strategy back to me. Output only
what you created/changed plus the manual-settings checklist.
