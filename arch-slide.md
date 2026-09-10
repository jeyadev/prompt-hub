# Regenerate the architecture exhibit

Drop this at `prompts/arch-slide.md` in the kit repo. Run it in Cline or Copilot agent
mode with the deck file open, or paste it with the deck path appended.

**Invocation:** `Follow prompts/arch-slide.md. Deck: ./docs/aegis-kit-exec-brief.html`

---

## Role

You are updating one exhibit in a management deck. The exhibit must describe the
architecture **as this repository actually implements it today**, not as anyone
intends it to be. A diagram that shows a component we have not built is a governance
problem, not a rounding error.

## Step 1 — Establish ground truth before drawing anything

Read, in this order, and take notes as you go:

1. `AGENTS.md` — the canonical layer: what context it carries, what boundaries it declares.
2. `skills/*/SKILL.md` — the operational tasks the kit actually ships. Count them.
3. Anything that generates or contains a surface adapter (Cline rules/modes, Copilot
   instruction files, and the script or template that produces them). Identify which
   adapters exist and which are generated versus hand-maintained.
4. MCP server configuration — the exact set of servers wired up, and for each one
   whether it is read-only or write-capable.
5. The write-gate implementation — the hook, its allow/deny list, and the point in the
   call path where it fires.
6. Telemetry — what fields are emitted, at what granularity, and where they land.

Label every component you plan to draw:

- `[VERIFIED]` — you read the file that implements it. Name the path.
- `[CONFIGURED]` — it appears in config but nothing implements it yet.
- `[ABSENT]` — talked about, not present.

**Draw only `[VERIFIED]` and `[CONFIGURED]` components.** Draw `[CONFIGURED]` ones with
a dashed stroke. Never draw `[ABSENT]`. List everything you excluded in your reply so
the presenter knows what the exhibit is deliberately not claiming.

## Step 2 — Compose the exhibit

Replace **only** the content between these two markers in the deck file:

```
<!-- ARCH_DIAGRAM_START ... -->
<!-- ARCH_DIAGRAM_END -->
```

Leave the markers, the `<figure>`, the `<figcaption>` and every other line of the file
untouched. If the markers are missing, stop and say so rather than guessing where to write.

The exhibit reads left to right in three bands, with a telemetry rail beneath:

```
  kit repository  →  engineer's IDE agent  ⇒ read →  systems of record
  (canonical layer)  (generated adapters)  ⇒ write ✕ gate
  ────────────────────────────────────────────────────────────────────
  telemetry: what is emitted  →  where it aggregates
```

Keep the existing structure if it still matches the repo. Change what has drifted;
do not restyle what has not.

## Step 3 — Hard constraints

- Root element `<svg viewBox="0 0 1000 460">` with a `role="img"` and an `aria-label`
  that describes the flow in one sentence.
- Colours, exactly these, no others: ink `#17202A`, secondary `#4A5762`, muted `#8A949B`,
  hairline `#C9CEC9`, fill `#E2E5E0`, denied `#8C3A2B`, permitted `#2F5D50`.
- `font-family="Inter, system-ui, sans-serif"` set once on a wrapping `<g>`.
- No text below `font-size="11"`. This is projected in a room, not read on a laptop.
- No `<style>`, no `<script>`, no `<foreignObject>`, no external images, no web fonts,
  no gradients, no drop shadows.
- Maximum 16 labelled nodes. If the architecture needs more, the exhibit is wrong —
  raise the abstraction level rather than shrinking the type.
- Every arrow is labelled with the verb it performs (`render`, `read`, `emit`).
- The write path must remain visually terminated at the gate. It never reaches the
  right-hand band.

## Step 4 — Verify before you finish

Check each of these and report pass or fail:

1. Every drawn node maps to a file path you read. Paths listed in your reply.
2. No node overlaps another; no text extends past `x=992` or below `y=450`. Check the
   long descriptive labels in particular — they are the ones that spill out of their box.
3. The file still parses: open it and confirm the deck renders five slides, arrow keys work.
4. Nothing outside the markers changed. Show the diff summary.
5. The `aria-label` matches what the diagram now shows.

## Step 5 — Report

Reply with, in this order: components drawn with their evidence paths, components
excluded and why, what changed since the previous version of the exhibit, and the one
thing in this architecture you would expect a reviewer to challenge.
