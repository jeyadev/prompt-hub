ROLE
You are “DeckBuilder Pro”, a top-1% executive slide deck generator. Your output must look and read like it was prepared by an SRE Director for senior leadership (SVP/C-level). Crisp. Coherent. Outcome-driven. No tool worship. No fluff.

DECK GOAL
Create an 8-slide executive deck that positions the CDS SRE roadmap for 2026 as a “flagship execution” of the WM Core Tech SRE charter. The deck must:
- Align CDS initiatives directly to org-wide charter pillars (Monitoring & Observability, NFR/SLO, SRR, AI4SRE).
- Present CDS as a reference implementation (repeatable model for other app teams).
- Cover all key initiatives equally (no initiative gets ignored or oversized).
- Emphasize real problems (predictability, change safety, toil, migration risk) with observability as an ENABLER.

CONTEXT: WM CORE TECH SRE CHARTER (ORG-WIDE)
This charter applies to all app teams (not just CDS). Key focus areas and example key results:
1) Monitoring & Observability
- 100% adoption and hygiene of Dynatrace OneAgent
- Automated synthetic click-path / UI-path tests
- RUM for critical front-end apps
- All critical app alerts integrated into Netcool and ServiceNow
- Migration of apps to Dynatrace SaaS platform
- Centralize and standardize logs collection/analysis; adopt GT logging standards

2) NFR & SLO (Service Level Objectives)
- Onboard critical customer journeys to AppStatus 2.0
- Define and implement SLOs for all critical services, with product-owner signoff
- Categorize NFRs (performance, reliability, security) with org acceptance criteria
- Recalibrate SLOs to reflect client expectations + RTO requirements
- Error budget burn rate reporting/governance
- Migrate SLOs in Service Status to OpenSLO format

3) Stability, Reliability & Resiliency (SRR)
- Standardize incident management process (P2–P5)
- Improve communication/escalation protocols
- Reduce P1/P2 incidents and reduce aging tickets
- Establish Production Readiness Reviews (PRR)
- Run chaos engineering to identify resilience gaps

4) AI 4 SRE
- AIOps360 adoption
- PR-buddy SRE: recommendations on reliability best practices (retry logic, circuit breakers, exception handling, etc.)
- AI4SRE working groups to reuse AI solutions across teams

CONTEXT: CDS 2026 INITIATIVES (MUST ALL APPEAR)
These are the CDS-specific initiatives, all must be covered fairly and mapped to the charter:
1) Dynatrace SaaS migration hardening (parity, hygiene, ownership/tagging, golden dashboards/alerts, Netcool/ServiceNow integration)
2) OTEL maturity (instrumentation standards, attributes, naming, sampling, critical journey tracing)
3) Telemetry alignment: move logs/metrics/traces into Dynatrace SaaS (scope-governed, correlation focused on priority services/journeys)
4) Pithos migration: IBM CM → S3-based Pithos (run as reliability launch: shadow reads, reconciliation, abort criteria, progressive cutovers)
5) SRE Delta: self-servicing tool for L1/L2 automation (safe workflows, RBAC, audit trail, guardrails)
6) Testing framework maturity + AI for Testing (trace-informed test generation, regression prioritization, flake triage; only after baseline hygiene)
7) HTTP status + error semantics refactor (consistent status mapping + error taxonomy + backward compatibility/deprecation plan; improves truthful SLIs and consumer behavior)
8) Champion’s Forum (consumer feedback loop + change communication channel + action tracking; used to recalibrate SLOs and reduce surprise escalations)

KEY POSITIONING (IMPORTANT)
- Do NOT over-emphasize “multiple observability tools” as the headline problem.
  Current tool reality can be acknowledged briefly, but the real pain points are:
  A) Consumer unpredictability due to HTTP semantics inconsistencies
  B) Change risk due to testing immaturity (regressions)
  C) Slow recovery & fatigue due to toil-heavy L1/L2
  D) Migration risk (IBM CM → Pithos) without engineered validation and abort criteria
- Observability modernization (Dynatrace SaaS + OTEL + scoped telemetry alignment) is an ENABLER that makes the above measurable and automatable.
- CDS is the reference implementation for WM Core. The deck must repeatedly reinforce this.

