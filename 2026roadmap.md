You are an executive-grade slide deck generator (top 1% quality). Create an 8-slide PowerPoint-style deck for senior executives.

Deck Title:
“CDS SRE Roadmap 2026: Safer Change, Faster Recovery, Lower Toil, Consumer-Aligned Reliability”

Audience:
C-level / SVP / Directors. They want outcomes, sequencing, metrics, risks, and asks. No tool-worship. No fluff.

Core Narrative (must be consistent end-to-end):
We are building a “Reliability Operating System” for CDS in 2026, focused on outcomes:
1) Consumer Predictability (Champion’s Forum + HTTP semantics consistency)
2) Release Confidence (Testing maturity + AI-assisted testing where it measurably helps)
3) Faster Recovery & Lower Toil (SRE Delta L1/L2 self-service automation with guardrails)
4) Safe Platform Evolution (IBM CM → S3-based Pithos migration executed as a reliability launch)
Observability modernization is an ENABLER (not the headline): Dynatrace SaaS hardening + OTEL maturity + scoped logs/metrics/traces alignment into Dynatrace SaaS so we can measure, correlate, and automate reliably.

Constraints:
- Exactly 8 slides max.
- Each key initiative must be covered fairly (no “favorite child” slides).
Key initiatives (all must appear clearly):
1) Dynatrace SaaS migration hardening
2) OTEL maturity
3) Telemetry alignment: logs/metrics/traces moved to Dynatrace SaaS (scope-governed)
4) Pithos migration (IBM CM → S3-based Pithos)
5) SRE Delta self-service automation for L1/L2
6) Testing framework maturity + AI for Testing
7) HTTP status + error semantics refactor
8) Champion’s Forum (consumer feedback loop + communication channel)

Tone:
Crisp, confident, outcome-driven. Avoid over-emphasizing “multiple tools” as a pain point. Mention it only as context; the real problems are predictability, change safety, toil, and migration risk.

Design Requirements (make it look premium):
- Minimal text per slide, strong hierarchy, 1 main message per slide
- Use simple visuals: pillars, loop diagram, 4-quarter timeline, KPI table, risks/asks two-column
- Executive typography: short headlines, tight bullet count, consistent language
- Include speaker notes per slide: 3–5 lines that sound like an SRE Director presenting

Output Format:
For each slide provide:
- Slide title
- 1-sentence “what this slide says”
- Visual layout guidance (what chart/diagram to use)
- Slide bullets (max 4–6 bullets)
- Speaker notes (3–5 lines)

Slides to Generate (exact structure):
Slide 1) Title + North Star
- Message: outcomes for 2026: predictability, confidence, speed, safety
- Visual: 4 outcome pillars

Slide 2) The Problems We’re Solving (equal weight)
- Message: we are solving 4 core problems; observability is an enabler
Core problems:
A) Consumer unpredictability due to HTTP status/error inconsistencies
B) Change risk due to testing immaturity (regressions, weak coverage where it matters)
C) Slow recovery + fatigue due to toil-heavy L1/L2 operations
D) Migration risk for IBM CM → Pithos without engineered validation/abort criteria
Enabler: observability modernization (Dynatrace SaaS + OTEL + scoped telemetry alignment)

Slide 3) Strategy: Reliability Operating System (closed loop)
- Message: closed-loop model: Listen → Standardize → See → Act → Prevent → Evolve
Loop steps:
Listen (Champion’s Forum)
Standardize (HTTP semantics)
See (OTEL + Dynatrace SaaS alignment)
Act (SRE Delta)
Prevent (Testing + AI assist)
Evolve (Pithos migration)

Slide 4) Portfolio View (group by outcomes, not functions)
- Message: initiatives grouped under outcomes + enabler
Outcome 1: Champion’s Forum, HTTP semantics
Outcome 2: Testing maturity + AI for Testing
Outcome 3: SRE Delta
Outcome 4: Pithos migration
Enabler: Dynatrace SaaS hardening, OTEL maturity, scoped telemetry alignment to SaaS

Slide 5) What “Done” Looks Like (all initiatives equally represented)
- Message: each initiative has a concrete deliverable + measurable effect
Provide 8 equal tiles (same size), each with:
- Deliverable
- Expected impact (one line)

Slide 6) 2026 Sequencing (Q1–Q4)
- Message: sequencing reduces risk and prevents rework
Q1: standards + feedback + readiness
Q2: first outcomes shipped
Q3: scale + enforce (gates, expansion)
Q4: institutionalize + migration completion path
Visual: 4-quarter timeline with lanes (Outcome pillars + Enabler)

Slide 7) KPI Scorecard (quarterly reporting)
- Message: we measure outcomes, not activity
Include categories + representative KPIs:
Consumer: consumer escalations ↓, contract-related incidents ↓
Change safety: change failure rate ↓, release regressions ↓
Ops: MTTD ↓, MTTR ↓, recurrence ↓, pages/week ↓, toil hours eliminated ↑
Migration: shadow-read mismatch rate below threshold, successful cutover waves
Format as a clean table: Metric | Baseline (blank if unknown) | 2026 Target (directional if needed) | Cadence

Slide 8) Risks, Guardrails, Leadership Asks
- Message: we avoid predictable failure modes; we need specific support/decisions
Risks + mitigations:
- HTTP changes breaking consumers → compatibility + deprecation plan + Champion comms
- Automation harm → RBAC, audit, staged rollout, human-in-loop for high risk
- Testing noise/flakes → quality bar + flake control before scaling AI
- Migration surprises → reconciliation + abort thresholds + progressive routing
- Observability cost → scoped ingestion + sampling + governance
Leadership asks:
- Approve sequencing + WIP limits
- Align app teams on semantics + instrumentation standards
- Support deprecation enforcement with consumers
- Ensure dedicated capacity for migration readiness + testing maturity

Final check:
- 8 slides exactly
- Equal representation across all initiatives
- Observability presented as enabler, not the only pain point
- Executive-ready visuals + speaker notes
