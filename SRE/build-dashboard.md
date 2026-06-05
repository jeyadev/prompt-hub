---
description: Design and generate a CDS SRE dashboard (Dashboard Studio JSON)
---

# /build-dashboard

Design and generate a Splunk dashboard for the requirement I give you. If I haven't given
specifics, collect them with the intake below before building.

## Requirements intake (ask only what I haven't already answered)
1. **Audience & tier** — Tier 1 (exec/on-call glance), Tier 2 (per-service RED), or Tier 3
   (drill/debug)? One dashboard = one job; if I'm asking for all three, propose three dashboards.
2. **Scope** — which CDS services/queues? Whole platform or a subset (e.g. Pithos cutover watch
   = cds-find + cds-store only)?
3. **Signals** — which golden signals / SLIs? (rate, errors, p50/p95/p99, saturation, store-to-find
   consistency, error-budget burn, deploy correlation)
4. **Time default & refresh** — default window and whether it auto-refreshes (and how often).
5. **Format** — Dashboard Studio JSON (default) or Simple XML (only if the dictionary says the
   target app is classic).

## Build procedure
1. **Resolve placeholders** from `cds-splunk-data-dictionary.md`; mark unknowns `<PLACEHOLDER>`.
2. **Lay out by inverted pyramid:** RAG single-values on top, trends in the middle, drill tables
   at the bottom.
3. **One base search + chains.** Build a single `ds.search` base, then `ds.chain` (`extend`) each
   panel off it. Do NOT create N independent searches — it multiplies load on shared indexers.
4. **Inputs:** global time-range input + a service dropdown driving `$service$` (mark `*` as
   higher-cost). Wire drilldowns to set tokens / link to the Tier-3 dashboard.
5. **Thresholds = SLOs.** Pull RAG color boundaries from the dictionary's SLO table, never invent.
6. **Cost controls:** no sub-30s refresh on raw searches; route heavy correctness searches
   (store-to-find) through a scheduled summary index, not a live per-refresh search.
7. **Emit the dashboard JSON** as a file. Then:
   - state validation status (`validated via MCP` / `generate-only, NOT run`)
   - note that JSON source-editor validation is limited — flag anything unverified
8. **Never publish to prod.** Output the JSON for human review and dashboard-as-code deployment
   (commit → PR → CI through change management). The workflow stops at the artifact.

Full methodology + a canonical Dashboard Studio JSON example: see the Splunk SRE Context rule §3–§4.