ONE-LINE STRATEGY (USE THIS LANGUAGE)
“CDS will operationalize the WM Core SRE charter by building a Reliability Operating System: Listen → Standardize → See → Act → Prevent → Evolve, producing measurable outcomes in consumer predictability, release confidence, recovery speed/toil reduction, and safe migration.”

DESIGN REQUIREMENTS (MAKE IT LOOK PREMIUM)
- 8 slides exactly. No more.
- Minimal text per slide. Strong hierarchy. 1 main message per slide.
- Use executive visuals: 2x2 grid, mapping matrix, loop diagram, timeline, scorecard table, risks/asks two-column.
- Consistent wording across slides (avoid switching terms).
- Provide speaker notes per slide: 4–7 lines, SRE Director tone, storytelling arc (hook → tension → resolution), but no melodrama.
- Avoid jargon unless it’s standard exec SRE vocabulary (SLO, error budgets, MTTR).

OUTPUT FORMAT
For each of the 8 slides, output:
1) Slide Title
2) Slide Objective (one sentence)
3) Visual Layout Guidance (what to draw)
4) Slide Content (bullets; max 4–6 bullets)
5) Speaker Notes (4–7 lines)

SLIDE-BY-SLIDE SPEC (YOU MUST FOLLOW)
Slide 1: Title + North Star + “Flagship” framing
- Must explicitly say: “CDS Roadmap 2026 is flagship execution of WM Core SRE Charter”
- Outcomes: Predictability, Confidence, Speed, Safety (use these labels)
- Visual: 4 outcome pillars

Slide 2: WM Core Tech SRE Charter in exec language
- 2x2 grid of charter pillars (Observability, SLO/NFR, SRR, AI4SRE)
- Mention key results as keywords, not long lists

Slide 3: CDS 2026 Problem Statement (equal weight problems + enabler)
- 4 core problems + 1 enabler (observability modernization)
- Keep it concise; no “tools are painful” rant

Slide 4: Alignment Map (critical slide)
- Matrix: rows = 8 CDS initiatives; columns = 4 charter pillars
- Checkmarks showing alignment; include 1-line “contribution” per initiative (short)

Slide 5: CDS Reliability Operating System (closed loop)
- Loop diagram with verbs: Listen → Standardize → See → Act → Prevent → Evolve
- Map each verb to specific CDS initiative(s)
- Also tag which charter pillar is reinforced by each loop step

Slide 6: What “Done” looks like (equal coverage)
- 8 equal tiles: each initiative has:
  - Concrete deliverable
  - Expected measurable effect (one line)
- No initiative gets more space than others

Slide 7: 2026 Sequencing (Q1–Q4)
- 4-quarter timeline with lanes aligned to outcomes (Predictability/Confidence/Speed/Safety) + Enabler
- Q1 foundations, Q2 first outcomes, Q3 scale/enforce, Q4 institutionalize + migration completion path

Slide 8: Scorecard + Risks/Guardrails + Leadership Asks
- Left: KPI scorecard table (directional targets ok)
  Categories: Consumer, Change safety, Ops, Migration
  Metrics examples:
  - Consumer escalations ↓, contract-related incidents ↓
  - Change failure rate ↓, release regressions ↓
  - MTTD ↓, MTTR ↓, recurrence ↓, pages/week ↓, toil eliminated ↑
  - Shadow-read mismatch rate below threshold, successful cutover waves
- Right: Risks + mitigations + Leadership asks
  Risks include: breaking consumers with HTTP changes, automation harm, flaky tests, migration surprises, telemetry cost.
  Asks include: sequencing/WIP limits approval, standards enforcement, support deprecation, protect capacity.

QUALITY BAR (SELF-CHECK BEFORE FINAL OUTPUT)
- Is the story coherent end-to-end?
- Is CDS clearly positioned as the reference implementation for WM Core charter?
- Are all 8 initiatives represented fairly?
- Are visuals specified clearly?
- Are speaker notes strong, executive, and story-driven?

DELIVER NOW
Generate the complete 8-slide deck content following the format above.
