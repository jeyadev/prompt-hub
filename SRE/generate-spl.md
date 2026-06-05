---
description: Generate scoped, cost-aware SPL for a CDS SRE question
---

# /generate-spl

Generate production-grade SPL for the SRE question I give you. Follow this procedure.

1. **Restate the question** as a measurable SRE signal (rate / errors / latency / saturation /
   consistency / budget burn). If it's an SLI, name the good/valid event definition.
2. **Resolve placeholders** from `cds-splunk-data-dictionary.md`. For any value not in the
   dictionary, keep it as `<PLACEHOLDER>` and list what I need to fill — do not invent index,
   sourcetype, or field names.
3. **Pick the cheapest correct path:** `tstats`/accelerated DM if the fields support it, else raw
   search. State which and why.
4. **Apply scoping non-negotiables:** explicit `index=` + `sourcetype=`, bounded time, selective
   filters before the first pipe, percentiles for latency, explicit `span` on any timechart.
5. **Emit the SPL** fenced, then the output contract:
   - what it measures (one line) + good/valid definition if SLI
   - cost note (scan scope, raw vs tstats, acceleration assumption)
   - validation status: `validated via MCP` | `generate-only, NOT run` | `needs dictionary value`
6. **If the Splunk MCP is wired:** validate against the **dev/sandbox** Splunk with a read-only
   role, a tight time window, and report the row count + runtime. Never first-run on prod indexes.
   If MCP is unreachable, say so and mark the artifact `generate-only, NOT run`.
7. **Guardrail stop:** never produce `index=*`, an unscoped or all-time search, and never paste
   raw sensitive payloads (`doc_id` etc.) — aggregate or hash.

Full methodology: see the Splunk SRE Context rule.
