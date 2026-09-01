---
title: Agentic Observability Foundation — Compliance QA Agent
doc_type: sre_reference
status: living
owner: SRE
audience: [sre, platform-engineering, ide-assistant]
system: compliance-qa-agent
architecture: deterministic-orchestrator + per-rule tool-calling agent + human-review escalation
telemetry_base: OpenTelemetry (GenAI semconv, Development status)
backends: [splunk-observability-cloud, dynatrace, llm-native-trace-store]
last_reviewed: 2026-08-31
review_cadence: quarterly, or on any change to the seven change surfaces (§7.1)
---

# Agentic Observability Foundation

**A reliability reference for the Compliance QA Agent.**

This document is the backbone for reasoning about the performance, correctness and cost
of an agentic system that generates a compliance checklist and validates documents against it.
It is written to serve two readers at once:

- **A human engineer** building the mental model for how these systems fail and how you see them failing.
- **An IDE assistant** with this repository open, which will treat this file as authority when suggesting
  instrumentation, error handling, SLO definitions and review comments.

That dual audience imposes a discipline: every claim must be either *derivable*, *measurable*, or
*explicitly marked as a judgement call*. A reference document that an assistant reads as ground truth
becomes wrong code the moment it goes stale. §11 is the maintenance contract that keeps it honest.

---

## Table of contents

| § | Part | What it gives you |
|---|------|-------------------|
| 0 | [How to use this document](#0-how-to-use-this-document) | Reading paths, conventions, provenance tiers |
| 1 | [The mental model](#1-the-mental-model) | Three planes, unit hierarchy, the golden-signal extension, error asymmetry |
| 2 | [The telemetry contract](#2-the-telemetry-contract) | Span tree, attribute registry strategy, content policy, cardinality, sampling |
| 3 | [The metric catalogue](#3-the-metric-catalogue) | Every SLI, by plane, with definition and the failure it detects |
| 4 | [SLOs and error budgets](#4-slos-and-error-budgets) | Two budgets, the statistics of quality SLOs, burn-rate alerting |
| 5 | [The failure mode catalogue](#5-the-failure-mode-catalogue) | 40+ modes with symptom → signal → discriminator → mitigation |
| 6 | [Diagnostic playbooks](#6-diagnostic-playbooks) | Triage tree and the eight investigations you will actually run |
| 7 | [Change safety](#7-change-safety) | Seven change surfaces, eval gates, progressive delivery, rollback |
| 8 | [Evidence, audit and reproducibility](#8-evidence-audit-and-reproducibility) | Trace-as-evidence, retention, replay contract |
| 9 | [Instrumentation reference](#9-instrumentation-reference) | Code patterns, collector pipeline, anti-patterns |
| 10 | [Rules for the IDE assistant](#10-rules-for-the-ide-assistant) | Normative MUST/SHOULD/NEVER, and a diff review checklist |
| 11 | [Maintenance contract](#11-maintenance-contract) | How this document stays true |
| A–F | [Appendices](#appendix-a--attribute-registry) | Registries, taxonomy mappings, maturity model, sources |

---

# 0. How to use this document

## 0.1 Reading paths

**First read (60 minutes), in order:** §1 → §2.1 → §3 (skim the catalogue, read the "why it matters" column)
→ §4.1–4.3 → §5.0 (the taxonomy map) → §10.

**When you are on call:** §6 first. It is written to be entered from a symptom, not from the top.

**When you are instrumenting:** §2 then §9. §2 is the contract; §9 is the implementation of the contract.

**When you are reviewing a change:** §7 and §10.2.

**When an auditor or a regulator asks:** §8.

## 0.2 Conventions

- **MUST / SHOULD / MAY / NEVER** are used in the RFC 2119 sense. They are normative for this repository.
- `code font` marks an attribute, metric, span or configuration name that appears literally in telemetry.
- Attribute names in the `gen_ai.*` namespace come from the OpenTelemetry GenAI semantic conventions.
  Attribute names in the `qa.*` namespace are **ours** and are stable by our own decree (see §2.2 for why
  that separation is load-bearing, not cosmetic).
- Where a number is given as a target or threshold, it is a **starting point calibrated from published
  practice, not a law**. Re-derive it from your own baseline within the first 30 days of operation.

## 0.3 Provenance tiers

Numbers in this document carry an implicit tier. When you are about to make a decision that depends on one, check the tier.

| Tier | Meaning | Examples in this doc |
|------|---------|----------------------|
| **P1 — Primary/derivable** | Specification text, peer-reviewed result, or arithmetic derived in-document | OTel semconv attribute names; MAST failure distribution; the fan-out tail derivation in §1.3; the sample-size arithmetic in §4.4 |
| **P2 — Vendor documentation** | Product documentation from the backend vendors | Splunk AI Agent Monitoring env vars; Dynatrace OTLP ingest |
| **P3 — Practitioner report** | Blog/practitioner writing, single-source, often unverifiable | The "1.2 second review dwell time" anecdote; specific cache-hit-rate targets; latency-cliff ratios |
| **P4 — Our judgement** | A recommendation made here, defensible but not sourced | Most SLO targets; the three-plane model; the sampling tiers |

**Treat every P3 number as a hypothesis to test against your own telemetry, not as a benchmark to hit.**
A meaningful share of freely available "2026 AI observability" writing is search-optimised and
unverifiable. It is useful for *shapes of arguments*, not for *values of constants*.

---

# 1. The mental model

## 1.1 The inversion

Classical SRE was built for systems where **correctness is assumed and availability is scarce**.
The service either computes the right answer or it fails loudly. So we measured whether it responded,
how fast, and how often it errored — and correctness rode along for free, guaranteed by the code.

Agentic systems invert this. **Availability is cheap and correctness is scarce.**
The model will almost always return something. It will return it quickly. It will return it with an
HTTP 200 and a confident tone. And it may be wrong in a way no infrastructure signal can see.

> A compliance QA agent that marks a document *compliant* against a clause it never actually read
> emits exactly the same telemetry as one that read the clause and got it right — unless you
> deliberately instrument the difference.

This is the **green dashboard paradox**: every infrastructure signal healthy, every business outcome
wrong. It is the single organising problem of this document. Everything downstream — the span schema,
the second error budget, the evaluation plane, the human-review metrics — exists to close the gap
between "the system responded" and "the system was right."

**The consequence for you as an SRE:** your job expands from *keeping the service up* to
*keeping the service correct, and being able to prove it was correct*. The second half is new. In a
compliance context it is also the half with regulatory teeth (§8).

## 1.2 Three planes

Do not model this system as one thing. Model it as three planes with genuinely different physics.
Mixing them is the most common analytical error in agentic incident response — you reach for a
model-quality explanation when the actual fault is a queue, or you tune retries when the actual fault
is a prompt.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  CONTROL PLANE — deterministic                                               │
│  Orchestrator: intake, checklist expansion, fan-out, scheduling, retry,      │
│  aggregation, persistence, escalation routing                                │
│                                                                              │
│  Physics: classic distributed systems. Queues, concurrency limits, backoff,  │
│  idempotency, partial failure, straggler tails. Reproducible.                │
│  Failure signature: latency, saturation, error codes. Legible to classic SRE.│
├──────────────────────────────────────────────────────────────────────────────┤
│  COGNITION PLANE — stochastic                                                │
│  One tool-calling agent invocation per QA rule: reason → call tool →         │
│  observe → reason → … → emit verdict                                         │
│                                                                              │
│  Physics: probabilistic. Same input, different trajectory. Heavy-tailed      │
│  latency because output length is a random variable. Silent semantic         │
│  failure. Non-reproducible without a replay contract.                        │
│  Failure signature: trajectory shape, verdict quality, token economics.      │
│  Invisible to classic SRE. Requires an evaluation plane to observe at all.   │
├──────────────────────────────────────────────────────────────────────────────┤
│  CONSEQUENCE PLANE — human + institutional                                   │
│  Verdicts, evidence, escalation queue, reviewer decisions, overrides,        │
│  the audit record, the downstream compliance decision                        │
│                                                                              │
│  Physics: human throughput, attention, automation bias, organisational       │
│  incentive. Slow feedback. This is where errors become consequences.         │
│  Failure signature: escalation rate drift, override collapse, review dwell   │
│  time, queue age. Almost never instrumented. Frequently the real incident.   │
└──────────────────────────────────────────────────────────────────────────────┘
```

**The three questions that follow from the model** — ask them in this order in every investigation:

1. **Did the work happen?** (control plane) — every rule dispatched, every agent invoked, every verdict persisted.
2. **Was the work good?** (cognition plane) — did the agent reason over the right evidence and reach a defensible verdict.
3. **Did the outcome land correctly?** (consequence plane) — did the right things escalate, did humans actually review them, did the record survive.

A system can pass 1 and fail 2 silently for weeks. It can pass 1 and 2 and fail 3 — that is the
rubber-stamp failure (§5.7), and it is the one that ends up in a regulatory finding.

## 1.3 The unit-of-work hierarchy (and why your denominators lie)

Nothing in an agentic system means anything until you name the unit. "95% success rate" is not a
statement until you say success *of what*.

```
Run                    one submission: a document set + a rule set + a requester
 └── Document          one compliance document under validation
      └── Rule         one checklist item / QA rule evaluated against that document
           └── Agent invocation   one tool-calling agent instance, scoped to one rule
                └── Step (turn)   one model call + the tool calls it triggers
                     ├── Model call     one inference request to a provider
                     ├── Tool call      one retrieval / extraction / lookup
                     └── Guardrail call one input or output check
           └── Verdict            pass | fail | insufficient-evidence | escalate
      └── Document result   aggregation of rule verdicts
 └── Run result        aggregation of document results + escalation set
```

**Every SLI in §3 declares its unit. If you cannot name the denominator, the metric is decoration.**

### 1.3.1 The fan-out tail multiplication — derive this once, remember it forever

Your architecture evaluates R rules per document, largely in parallel, and the document is not done
until the slowest rule is done. Document latency is therefore a **maximum over R draws**, not a mean.

If a single rule's latency exceeds `t` with probability `q`, and rule latencies are approximately
independent, then:

```
P(document latency > t) = 1 − (1 − q)^R
```

Set `t` = the per-rule p99 (so `q = 0.01`):

| R (rules per document) | P(document exceeds the *per-rule p99*) | In plain terms |
|---|---|---|
| 10 | 9.6% | rule p99 ≈ document p90 |
| 50 | **39.5%** | rule p99 ≈ document **p60** |
| 100 | 63.4% | rule p99 ≈ document p37 |
| 200 | 86.6% | rule p99 is worse than the document median |

Inverted — the per-rule tail you must hold to keep a *document* p99 at `t`:

```
q_required = 1 − 0.99^(1/R)
```

| R | Required per-rule percentile |
|---|---|
| 10 | p99.90 |
| 50 | **p99.98** |
| 100 | p99.99 |

**Three consequences that should change your engineering:**

- **Per-rule p99 is not a useful SLI on its own.** Document-level and run-level latency SLIs are the
  ones customers feel. Track the per-rule tail as a *driver*, not as an objective.
- **Reducing per-rule latency *variance* beats reducing per-rule *mean*.** At R=50 you are living in
  the far tail of the rule distribution, and the far tail is governed by variance.
- **Hedging and per-rule deadlines have outsized leverage.** Capping a rule at a deadline and returning
  `insufficient-evidence → escalate` converts an unbounded tail into a bounded one, and moves the
  residual into the consequence plane where a human can absorb it. That is a *good* trade — provided
  you instrument the escalation so the trade stays visible (§3.5).

This is the classic tail-at-scale problem, and it is sharper here than in microservices because
LLM latency is genuinely heavy-tailed: output length is itself a random variable and latency scales
with it, so the distribution has a long right tail rather than a bounded one. Practitioner reports of
p50 ≈ 0.8 s against p99 ≈ 28 s on the same endpoint (P3) are extreme but directionally real —
**never reason about this system using mean latency.**

### 1.3.2 Retry amplification — the second-order load term

Let `f` be the per-call transient failure rate and `k` the maximum attempts. Expected calls per logical
call is a geometric sum:

```
E[calls] = (1 − f^k) / (1 − f)
```

| f (failure rate) | k=3 | Load multiplier |
|---|---|---|
| 0.01 | 1.010 | +1% — invisible |
| 0.05 | 1.052 | +5% — invisible |
| 0.30 | 1.39 | +39% |
| 0.60 | **1.96** | **+96% — you have doubled load on an already-degraded provider** |

**Retries are a load amplifier that switches on exactly when the system can least afford it.** This is
the retry storm, and in an agentic system it is worse than in a microservice because a retry re-bills
the full token cost. Failures are also *correlated*, not independent: a provider 429 or 529 hits your
whole fleet at once, which violates the assumption every stock circuit breaker is built on.
See §5.2 and §9.4 for the failure-class routing that fixes this.

## 1.4 Golden signals, extended — not replaced

There is a fashionable claim that the four golden signals are dead for AI systems. That is wrong, and
adopting it will cost you. Latency, traffic, errors and saturation remain necessary — they are the only
signals that see the control plane, which is where a large fraction of your real incidents will live.

They are, however, **radically insufficient**. The correct move is extension, not replacement.

| Classical signal | Still true because | Agentic extension | The failure it catches that the classical signal cannot |
|---|---|---|---|
| **Latency** | The orchestrator, queues and tools are ordinary software | **Decomposed latency**: TTFT, inter-token latency, tool latency, queue wait, *and* step count | A run that is slow because the agent looped 14 times, not because anything was slow |
| **Traffic** | Request volume still drives capacity | **Token velocity** and **step velocity** | Flat request rate with 3× token burn after a prompt change |
| **Errors** | Transport and tool failures are real | **Semantic error rate**: wrong verdict, ungrounded citation, guardrail intervention | A 100%-success run that produced a wrong compliance verdict |
| **Saturation** | Queues, workers, connections saturate normally | **Context saturation** (% of window used), **provider quota headroom**, **reviewer queue depth** | Quality degrading because prompts crossed a context threshold |

And then four signals with no classical ancestor at all:

| Agentic signal | Question it answers | Primary metrics (§3) |
|---|---|---|
| **Correctness** | Was the verdict right? | verdict accuracy, false-pass rate, false-fail rate, groundedness |
| **Convergence** | Did the agent finish the way it should have? | steps per verdict, loop rate, termination reason distribution, deadline-hit rate |
| **Containment** | Did the safety and escalation machinery hold? | guardrail intervention rate, escalation rate, insufficient-evidence rate |
| **Cost** | What did the answer cost, and is that stable? | cost per verdict, cost per document, cache hit rate, retry-attributed spend |

Mnemonic if you want one: **LTES + C⁴** (Correctness, Convergence, Containment, Cost).

## 1.5 The determinism gradient

Instrument each component according to where it sits on this gradient. Applying stochastic tooling to
deterministic code wastes money; applying deterministic tooling to stochastic code produces false confidence.

```
fully deterministic ──────────────────────────────────────────► fully stochastic

orchestrator    tool I/O      retrieval      guardrail       agent reasoning
scheduling      schema        ranking        classifier      verdict + rationale
persistence     validation

├─ classic SRE ─┤├── classic + distributional ──┤├──── evaluation plane required ────┤
   exact SLIs        percentile SLIs, drift        sampled SLIs with confidence
   binary alerts     detection on distributions    intervals, judge calibration
```

**Rule of thumb:** if you can write an assertion that is true for every correct execution, it belongs
on the left and you should assert it in code, not evaluate it with a model. Deterministic checks are
free, instant, and never drift. The most common instrumentation mistake in agentic systems is paying a
judge model to check something a five-line assertion could have proved — e.g. "did the agent cite a
span of text that actually exists in the document?" is a substring check, not an LLM judgement.

## 1.6 Error asymmetry — the most important framing for *this* system

A compliance QA agent has two failure directions, and they are not remotely equal.

|  | Agent says **PASS** | Agent says **FAIL** |
|---|---|---|
| **Truly compliant** | ✅ True pass | ⚠️ **False fail** — wasted review effort, friction, credibility loss, delivery delay |
| **Truly non-compliant** | 🔴 **FALSE PASS** — a non-compliant document is certified. Regulatory exposure. Discovered externally, months later, at maximum cost | ✅ True fail |

**A false pass is a silent, delayed, high-severity defect. A false fail is a loud, immediate, low-severity annoyance.**

This asymmetry must be encoded everywhere:

- **In the SLOs.** False-pass rate gets a far tighter objective than false-fail rate (§4.3). One SLI over
  "accuracy" hides the asymmetry and is actively harmful.
- **In the alerting.** False-pass rate drift pages. False-fail rate drift files a ticket.
- **In the escalation policy.** Uncertainty must resolve toward escalation, never toward pass.
  `insufficient-evidence` MUST be a first-class verdict — an agent that cannot find the evidence must
  be able to say so rather than guessing. An agent with only {pass, fail} available will manufacture
  confidence, and it will manufacture it in whichever direction the prompt biases it.
- **In the sampling.** Your human-review sample MUST over-sample the PASS verdicts, because that is where
  the expensive error hides. Reviewing the escalations tells you nothing about the failure that matters.
  This is counter-intuitive and is the single highest-leverage recommendation in this document.
- **In the metric names.** Never emit a single `qa.verdict.accuracy` without also emitting the two
  directional rates. Aggregate accuracy is a number that can improve while your actual risk gets worse.

> **The asymmetry trap:** a model change that raises overall accuracy from 94% → 96% while shifting the
> error mix from mostly-false-fail to mostly-false-pass is a **regression**, and a single accuracy SLI
> will show it as an improvement and ship it.

## 1.7 The verification gap

Two independent bodies of evidence point at the same weak spot, from opposite ends of the pipeline.

**From the research side:** the MAST taxonomy (Berkeley; 150 hand-annotated traces, inter-annotator
κ = 0.88, extended to 1,600+ traces across 7 frameworks) finds that ~24% of multi-agent failures sit in
**task verification** — the agent terminating early, not verifying, or verifying incorrectly. Another
~13% is *reasoning-action mismatch*: the agent's stated reasoning does not match what it actually did.
Those are, precisely, failures of *checking* rather than failures of *capability*.

**From the human side:** the human review layer that is supposed to catch what the agent missed is
subject to automation bias. Reviewers align more with an AI recommendation than they would with the
same evidence unaided, and — counter-intuitively — *better explanations make this worse*, because a
plausible rationale substitutes for independent evaluation rather than enabling it (P3).

**The gap:** the agent under-verifies, and the human under-verifies the agent. Two weak checks in
series do not make a strong one. Every design and every metric in this document that touches
verification (§3.4, §3.5, §5.5, §5.7) exists to make that composite check observable.

## 1.8 The trace is the primary artifact

In classical systems, logs are the record and traces are a debugging luxury. Invert that here.

- A **log line** tells you a thing happened.
- A **trace** tells you the *shape of the reasoning*: which evidence was retrieved, in what order, how
  many times the agent went round the loop, what it decided and on what basis.
- For an agentic compliance system, **that shape is the evidence**, and the evidence is what a regulator
  or an internal auditor will ask for (§8).

Three consequences:

1. **Trace completeness is a reliability property**, not an observability nicety. A dropped span in the
   compliance path is a gap in an audit record. Head-based sampling of the verdict path is therefore
   prohibited (§2.6).
2. **Trace size is an engineering constraint.** Agent traces are enormous — the TRAIL benchmark reports
   mean trace inputs of roughly 263K–728K tokens with maxima in the millions. Your storage, your UI and
   any model you point at a trace will all hit limits. Design for it: structured summaries alongside
   raw detail, and externalised content (§2.5).
3. **Do not assume a model can debug the trace for you.** On TRAIL, the best evaluated model achieved
   ~11% joint accuracy at locating *and* categorising errors in agent traces. Automated trace triage is
   a promising assistant and a terrible oracle. Build the deterministic signals in §3 so that you are
   not dependent on one.

---

# 2. The telemetry contract

This section is a **contract**, not a suggestion. Code that emits telemetry outside it is a defect,
because every dashboard, alert, SLO and audit query in §3–§8 is written against these names.

## 2.1 The span tree

This is the canonical shape for one document validated against R rules. Learn it; it is the map you
navigate during an incident.

```
qa.run                                            SERVER   ← root; one submission
│  qa.run.id, qa.tenant.id, qa.rule_set.version, qa.change.manifest
│
├── qa.intake                                     INTERNAL ← ingest, parse, classify
│   ├── execute_tool parse_document               INTERNAL
│   └── execute_tool classify_document_type       INTERNAL
│
├── qa.checklist.build                            INTERNAL ← checklist expansion
│   ├── invoke_agent checklist_planner            INTERNAL (only if generated, not looked up)
│   │   └── chat <model>                          CLIENT
│   └── qa.checklist.validate                     INTERNAL ← deterministic schema + coverage check
│
├── qa.document                                   INTERNAL ← one per document
│   │  qa.document.id, qa.document.version, qa.document.page_count
│   │
│   ├── qa.rule                                   INTERNAL ← ONE PER RULE. The critical unit.
│   │   │  qa.rule.id, qa.rule.version, qa.rule.severity, qa.rule.category
│   │   │
│   │   ├── qa.guardrail.input                    INTERNAL ← pre-flight checks
│   │   │
│   │   ├── invoke_agent rule_validator           INTERNAL ← the agent loop
│   │   │   │  gen_ai.agent.name, gen_ai.conversation.id, qa.agent.step_budget
│   │   │   │
│   │   │   ├── chat <model>                      CLIENT   ← step 1 reasoning
│   │   │   │      gen_ai.usage.input_tokens, gen_ai.usage.output_tokens,
│   │   │   │      gen_ai.usage.cache_read.input_tokens, gen_ai.response.finish_reasons
│   │   │   │      event: gen_ai.client.inference.operation.details
│   │   │   │
│   │   │   ├── execute_tool retrieve_clause      INTERNAL ← qa.retrieval.* attributes
│   │   │   ├── execute_tool extract_table        INTERNAL
│   │   │   ├── chat <model>                      CLIENT   ← step 2 reasoning
│   │   │   └── ...                                        ← repeats until verdict or budget
│   │   │
│   │   ├── qa.guardrail.output                   INTERNAL ← grounding + policy checks
│   │   ├── qa.verdict                            INTERNAL ← verdict materialised
│   │   │      qa.verdict.value, qa.verdict.confidence, qa.verdict.evidence_count,
│   │   │      qa.verdict.termination_reason
│   │   │      event: gen_ai.evaluation.result (× N evaluators)
│   │   │
│   │   └── qa.escalation.decide                  INTERNAL ← routing decision
│   │
│   ├── qa.rule ...                               (× R, mostly concurrent)
│   └── qa.document.aggregate                     INTERNAL
│
├── qa.run.aggregate                              INTERNAL
└── qa.persist                                    INTERNAL ← audit record written

── asynchronous, linked by span link, NOT a child (different trace, different lifetime) ──
qa.review                                         SERVER   ← human review, minutes-to-days later
   qa.review.reviewer_role, qa.review.dwell_ms, qa.review.decision,
   qa.review.override, qa.review.override_reason_code
```

**Three structural decisions worth understanding:**

1. **`qa.rule` is a span, not an attribute.** It is the unit at which quality is measured, cost is
   attributed, and failure is localised. Making it a span is what lets you ask "which rules are
   expensive / slow / wrong" without a separate analytics pipeline.
2. **Human review is a linked trace, not a child span.** It happens minutes to days later. Keeping it in
   the same trace would force you to hold traces open for days and would break every latency metric.
   Use an OTel **span link** plus the shared `qa.verdict.id` correlation key.
3. **Guardrails and verdict materialisation are explicit spans.** If a guardrail is an `if` statement
   buried in a function, its intervention rate is unobservable — and intervention rate is one of your
   four new golden signals.

## 2.2 Semantic convention strategy: two namespaces, on purpose

**Position: emit OpenTelemetry GenAI conventions for the model layer, and own a stable `qa.*` namespace for the domain layer. Never let a domain SLI depend on an unstable attribute.**

The reasoning is not stylistic. As of this writing the GenAI conventions are still **Development
status** — every `gen_ai.*` span, attribute, metric and event carries that badge. They moved out of the
core `semantic-conventions` repo into a dedicated `open-telemetry/semantic-conventions-genai` repo,
which as of mid-2026 has **no tagged releases and no pinnable schema URL**. And they have already broken
things at least three times:

| Old (2024–25) | Current | Blast radius if you built on the old name |
|---|---|---|
| `gen_ai.system` | `gen_ai.provider.name` | Every provider-dimensioned dashboard and alert |
| `gen_ai.usage.prompt_tokens` | `gen_ai.usage.input_tokens` | Every cost metric |
| `gen_ai.prompt` / `gen_ai.completion` | **removed** → opt-in `gen_ai.input.messages` / `gen_ai.output.messages` | Every content-capture pipeline and redaction rule |

Meanwhile framework emitters lag independently — some default to older convention versions, some emit
both generations simultaneously during transitions.

**The operational rules that follow:**

- **PIN** exact versions of framework, instrumentation library, SDK and exporter. Treat an
  instrumentation-library bump as a telemetry-schema migration with its own change ticket, not as a
  routine dependency update.
- **COALESCE** on read. Every query that touches a token count or a provider name reads both generations:
  `coalesce(gen_ai.usage.input_tokens, gen_ai.usage.prompt_tokens)`. Put this in a shared query library or
  a collector transform, never scattered across dashboards.
- **NORMALISE at the collector**, not in the application. A transform processor that rewrites legacy
  attribute names into the current generation gives you exactly one place to fix a convention change.
- **NEVER build an SLO on a `gen_ai.*` attribute alone.** SLOs read `qa.*`. `gen_ai.*` is for
  cross-vendor tooling, provider comparison and vendor-supplied dashboards; `qa.*` is for your contract.
- **TEST the exported spans**, not the documentation. A test that asserts on the actual OTLP payload for
  a representative agent run is the only thing that catches a silent convention drift on a dependency
  bump. This test is cheap and it will pay for itself. (§9.5)

**On OpenInference:** if you adopt an LLM-native trace store, you will meet OpenInference — a parallel
convention with its own span kinds (`LLM`, `EMBEDDING`, `CHAIN`, `RETRIEVER`, `RERANKER`, `TOOL`,
`AGENT`, `GUARDRAIL`, `EVALUATOR`, `PROMPT`) selected via `openinference.span.kind`, and namespaces
`llm.*`, `retrieval.*`, `tool.*`, `document.*`. Its `GUARDRAIL` and `EVALUATOR` span kinds are genuinely
more expressive than anything currently in the OTel GenAI set. Both can ride the same OTLP stream —
attributes are additive. **Dual-emit if your tooling needs it, but the `qa.*` namespace remains the
single source of truth for SLOs.** Do not let the choice of trace store leak into your objectives.

## 2.3 Attribute registry — required by span type

Full registry in [Appendix A](#appendix-a--attribute-registry). This is the required set.

### Resource attributes (on every span)

| Attribute | Type | Example | Why |
|---|---|---|---|
| `service.name` | string | `compliance-qa-orchestrator` | Standard |
| `service.version` | string | `2026.8.3+a1b2c3d` | Correlate regressions to deploys |
| `deployment.environment.name` | string | `prod` | Standard |
| `qa.pipeline.version` | string | `7.2.0` | The composite version of the whole QA pipeline |

### Correlation keys (propagated via baggage through the whole request path)

| Attribute | Cardinality | Notes |
|---|---|---|
| `qa.run.id` | unbounded | ULID. The join key for everything. |
| `qa.document.id` | unbounded | Stable across runs — lets you compare verdicts on the same document over time |
| `qa.document.version` | low | Content hash or version tag. **Required** — a verdict is only meaningful against a document version |
| `qa.rule.id` | bounded (see §2.4) | Stable rule identifier |
| `qa.verdict.id` | unbounded | Joins the async review trace back to the verdict |
| `qa.tenant.id` | bounded | Business unit / entity |

> **`qa.document.version` is the most commonly omitted required attribute and the most expensive to omit.**
> Without it, "the agent changed its mind about this document" is indistinguishable from "the document changed."

### The change manifest (on the `qa.run` root span — non-negotiable)

Seven things can change the behaviour of this system independently of a code deploy. All seven MUST be
recorded on every run, or you cannot attribute a regression. See §7.1.

| Attribute | Example |
|---|---|
| `qa.change.prompt_version` | `rule_validator@v14` |
| `qa.change.model_id` | `<provider>:<model>:<snapshot-or-date>` — the *pinned snapshot*, never a floating alias |
| `qa.change.tool_schema_hash` | `sha256:7f3a…` over the serialised tool definitions |
| `qa.change.rule_set_version` | `RS-2026.08` |
| `qa.change.retrieval_index_version` | `idx-2026-08-27T03:00Z` |
| `qa.change.judge_version` | `judge-groundedness@v3` |
| `qa.change.threshold_profile` | `thresholds@v5` (confidence cutoffs, escalation triggers) |

### `qa.rule` span

| Attribute | Req | Notes |
|---|---|---|
| `qa.rule.id` | MUST | |
| `qa.rule.version` | MUST | |
| `qa.rule.category` | MUST | Low cardinality. Your primary quality slice. |
| `qa.rule.severity` | MUST | `critical` / `major` / `minor` — drives escalation policy and error-budget weighting |
| `qa.rule.evidence_required` | SHOULD | Boolean. Whether a citation is mandatory for a verdict |
| `error.type` | COND | On failure |

### `invoke_agent` span

| Attribute | Req | Notes |
|---|---|---|
| `gen_ai.operation.name` | MUST | `invoke_agent` |
| `gen_ai.agent.name` | MUST | `rule_validator` |
| `gen_ai.conversation.id` | SHOULD | Ties the multi-step conversation together |
| `qa.agent.step_budget` | MUST | The configured cap |
| `qa.agent.steps_used` | MUST | Actual. `steps_used / step_budget` is your convergence signal |
| `qa.agent.termination_reason` | MUST | Enum, see §2.3.1 — **the single highest-value attribute in the schema** |
| `qa.agent.context_tokens_peak` | SHOULD | Peak context occupancy across steps |
| `qa.agent.context_utilisation_peak` | SHOULD | Peak as a fraction of the window |

#### 2.3.1 `qa.agent.termination_reason` — the enum that earns its keep

Every agent invocation ends for exactly one reason. Making that reason a low-cardinality enum turns the
most important behavioural question — *why did the agent stop?* — into a one-line query.

| Value | Meaning | Health |
|---|---|---|
| `verdict_reached` | Agent produced a verdict with evidence | ✅ the only good outcome |
| `insufficient_evidence` | Agent explicitly could not find evidence | ✅ healthy honesty — track the rate, it should be non-zero |
| `step_budget_exhausted` | Hit the step cap | 🔴 convergence failure — see §5.3 |
| `deadline_exceeded` | Wall-clock deadline hit | 🟠 latency failure; verdict is missing |
| `context_overflow` | Context window exceeded | 🔴 context engineering failure — see §5.2 |
| `guardrail_blocked` | Output guardrail refused the verdict | 🟠 containment worked; investigate the rate |
| `tool_unavailable` | Required tool failed terminally | 🟠 control-plane failure surfacing in cognition plane |
| `provider_error` | Terminal model provider error | 🟠 |
| `cancelled` | Upstream cancellation | ⚪ |

**A healthy steady state is dominated by `verdict_reached`, with a small stable band of
`insufficient_evidence`. Any movement in the distribution is a change signal, even when every other
metric is flat.** Alert on the *distribution*, not just the error members — a rise in
`insufficient_evidence` from 4% to 11% with no code change is one of the earliest available warnings
of retrieval or model degradation.

### `chat` / inference span (OTel GenAI)

| Attribute | Req | Notes |
|---|---|---|
| `gen_ai.operation.name` | MUST | `chat` |
| `gen_ai.provider.name` | MUST | Renamed from `gen_ai.system` |
| `gen_ai.request.model` | MUST | Pinned snapshot |
| `gen_ai.response.model` | MUST | **What actually served.** Compare against request — see §5.2 silent version change |
| `gen_ai.usage.input_tokens` | MUST | Renamed from `prompt_tokens` |
| `gen_ai.usage.output_tokens` | MUST | |
| `gen_ai.usage.cache_read.input_tokens` | SHOULD | Where the provider exposes it; drives §3.6 cache economics |
| `gen_ai.usage.cache_creation.input_tokens` | SHOULD | |
| `gen_ai.usage.reasoning.output_tokens` | SHOULD | Reasoning models: this is often the majority of output spend and is invisible otherwise |
| `gen_ai.response.finish_reasons` | MUST | Array. `["length"]` is a truncation bug in disguise |
| `gen_ai.response.id` | SHOULD | Provider-side correlation for support escalations |
| `gen_ai.request.temperature`, `top_p`, `seed` | SHOULD | Reproducibility inputs |
| `error.type` | COND | |

### `execute_tool` span

| Attribute | Req | Notes |
|---|---|---|
| `gen_ai.operation.name` | MUST | `execute_tool` |
| `gen_ai.tool.name` | MUST | In the span name too |
| `gen_ai.tool.type` | SHOULD | `function` / `extension` / `datastore` |
| `qa.tool.idempotency_key` | MUST for mutating tools | Derived from `(run_id, rule_id, step_index)` |
| `qa.tool.arg_validation` | MUST | `valid` / `invalid_schema` / `invalid_semantics` — separates "model called the tool wrong" from "tool broke" |
| `qa.tool.result_cardinality` | SHOULD | Rows/chunks returned. Zero is a strong signal |

### Retrieval spans (`execute_tool` with retrieval semantics)

| Attribute | Req | Notes |
|---|---|---|
| `qa.retrieval.query_type` | SHOULD | `semantic` / `lexical` / `hybrid` / `exact` |
| `qa.retrieval.k` | SHOULD | Requested |
| `qa.retrieval.returned` | MUST | Actual. `returned == 0` is the leading indicator of `insufficient_evidence` |
| `qa.retrieval.top_score` | SHOULD | Distribution shift here precedes quality shift |
| `qa.retrieval.index_version` | MUST | Matches the change manifest |

### `qa.verdict` span

| Attribute | Req | Notes |
|---|---|---|
| `qa.verdict.id` | MUST | |
| `qa.verdict.value` | MUST | `pass` / `fail` / `insufficient_evidence` / `escalate` |
| `qa.verdict.confidence` | SHOULD | Only if calibrated — see the warning in §3.4 |
| `qa.verdict.evidence_count` | MUST | Number of citations attached |
| `qa.verdict.evidence_verified` | MUST | Count that passed the **deterministic** span-exists check |
| `qa.verdict.grounded` | MUST | Boolean: `evidence_verified >= 1` when evidence is required |
| `qa.verdict.escalated` | MUST | Boolean |
| `qa.verdict.escalation_reason` | COND | Enum: `low_confidence` / `high_severity_rule` / `guardrail` / `sampling` / `disagreement` / `deadline` |

> `qa.verdict.evidence_verified` is a **deterministic** check — does the cited text actually appear in
> the cited document version at the cited location. It costs microseconds, it never drifts, and it
> catches the highest-consequence hallucination class in this system. Build it before you build any
> LLM judge. This is §1.5 applied.

## 2.4 Cardinality budget

Cardinality is where observability bills and query performance die, and agentic telemetry is unusually
dangerous because so many natural dimensions are unbounded.

| Dimension | Bound | Metric dimension? | Span attribute? |
|---|---|---|---|
| `qa.rule.category` | ~10–50 | ✅ yes | ✅ |
| `qa.rule.severity` | 3 | ✅ yes | ✅ |
| `qa.verdict.value` | 4 | ✅ yes | ✅ |
| `qa.agent.termination_reason` | 9 | ✅ yes | ✅ |
| `gen_ai.provider.name` | <10 | ✅ yes | ✅ |
| `gen_ai.request.model` | <20 | ✅ yes | ✅ |
| `qa.tenant.id` | 10s–100s | ⚠️ only if genuinely bounded and needed | ✅ |
| `qa.rule.id` | **100s–1000s** | ⚠️ **exemplars + traces only** | ✅ |
| `qa.run.id`, `qa.document.id`, `qa.verdict.id` | unbounded | 🔴 **never** | ✅ |

**Rules:**

- **Metrics carry bounded dimensions. Traces carry identity.** Join them with **exemplars** — attach
  trace IDs to metric datapoints so a spike on a chart is one click from the trace that produced it.
  This is the single most useful integration to configure and it is routinely skipped.
- **`qa.rule.id` MUST NOT be a metric dimension by default.** With a few hundred rules × severity ×
  verdict × model you cross into six-figure series counts. Get per-rule quality from the trace store
  or a periodic batch rollup instead.
- If you need per-rule metrics for a *specific* investigation, add a **temporary, explicitly scoped**
  metric with a removal date in the code comment. Undocumented "temporary" high-cardinality metrics
  are how observability bills triple.
- **Estimate before you ship.** Series count ≈ ∏(dimension cardinalities) × number of metrics.
  Do this arithmetic in the PR description for any new dimensioned metric. §10.1 makes it a rule.

## 2.5 Content capture policy

Prompts, document text and verdict rationales are simultaneously **the most valuable debugging data
you have** and **the highest-risk data you hold**. In a compliance system the documents are, by
definition, sensitive.

Three modes, per OTel's model, and you will use all three in different places:

| Mode | What | Where to use it |
|---|---|---|
| **Disabled** | No content in telemetry | Default for all high-volume, low-value spans |
| **Span attributes** | Content inline on the span | ⚠️ Only in dev/staging. Exceeding backend attribute limits causes silent truncation or span rejection — Splunk explicitly warns about this |
| **Externalised** | Content in a governed store; span carries a **reference** | ✅ **Production default** |

**The production pattern: two stores, one reference.**

```
Application ──► OTLP: spans + metrics + events (NO raw content)
                 │      each content-bearing span carries qa.content.ref = <opaque URI>
                 │
                 ├──► Observability backends (Splunk / Dynatrace) — 13-month metrics,
                 │    30-day traces, no sensitive content, broad engineering access
                 │
                 └──► Evidence store — content addressed by qa.content.ref,
                      encrypted, access-controlled, retention set by the audit
                      requirement (§8), narrow access, full access logging
```

**Non-negotiables:**

- Redaction happens in the **collector**, not the application. One choke point, auditable, testable,
  fixable without a redeploy of every service.
- Redaction is **allow-list**, not deny-list. A deny-list fails open, and failing open here means
  document content in a general-access observability backend.
- `gen_ai.input.messages` / `gen_ai.output.messages` are opt-in in the spec. **Keep them off in
  production.** If you need message content for an incident, turn it on for a scoped, time-boxed,
  ticketed window and turn it off again. Make the flag a runtime config, not a deploy.
- Test the redaction pipeline with **synthetic sensitive content** in CI. A redaction rule that has
  never been tested is a redaction rule that does not work.
- Log the *shape* freely and the *content* never: token counts, chunk counts, score distributions,
  boolean grounding, character-length buckets. Almost all diagnostic value is in the shape.

## 2.6 Sampling policy

**Head-based sampling is prohibited on the verdict path.** A sampled-away span is a hole in an audit
record. But retaining every span of every agent step at full fidelity is not affordable either — agent
traces are enormous.

The resolution is **tiered retention with tail-based sampling**, keeping *identity and outcome* always
and *detail* selectively.

| Tier | What | Sampling | Retention | Rationale |
|---|---|---|---|---|
| **T0 — Audit** | `qa.run`, `qa.document`, `qa.rule`, `qa.verdict`, `qa.review` spans; the change manifest; verdict + evidence refs | **100%, never sampled** | Per §8 retention requirement (≥ regulatory minimum) | This is the record. It must be complete. |
| **T1 — Diagnostic** | All spans of runs that (a) errored, (b) escalated, (c) exceeded a latency threshold, (d) had a failing eval score, (e) were randomly selected | Tail-based, keep 100% of a–d + ~5–10% random | 30 days | Where debugging actually happens |
| **T2 — Bulk** | Everything else, full step detail | ~1% | 7 days | Baseline shape, rare deep dives |
| **T3 — Content** | Prompts, document text, rationales | Externalised, referenced | Per data policy | §2.5 |

**Tail-based sampling requires the collector to buffer complete traces.** For long-running agent traces
this means real memory. Size it, monitor collector queue depth and drop counts, and treat
`otelcol_processor_tail_sampling_sampling_trace_dropped_too_early` (or equivalent) as a **reliability
signal**, not an observability detail. A collector that silently drops traces has quietly become a
compliance defect.

**Sampling and metrics are independent.** Metrics are aggregated at emission and are unaffected by trace
sampling — which is exactly why the §3 SLIs are metric-based and not derived from sampled traces.
Do not build an SLO on a query over sampled traces; you will be measuring your sampler.

## 2.7 Backend routing

| Signal | Splunk Observability | Dynatrace | LLM-native trace store | Evidence store |
|---|---|---|---|---|
| Control-plane metrics + APM traces | ✅ primary | ✅ | — | — |
| `gen_ai.*` spans and token metrics | ✅ AI Agent Monitoring | ✅ AI Observability app | ✅ | — |
| Prompt / trajectory inspection | partial | partial | ✅ **primary** | — |
| Eval scores attached to spans | via `gen_ai.evaluation.result` | via events | ✅ **primary** | — |
| SLOs + burn-rate alerting | ✅ **primary** | ✅ | ⚠️ weak | — |
| Long-horizon audit record | ❌ retention too short | ❌ | ❌ | ✅ **primary** |

**Practical notes for this pairing:**

- **Splunk AI Agent Monitoring** keys off `gen_ai.operation.name` to classify spans, and expects
  delta-temporality metrics with OTLP histograms enabled in the collector. Verify these
  (values current as of writing — re-check against vendor docs, tier P2):
  `OTEL_INSTRUMENTATION_GENAI_EMITTERS=span_metric`,
  `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=delta`, and `send_otlp_histograms: true` on the
  SignalFx exporter. Content capture is gated separately by
  `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT` — keep it off in prod per §2.5.
- **Dynatrace** ingests via standard OTLP endpoints and reads the GenAI conventions directly; the
  AI Observability app gives trace- and token-level views. Its value here is correlating agent
  behaviour with the surrounding infrastructure — the thing the LLM-native tools cannot see.
- **The LLM-native store earns its place on trajectory and eval ergonomics, nothing else.** Do not put
  SLOs there. Vendors change; your objectives should not.
- **Fan-out at the collector, not in the application.** One OTLP export path, multiple exporters. An
  application that knows the name of a backend is an application you cannot re-platform.

---

# 3. The metric catalogue

Every entry declares: **name · type · unit · unit-of-work · dimensions · what it detects**.
Metric names use the `qa.*` namespace for domain signals and `gen_ai.*` where the OTel convention
already defines the right instrument (do not reinvent those — vendor dashboards read them).

Instrument types: `H` histogram, `C` counter, `UD` up-down counter, `G` gauge.

## 3.0 The dashboard hierarchy

Build exactly three levels. More than three and nobody looks at any of them.

```
L1  EXECUTIVE / SERVICE HEALTH  — 8 tiles, one screen, answers "is it working and is it right?"
    run success rate · document p95 latency · false-pass rate (sampled, with CI) · escalation rate
    · cost per document · guardrail intervention rate · reviewer queue age p95 · error budget burn

L2  PLANE DASHBOARDS — one per plane (§1.2). Where you go when L1 goes amber.
    Control: queue depth, fan-out concurrency, retry rate, straggler count, tool error rate
    Cognition: termination_reason distribution, steps/verdict, context utilisation, token velocity, eval scores
    Consequence: escalation mix, override rate, dwell time, queue age, reviewer agreement

L3  DIAGNOSTIC — per-rule-category, per-model, per-tenant breakdowns + exemplar links into traces.
    Not curated. Built ad hoc during investigations. Deleted afterwards.
```

## 3.1 Control plane

| Metric | Type | Unit | Unit of work | Dimensions | Detects |
|---|---|---|---|---|---|
| `qa.run.duration` | H | s | run | `tenant`, `status` | End-to-end SLI. The number the business feels. |
| `qa.run.count` | C | {run} | run | `status` | Traffic |
| `qa.document.duration` | H | s | document | `doc_type`, `status` | The fan-out max (§1.3.1) |
| `qa.rule.duration` | H | s | rule | `rule_category`, `severity` | Tail driver. Track p50/p95/p99/p99.9 |
| `qa.rule.dispatched` | C | {rule} | rule | `rule_category` | **Denominator for everything.** |
| `qa.rule.completed` | C | {rule} | rule | `rule_category`, `termination_reason` | Completeness |
| `qa.rule.coverage_ratio` | H | 1 | document | `doc_type` | `completed / expected`. **< 1.0 means a rule silently vanished** — the most under-detected control-plane failure |
| `qa.fanout.concurrency` | UD | {rule} | — | — | In-flight rule evaluations. Saturation. |
| `qa.fanout.straggler_count` | H | {rule} | document | `doc_type` | Rules exceeding the document deadline budget |
| `qa.queue.depth` | UD | {item} | — | `queue` | Classic saturation |
| `qa.queue.wait` | H | s | item | `queue` | Latency decomposition: wait vs work |
| `qa.tool.duration` | H | s | tool call | `tool_name`, `status` | Alias of `gen_ai.execute_tool.duration`; keep our name for SLOs |
| `qa.tool.errors` | C | {error} | tool call | `tool_name`, `error.type`, `error_class` | `error_class` ∈ {transient, systemic, terminal} — see §9.4 |
| `qa.retry.attempts` | H | {attempt} | logical call | `target`, `error_class` | Retry amplification (§1.3.2). **Alert on the mean, not just the max.** |
| `qa.deadline.exceeded` | C | {event} | rule \| document | `level` | Budget exhaustion |
| `qa.idempotency.replay` | C | {event} | tool call | `tool_name` | Duplicate suppression working — a zero here after a retry storm means duplicates went through |

**Reading guide:** the control plane is the only plane where classic SRE intuition transfers intact.
Debug it with classic SRE technique. `qa.rule.coverage_ratio < 1.0` deserves its own paragraph of your
attention: a rule that was never dispatched produces no error, no span and no verdict — the document
simply gets validated against fewer rules than the checklist claims, and every quality metric stays
green while coverage silently erodes.

## 3.2 Cognition plane — behaviour

These describe *how the agent behaved*, independently of whether it was right. They are cheap,
deterministic, high-frequency, and they move **before** quality metrics do. Treat them as leading indicators.

| Metric | Type | Unit | Unit of work | Dimensions | Detects |
|---|---|---|---|---|---|
| `qa.agent.steps` | H | {step} | agent invocation | `rule_category`, `termination_reason` | **Convergence.** The distribution's *shape* matters more than its mean — a bimodal distribution means two behaviours in one population |
| `qa.agent.step_budget_utilisation` | H | 1 | agent invocation | `rule_category` | `steps_used / step_budget`. Mass near 1.0 = agents fighting the ceiling |
| `qa.agent.terminations` | C | {invocation} | agent invocation | **`termination_reason`**, `rule_category` | The §2.3.1 enum. **Highest information density in the catalogue.** |
| `qa.agent.tool_calls` | H | {call} | agent invocation | `rule_category` | Also available as `gen_ai.invoke_agent.tool_calls` |
| `qa.agent.inference_calls` | H | {call} | agent invocation | `rule_category` | Also `gen_ai.invoke_agent.inference_calls` |
| `qa.agent.repeated_tool_call_ratio` | H | 1 | agent invocation | `rule_category` | Identical tool+args called ≥2×. **Direct instrumentation of MAST FM-1.3 "step repetition" (15.7% of observed failures)** |
| `qa.agent.context_utilisation` | H | 1 | step | `model` | Peak context as fraction of window. Quality degrades before the hard limit |
| `qa.agent.context_overflow` | C | {event} | step | `model` | Hard failures |
| `qa.agent.tool_arg_invalid` | C | {event} | tool call | `tool_name`, `reason` | Model calling tools wrong ≠ tools broken. Spikes after tool-schema changes |
| `gen_ai.client.operation.duration` | H | s | model call | `operation`, `provider`, `model`, `error.type` | OTel standard |
| `gen_ai.client.operation.time_to_first_chunk` | H | s | model call | `operation`, `provider`, `model` | TTFT — prefill + queue |
| `gen_ai.client.operation.time_per_output_chunk` | H | s | model call | `operation`, `provider`, `model` | ITL — decode health. **Rises under provider contention before error rates do** |
| `gen_ai.client.token.usage` | H | {token} | model call | `operation`, `provider`, `model`, `token.type` | Token accounting |
| `qa.model.response_model_mismatch` | C | {event} | model call | `requested`, `served` | `gen_ai.request.model != gen_ai.response.model`. **Silent version change detector** (§5.2) |
| `qa.model.finish_reason` | C | {event} | model call | `reason`, `model` | `length` means truncation — a correctness bug wearing a performance costume |

**Why the behavioural metrics matter more than they look:** they are *free* (no judge model), they are
*deterministic* (no drift), and they are *fast* (available on every invocation, not on a sample). When
a prompt change degrades quality, `qa.agent.steps` and the termination-reason distribution typically
move within minutes, while a sampled quality SLI needs hours or days to reach significance (§4.4).
**Route your fastest alerts through the behavioural metrics and use quality metrics to confirm.**

## 3.3 Cognition plane — grounding and evidence

Deterministic correctness checks. Cheap, non-drifting, high-value. Build these first (§1.5).

| Metric | Type | Unit | Unit of work | Dimensions | Detects |
|---|---|---|---|---|---|
| `qa.evidence.count` | H | {citation} | verdict | `rule_category`, `verdict` | Zero-evidence verdicts on evidence-required rules |
| `qa.evidence.verified_ratio` | H | 1 | verdict | `rule_category` | Fraction of citations whose quoted span **actually exists** in the cited document version |
| `qa.evidence.fabricated` | C | {event} | citation | `rule_category` | Citation to text that does not exist. **Should be ~zero. Any sustained non-zero is a P2 incident.** |
| `qa.evidence.stale_version` | C | {event} | citation | — | Citation resolves against a different document version than the one under validation |
| `qa.grounded_verdict_ratio` | H | 1 | rule_category | `severity` | Fraction of verdicts with ≥1 verified citation where evidence is required |
| `qa.retrieval.empty` | C | {event} | retrieval | `query_type` | Zero results. Leading indicator of `insufficient_evidence` |
| `qa.retrieval.top_score` | H | 1 | retrieval | `query_type` | **Distribution drift here precedes quality drift.** Index rebuilds, embedding-model changes and corpus shifts all show up here first |
| `qa.retrieval.returned` | H | {doc} | retrieval | `query_type` | |

> **Build order, and be strict about it:** `qa.evidence.verified_ratio` and `qa.evidence.fabricated`
> before any LLM judge. They are substring and offset checks. They cost microseconds. They never drift.
> They catch the highest-consequence hallucination class in a compliance system — a confident verdict
> citing a clause that was never written. Teams routinely skip this and go straight to a groundedness
> judge, paying model costs for a weaker signal.

## 3.4 Quality and evaluation plane

This is the plane that does not exist unless you build it. It is what converts "the system responded"
into "the system was right."

### 3.4.1 Three evaluation sources, ranked by trust

| Source | Cost | Latency | Trust | Coverage | Use for |
|---|---|---|---|---|---|
| **Deterministic checks** | ~0 | µs | **Highest** — no drift, no variance | 100% | Schema, grounding, citation existence, budget adherence, format |
| **LLM judges** | $ | seconds | Medium — drifts, has variance, needs calibration | Sampled | Semantic judgements only: reasoning quality, verdict defensibility |
| **Human annotation** | $$$ | hours–days | **Ground truth** | Small sample | Calibrating judges; the false-pass estimate; adjudicating disagreement |

**The trust ordering is a spending rule:** never buy a judge for something a check can prove, and never
trust a judge you have not calibrated against humans.

### 3.4.2 Metrics

| Metric | Type | Unit | Unit of work | Dimensions | Detects |
|---|---|---|---|---|---|
| `qa.verdict.count` | C | {verdict} | verdict | `verdict`, `rule_category`, `severity` | The verdict mix. **Mix shift is a first-class alarm** — see below |
| `qa.verdict.false_pass_rate` | G | 1 | sampled verdicts | `rule_category`, `severity` | 🔴 **The metric that matters most** (§1.6). Estimated from the human-reviewed sample. **MUST be published with a confidence interval** |
| `qa.verdict.false_fail_rate` | G | 1 | sampled verdicts | `rule_category` | Cost and credibility |
| `qa.verdict.accuracy` | G | 1 | sampled verdicts | `rule_category` | Convenience only. **Never alert on this alone** (§1.6 asymmetry trap) |
| `qa.eval.score` | H | 1 | verdict / step | **`eval_name`**, `rule_category` | Generic carrier for every automated evaluator. Emitted as `gen_ai.evaluation.result` events, aggregated into this histogram |
| `qa.eval.judge_agreement` | G | 1 | eval sample | `eval_name` | Judge self-consistency across N repeats. **< 0.8 ⇒ the eval is UNSTABLE and its output is not a measurement** |
| `qa.eval.human_judge_agreement` | G | 1 | calibration set | `eval_name` | Judge vs human. Drives the attenuation correction in §4.4.3 |
| `qa.eval.coverage` | G | 1 | verdicts | `eval_name` | Fraction of verdicts actually evaluated. Your quality SLI's real sample size |
| `qa.confidence.calibration_error` | G | 1 | calibration set | `rule_category` | Expected Calibration Error. **See warning below** |

**Standard `eval_name` values for this system:**

| `eval_name` | Method | What it asks |
|---|---|---|
| `citation_exists` | deterministic | Does the cited text exist in the cited version? |
| `evidence_sufficient` | judge | Does the cited evidence actually support the verdict? |
| `rule_interpretation` | judge | Did the agent interpret the rule as written, or drift? |
| `verdict_defensible` | judge | Would a competent reviewer reach the same verdict on this evidence? |
| `reasoning_action_consistency` | judge | Does the stated reasoning match the tools actually called? (**MAST FM-2.6, 13.2%**) |
| `scope_adherence` | deterministic + judge | Did the agent stay within the rule's scope? (**MAST FM-1.1**) |
| `format_valid` | deterministic | Schema conformance |

> **⚠️ On `qa.verdict.confidence`:** a self-reported confidence score from a language model is not a
> probability until you have demonstrated it is calibrated. Publish `qa.confidence.calibration_error`
> alongside it, or do not emit confidence at all. An uncalibrated confidence score routed into an
> escalation threshold is a *systematically biased sampler* — it will preferentially escalate the cases
> the model finds linguistically uncertain, not the cases where it is actually wrong, and those two sets
> overlap less than you would hope. This is a real and common way for an escalation policy to look
> sophisticated while being close to random.

### 3.4.3 Verdict mix shift — the cheapest early warning you will ever build

`qa.verdict.count` dimensioned by `verdict` gives you a distribution over
{pass, fail, insufficient_evidence, escalate}. That distribution should be **boringly stable** for a
stable rule set against a stable document population.

Track its divergence from a trailing baseline (population stability index or a simple
Jensen–Shannon divergence over the four-way distribution, computed hourly against a 7-day baseline).

**Why it is valuable:** it requires no ground truth, no judge, no sample, and no cost. It responds
within one aggregation window. It catches prompt regressions, silent model swaps, retrieval-index
breakage and document-population shifts — all of which change the mix before anyone can measure accuracy.

**Why it is not sufficient:** it cannot distinguish "the model got worse" from "this week's documents
are genuinely different." It is a **detector**, never a diagnosis. Pair it with
`qa.document.population_drift` (below) to separate the two.

| Metric | Type | Unit | Dimensions | Detects |
|---|---|---|---|---|
| `qa.verdict.mix_divergence` | G | 1 | `rule_category` | Verdict-distribution shift vs trailing baseline |
| `qa.document.population_drift` | G | 1 | `doc_type` | Input drift: page count, doc type mix, language, source system. **The control for the above** |

## 3.5 Consequence plane — the human layer

Almost nobody instruments this. It is where the errors that matter become consequences that matter.

| Metric | Type | Unit | Unit of work | Dimensions | Detects |
|---|---|---|---|---|---|
| `qa.escalation.rate` | G | 1 | verdicts | `escalation_reason`, `severity` | Too low = agent over-confident, humans see nothing. Too high = automation not paying for itself |
| `qa.escalation.queue.depth` | UD | {item} | — | `priority` | Saturation of the human layer — a real, hard capacity limit |
| `qa.escalation.queue.age` | H | s | item | `priority` | **Oldest-item age, not mean.** Starvation hides in the mean |
| `qa.review.dwell` | H | s | review | `reviewer_role`, `verdict` | 🔴 **Rubber-stamp detector.** A mass of reviews at a few seconds means no review happened |
| `qa.review.override.rate` | G | 1 | reviews | `reviewer_role`, `original_verdict` | 🔴 **A trend toward zero is an alarm, not an achievement** |
| `qa.review.agreement` | G | 1 | reviews | `reviewer_role` | Reviewer–agent agreement. Rising toward 1.0 alongside falling dwell time = automation bias |
| `qa.review.override.reason` | C | {event} | override | `reason_code` | Structured override reasons are your **richest** failure-mode dataset. Free labelled training data. |
| `qa.review.probe.catch_rate` | G | 1 | injected probes | `reviewer_role` | **Injected known-bad cases caught by reviewers.** The only direct measurement of whether review is real |
| `qa.review.reopened` | C | {event} | verdict | `source` | Verdicts overturned later by downstream audit — your slowest, truest correctness signal |

### 3.5.1 The rubber-stamp signature

Three metrics moving together, none alarming alone:

```
qa.review.dwell         ↓ falling      (reviews getting faster)
qa.review.override.rate ↓ falling      (reviewers disagreeing less)
qa.review.agreement     ↑ → 1.0        (reviewers agreeing more)
```

Read individually this looks like a **maturing, efficient review process**. Read together it is the
signature of oversight collapsing into a formality — and the collapse accelerates as the agent's
rationales get *better*, because a plausible explanation substitutes for independent evaluation.

**The only reliable countermeasure is adversarial probing.** Inject known-bad verdicts into the review
queue at a low, unannounced rate and measure `qa.review.probe.catch_rate`. This is the difference
between *believing* your human-in-the-loop control works and *knowing* it does. It is also exactly the
kind of control an auditor will ask you to evidence, and "we measure it continuously" is a materially
stronger answer than "we have a documented procedure."

Design notes: probes must be indistinguishable from real cases; probe results must never enter the
production audit record; probe catch rate is a **process metric, not a performance metric for individuals**,
and it must be governed that way or reviewers will (correctly) resent it and game it.

## 3.6 Economic plane

Cost is a reliability signal in agentic systems, not a finance concern. A cost anomaly is usually a
behaviour anomaly you have not detected yet — the agent looping, retrying, or re-reading context.

| Metric | Type | Unit | Unit of work | Dimensions | Detects |
|---|---|---|---|---|---|
| `qa.cost.per_verdict` | H | {currency} | verdict | `rule_category`, `model` | The unit economic. **Watch p99, not mean** — the tail is where loops live |
| `qa.cost.per_document` | H | {currency} | document | `doc_type` | Business unit economic |
| `qa.cost.retry_attributed` | C | {currency} | — | `error_class` | Money spent on retried work. Rises before availability does |
| `qa.cache.hit_ratio` | G | 1 | model call | `model`, `prompt_version` | Cache-read tokens / total input tokens |
| `qa.cache.write_amortisation` | G | 1 | prompt_version | `model` | Reads per write. Below the breakeven, caching **costs** money |
| `qa.tokens.per_verdict` | H | {token} | verdict | `token.type`, `rule_category` | Currency-independent. Survives pricing changes |
| `qa.tokens.reasoning_ratio` | H | 1 | model call | `model` | Reasoning tokens / output tokens. Often the majority of spend, and invisible unless you ask |

### 3.6.1 The cost model, written out

```
cost_per_verdict = Σ_steps [ input_tokens × effective_input_price
                           + output_tokens × output_price ]
                   × retry_multiplier
                   + tool_costs

effective_input_price = h·p_read + w·p_write + (1 − h − w)·p_base

  h = cache-read fraction, w = cache-write fraction
  p_read, p_write, p_base = per-token prices for cached read, cache write, uncached

retry_multiplier = (1 − f^k)/(1 − f)          ← §1.3.2
```

**Caching breakeven.** Caching is only profitable above a threshold number of reads per write.
With write multiplier `Wm` and read multiplier `Rm` relative to base price:

```
reads_per_write  >  (Wm − 1) / (1 − Rm)
```

Worked example — a provider charging 1.25× base to write and 0.10× base to read:

```
(1.25 − 1) / (1 − 0.10) = 0.25 / 0.90 ≈ 0.28 reads per write
```

So caching pays for itself very quickly *with that pricing*. **Plug in your own contract prices —
multipliers differ substantially by provider and by negotiated terms, and some providers charge
nothing to write, which makes the breakeven zero.** Do not carry a memorised constant here.

**Cache killers to guard against** (each turns a 70%+ hit ratio into near-zero, silently):

- A timestamp, request ID or nonce anywhere in the cacheable prefix
- Non-deterministic serialisation of tool schemas — **hash and assert your tool-definition JSON is byte-stable**
- Per-document or per-tenant content placed *before* the stable system prompt instead of after
- Whitespace or template normalisation differences between code paths
- A prompt version bump — expect and **plan for** a hit-ratio trough after every prompt deploy;
  alert on the trough failing to recover, not on the trough itself

`qa.cache.hit_ratio` should be dimensioned by `prompt_version` precisely so this recovery is visible.

## 3.7 Metric-to-question index

The fast lookup. When someone asks the question in column 1, look at column 2.

| Question | Metric |
|---|---|
| Is the system up? | `qa.run.count{status}`, `qa.run.duration` |
| Is it fast enough? | `qa.document.duration` p95/p99 |
| Did every rule actually run? | `qa.rule.coverage_ratio` |
| Is the agent converging? | `qa.agent.terminations{termination_reason}`, `qa.agent.steps` |
| Is it inventing evidence? | `qa.evidence.fabricated`, `qa.evidence.verified_ratio` |
| Is it right? | `qa.verdict.false_pass_rate` (with CI), `qa.verdict.false_fail_rate` |
| Has something changed without a deploy? | `qa.verdict.mix_divergence`, `qa.model.response_model_mismatch`, `qa.change.*` |
| Are humans actually reviewing? | `qa.review.dwell`, `qa.review.override.rate`, `qa.review.probe.catch_rate` |
| Why did cost jump? | `qa.cost.per_verdict` p99, `qa.cache.hit_ratio`, `qa.retry.attempts`, `qa.agent.steps` |
| Is the provider degrading? | `gen_ai.client.operation.time_per_output_chunk`, `qa.tool.errors{error_class=systemic}` |
| Can I prove what happened? | T0 trace + change manifest + evidence store (§8) |

---

# 4. SLOs and error budgets

## 4.1 Two budgets, not one

A single availability error budget cannot govern this system, because the system's characteristic
failure is **being available and wrong**. Run two budgets with separate policies.

| | **Availability budget** | **Quality budget** |
|---|---|---|
| Governs | Did the work happen, on time, completely | Was the work right |
| Measured on | Every request (population) | A **sample**, with a confidence interval |
| Latency of signal | Seconds | Hours to days |
| Statistical character | Exact | Estimated — noise is a first-class concern |
| Alerting | Multi-window burn rate | Statistical burn rate + behavioural leading indicators |
| Exhaustion policy | Freeze feature deploys | **Freeze prompt/model/rule changes AND raise escalation rate** |
| Owner | SRE | SRE + the compliance product owner jointly |

**The quality budget's exhaustion action is different in kind and this is the part people get wrong.**
When availability budget runs out you stop changing things. When *quality* budget runs out you stop
changing things **and you shift work back to humans** by lowering escalation thresholds. The system
degrades gracefully into a slower, more expensive, more accurate mode. Design that mode deliberately,
test it, and make the threshold change a runtime config — not a deploy.

## 4.2 Availability and latency SLOs

| SLI | Definition | Target | Window | Notes |
|---|---|---|---|---|
| **Run completion** | `runs terminal-success / runs accepted` | 99.5% | 28d rolling | "Terminal-success" includes runs completed with escalations. Escalation is a **success**, not a failure |
| **Rule coverage** | `qa.rule.coverage_ratio == 1.0` | **99.95%** | 28d | 🔴 Tightest availability objective in the set. A missing rule is a silent compliance gap (§3.1) |
| **Document latency** | `qa.document.duration` p95 | ≤ target_p95 | 28d | Derive the target from the business commitment, then work backwards to per-rule tails via §1.3.1 |
| **Document latency (tail)** | `qa.document.duration` p99 | ≤ 3 × p95 target | 28d | Constrains the *shape*, not just the level. Prevents "fix the p95 by making the tail worse" |
| **Verdict durability** | verdicts persisted with complete audit record / verdicts produced | **100%** | 28d | Not an SLO with slack. Any loss is an incident. |
| **Evidence-store availability** | successful content writes / attempts | 99.9% | 28d | A verdict without retrievable evidence is not auditable |

**On the latency target:** do not set it from the current p95. Set it from what the downstream human
process needs, then derive the per-rule budget and check §1.3.1 tells you it is achievable. If it does
not, the correct response is an architectural change (batching, tiering rules by cost, deadline +
escalate), not a looser SLO.

## 4.3 Quality SLOs — asymmetric by construction

| SLI | Definition | Target | Window | Rationale |
|---|---|---|---|---|
| **False-pass rate (critical rules)** | P(verdict = pass \| truly non-compliant), severity = critical | **≤ 0.5%** | 90d | The consequential error. Tight target, long window because the sample accrues slowly |
| **False-pass rate (all rules)** | as above, all severities | ≤ 2% | 90d | |
| **False-fail rate** | P(verdict = fail \| truly compliant) | ≤ 8% | 28d | Costly but recoverable. Deliberately ~4–16× looser |
| **Grounded verdict ratio** | verdicts with ≥1 verified citation / verdicts requiring evidence | **≥ 99.5%** | 28d | Deterministic, measured on 100%. No sampling error |
| **Fabricated citation rate** | `qa.evidence.fabricated / citations` | **≤ 0.05%** | 28d | Deterministic. Effectively a bug rate |
| **Escalation calibration** | P(reviewer overturns \| escalated) | **within [15%, 60%]** | 28d | **Two-sided.** Below 15% = escalating cases that did not need it. Above 60% = the agent is wrong in a whole class of cases and you are using humans as a mop |
| **Judge stability** | `qa.eval.judge_agreement` | ≥ 0.85 | 7d | An unstable judge is not a measurement instrument |
| **Review integrity** | `qa.review.probe.catch_rate` | ≥ 80% | 90d | The human control is real, and provably so |

**Read the escalation-calibration SLI twice.** It is two-sided on purpose. A one-sided "escalate more
when unsure" policy converges to escalating everything, which destroys the value of the system while
looking conservative. A one-sided "escalate less" policy converges to the false-pass failure. The band
is the objective.

## 4.4 The statistics of a quality SLO

This is the part that gets hand-waved, and hand-waving it produces SLOs that cannot be measured and
release gates that block on noise. Work through it once.

### 4.4.1 You are estimating, not measuring

Availability is a census: you see every request. Quality is a **survey**: you see a sample of verdicts
that a human or a judge assessed. Therefore every quality number has a confidence interval, and
**a quality SLI published without one is misinformation.**

Use the **Wilson score interval**, not the normal approximation. Near p → 1 — exactly where your quality
metrics live — the normal approximation produces intervals that extend above 1.0 and understates
uncertainty at small counts. Most metric backends do not compute Wilson for you; compute it in the
rollup job that produces the gauge, and emit the bounds as separate series
(`qa.verdict.false_pass_rate.lower`, `.upper`).

### 4.4.2 How much sample do you need?

**For rare events — the right framing for false-pass rate.** The precision of an estimated rate is
governed by the *number of events observed*, not the sample size. Relative 95% CI half-width ≈ `1.96/√k`
for `k` observed events:

| Events observed (k) | Relative precision of the rate | At a true rate of 2%, verdicts to review |
|---|---|---|
| 15 | ±50% | ~770 |
| 62 | ±25% | ~3,100 |
| 384 | ±10% | ~19,200 |

**Read that table and let it land.** To know your false-pass rate to within ±25% relative — which is
barely enough to tell 2% from 2.5% — you need roughly **3,100 human-reviewed verdicts**. This is the
real constraint on quality SLOs, and it is why:

- The false-pass SLO window is **90 days**, not 28.
- Your human-review sample MUST **over-sample PASS verdicts** (§1.6). Reviewing escalations gives you
  almost no false-pass events, so almost no precision on the metric that matters.
- **Stratified sampling is mandatory**, not an optimisation. Sample heavily from critical-severity
  rules and from rule categories with historically higher error rates; weight the estimate back to the
  population. Uniform random sampling wastes most of your review budget on easy cases.
- Fast quality alerting on the false-pass rate is **statistically impossible**. Accept it and route fast
  detection through §3.2 behavioural metrics and §3.4.3 mix shift instead.

**For comparing two arms — the right framing for a release gate.** Per-arm sample size to detect an
absolute difference δ in a proportion near p, at 80% power / 5% significance:

```
n_per_arm  ≈  16 · p(1 − p) / δ²
```

| Baseline p | Detect δ | n per arm |
|---|---|---|
| 0.97 | 0.05 | 186 |
| 0.97 | 0.02 | 1,164 |
| 0.97 | 0.01 | 4,656 |
| 0.99 | 0.01 | 1,584 |
| 0.99 | 0.005 | 6,336 |

**Implication for release gates:** a 200-case eval set can detect a 5-point regression. It **cannot**
detect a 1-point regression, and a green run on 200 cases is not evidence that a 1-point regression is
absent. State the **minimum detectable effect** of your eval suite in the gate output. A gate that
reports "PASSED (94.5% vs 95.0%, n=200, MDE=5.0pp — this run cannot detect regressions smaller than
5pp)" is honest. One that reports "PASSED ✅" is not.

### 4.4.3 Judge noise attenuates the effect you are trying to detect

If your quality measurement comes from an LLM judge rather than a human, the judge's own error rates
bias the measurement **toward the middle** and shrink observed differences.

Let `α_J = P(judge says pass | truly fail)` and `β_J = P(judge says fail | truly pass)`. Then:

```
p_observed = p·(1 − β_J) + (1 − p)·α_J

∂p_observed/∂p  =  1 − α_J − β_J  ≡  a        (the attenuation factor; Youden's J)
```

A true difference δ appears as `a·δ`. Sample size therefore inflates by **1/a²**:

| Judge total error (α_J + β_J) | a | Sample-size multiplier |
|---|---|---|
| 0.05 | 0.95 | **1.11×** |
| 0.07 | 0.93 | 1.16× |
| 0.20 | 0.80 | **1.56×** |
| 0.30 | 0.70 | **2.04×** |
| 0.50 | 0.50 | 4.00× |

Practitioner reporting of ~7% verdict flips on *frozen, unchanged* responses (P3) is a reasonable
starting assumption for an uncalibrated judge. Two consequences:

- **Calibrate every judge against human labels and publish `a`.** Then correct your sample sizes by 1/a².
- **Judge *bias* shifts the level; judge *noise* shrinks the deltas.** Absolute quality targets are
  therefore more fragile than relative comparisons. Prefer **"is this release worse than the last one"**
  over **"is this release above 97%"** wherever the decision allows it.

### 4.4.4 Judge stability is a gate, not a metric

Before a judge's output is allowed to influence any decision, run it N times (N=5 is a reasonable
floor) on the same frozen inputs and compute self-agreement.

- **Agreement ≥ 0.85** → the judge is a measuring instrument. Use it.
- **Agreement < 0.8** → the case is **`UNSTABLE`**. `UNSTABLE` counts as a **failure**, not as a partial
  pass. A 95% pass rate produced by a judge with 0.6 self-agreement is a random number with good
  presentation.

Sources of judge instability, in the order you should check them: vague rubric (fix the rubric — this is
most of it); non-determinism at the provider even at temperature 0 (raise N, or move the check to a
deterministic assertion); harness drift changing what the judge actually sees between runs (fix the
harness — this one masquerades as model noise and is not).

### 4.4.5 Alert on the lower bound; gate on the upper bound

This asymmetry falls straight out of what each decision is protecting against, and it is worth
committing to memory:

- **Paging** protects against noise-driven interruption. Page only when the **lower** confidence bound
  of the error rate exceeds the threshold — i.e. you are confident things really are bad.
- **Release gating** protects against shipping a regression. Block when the **upper** confidence bound
  of the error rate exceeds the threshold — i.e. you require confidence that things are fine.

Same estimate, opposite bound, because the cost of being wrong points in opposite directions.
Encode it in the tooling so nobody has to remember it.

## 4.5 Burn-rate alerting

**Availability budget** — standard multi-window, multi-burn-rate. Nothing special:

| Burn rate | Long window | Short window | Budget consumed | Action |
|---|---|---|---|---|
| 14.4× | 1 h | 5 m | 2% | Page |
| 6× | 6 h | 30 m | 5% | Page |
| 3× | 24 h | 2 h | 10% | Ticket |
| 1× | 72 h | 6 h | 10% | Ticket |

**Quality budget** — the fast windows are unusable, because at realistic sampling rates a 1-hour window
contains too few reviewed verdicts to distinguish a burn from noise. Substitute a two-speed structure:

| Speed | Signal | Window | Action |
|---|---|---|---|
| **Fast (proxy)** | `qa.verdict.mix_divergence`, `qa.agent.terminations` distribution shift, `qa.evidence.verified_ratio` drop, `qa.retrieval.top_score` shift, `qa.model.response_model_mismatch` | 15–60 min | **Page.** These are cheap, deterministic, high-frequency, and they move first |
| **Slow (truth)** | False-pass / false-fail rate with Wilson bounds, judge-scored evals | 24 h – 7 d | Ticket; escalate to page if the lower bound crosses threshold |

**The mental model:** behavioural metrics are your smoke detector, quality metrics are your fire
investigation. You cannot make the fire investigation fast, so make the smoke detector good. Every
proxy in the fast row is free and deterministic — there is no excuse for not having all of them.

## 4.6 Error budget policy

| Budget state | Availability policy | Quality policy |
|---|---|---|
| **> 50% remaining** | Normal | Normal |
| **25–50%** | Normal; review recent changes | Freeze **threshold** changes; require a canary for prompt/model changes |
| **10–25%** | Feature freeze; reliability work prioritised | Freeze prompt, model, rule-set and judge changes. Raise the escalation rate one step |
| **< 10%** | Change freeze except reliability fixes | **Raise escalation rate to the defensive profile.** Notify the compliance owner. Written remediation plan required |
| **Exhausted** | Full freeze; incident review | **Defensive mode**: escalate all critical-severity verdicts to human review regardless of confidence. This is a business decision with a cost — pre-agree it with the compliance owner **now**, not during the incident |

**Two governance rules that make this real rather than decorative:**

1. **The defensive escalation profile MUST be a runtime configuration change**, exercised in a game day
   at least quarterly. A degradation mode that has never been executed does not exist.
2. **A change to a *threshold* is a change to the system.** It goes in the change manifest (§2.3), it
   goes through the release gate (§7), and it consumes budget like any other change. Threshold changes
   are the most common way an agentic system is silently altered in production, precisely because they
   do not look like deploys.

---

# 5. The failure mode catalogue

## 5.0 How to use this section

Each mode carries a **discriminator** — the observation that separates it from the modes it is most
easily confused with. In agentic incident response, misdiagnosis is the dominant time sink, because
several very different faults produce the same top-level symptom ("quality dropped", "it got slow",
"cost went up"). **Read the discriminator before you read the mitigation.**

### 5.0.1 Grounding in the published taxonomies

Two independent taxonomies inform the cognition-plane modes below, and it is worth knowing what each
is good for:

- **MAST** (Multi-Agent System Failure Taxonomy — 3 categories, 14 modes, 150 hand-annotated traces at
  κ = 0.88, extended to 1,600+ traces across 7 frameworks). Strength: it is *empirical and weighted*,
  so it tells you which failures actually happen and how often. Caveat: it was derived from
  multi-agent systems; your architecture is a deterministic orchestrator over single agents, which
  **eliminates some modes and concentrates others** — see the mapping below.
- **TRAIL** (Trace Reasoning and Agentic Issue Localization). Strength: it is organised around what is
  *visible in a trace*, which makes it the better fit for instrumentation design. Its categories —
  reasoning errors, system execution errors, planning & coordination errors — map cleanly onto the
  three-plane model in §1.2.

**The architecture-specific reading of MAST.** Your design deliberately removes the inter-agent
communication surface: a deterministic orchestrator, one agent per rule, no agent-to-agent chat. That
buys real safety — most of MAST's FC2 (Inter-Agent Misalignment, ~32%) does not apply. But two things
carry over regardless of architecture, and you should expect them to be **over-represented** in your
system precisely because the others are gone:

- **FM-2.6 Reasoning–action mismatch (13.2%)** — the agent's stated rationale does not match the tools
  it actually called. This is a *single-agent* failure. It is fully in scope, and in a compliance
  system it is severe: the rationale is the audit artifact.
- **All of FC3 Task Verification (~24%)** — premature termination, no/incomplete verification,
  incorrect verification. Single-agent systems have *fewer* checks, not more. Removing peer agents
  removed a (weak, unreliable) cross-check. Your verification burden moved into the deterministic
  guardrails and the human layer, which is exactly why §3.3 and §3.5 exist.

| MAST mode | % | Applies here? | Observable signal in this system |
|---|---|---|---|
| FM-1.1 Disobey task specification | 11.8 | ✅ high | `scope_adherence` eval; rule-category-scoped false-fail spike |
| FM-1.2 Disobey role specification | 1.5 | ⚠️ low | Verdict schema violations; `format_valid` |
| FM-1.3 Step repetition | 15.7 | ✅ **high** | `qa.agent.repeated_tool_call_ratio`, `qa.agent.steps` right tail |
| FM-1.4 Loss of conversation history | 2.8 | ✅ medium | `qa.agent.context_utilisation` near 1.0; post-compaction quality drop |
| FM-1.5 Unaware of stopping conditions | 12.4 | ✅ **high** | `termination_reason = step_budget_exhausted` |
| FM-2.1 Conversation reset | 2.2 | ❌ | — (no multi-agent conversation) |
| FM-2.2 Fail to ask for clarification | 6.8 | ✅ **high (transformed)** | Manifests as *not* emitting `insufficient_evidence` — low `insufficient_evidence` rate with low `evidence.verified_ratio` |
| FM-2.3 Task derailment | 7.4 | ✅ medium | `scope_adherence` eval; unexpected tool usage patterns |
| FM-2.4 Information withholding | 0.85 | ❌ | — |
| FM-2.5 Ignored other agent's input | 1.9 | ❌ | — |
| FM-2.6 Reasoning–action mismatch | 13.2 | ✅ **high** | `reasoning_action_consistency` eval; rationale citing tools absent from the span tree |
| FM-3.1 Premature termination | 6.2 | ✅ **high** | Low `qa.agent.steps` + low `evidence_count` + `verdict_reached` |
| FM-3.2 No/incomplete verification | 8.2 | ✅ **high** | `qa.evidence.verified_ratio`; guardrail span absent |
| FM-3.3 Incorrect verification | 9.1 | ✅ **high** | Guardrail passed a verdict a human overturned — `qa.review.override` on non-escalated |

**The single most important line in that table:** FM-2.2 does not disappear in a single-agent system —
it *transforms*. Instead of failing to ask a peer agent for clarification, the agent fails to admit it
lacks evidence and manufactures a verdict instead. **A suspiciously low `insufficient_evidence` rate is
not a sign of a capable agent. It is a sign of an agent that has no honest way to say "I don't know" —
or has learned not to.**

---

## 5.1 F1 — Control plane failures

| ID | Failure | Symptom | Primary signals | Discriminator | Mitigation |
|---|---|---|---|---|---|
| **F1.1** | **Silent rule drop** | Document validated against fewer rules than the checklist claims. No error anywhere | `qa.rule.coverage_ratio < 1.0`; `dispatched ≠ completed` | Coverage ratio is the *only* signal. Quality metrics stay green because the dropped rule produced no wrong verdict — it produced no verdict | Explicit dispatch ledger per document; reconcile completed against expected before aggregation; **fail the document** rather than returning a partial result |
| **F1.2** | Fan-out saturation | Document latency climbs; per-rule latency normal | `qa.fanout.concurrency` at ceiling; `qa.queue.wait` ≫ `qa.rule.duration` | **Wait time dominates work time.** Per-rule duration is *flat* | Raise concurrency only if provider quota allows (else you move the queue); admission control; shed to a slower tier |
| **F1.3** | Straggler tail | p99 document latency ≫ p95; small number of very slow rules | `qa.fanout.straggler_count`; `qa.rule.duration` p99.9 | A handful of rules, often the same `rule_category`, dominate | Per-rule deadline + `escalate`; hedge the slowest decile against a second provider; tier expensive rules |
| **F1.4** | **Retry storm** | Everything slows; provider errors climb; cost spikes | `qa.retry.attempts` mean rising; `qa.tool.errors{error_class=systemic}`; `qa.cost.retry_attributed` | Retry *mean* rising while request rate is flat = amplification, not load | Failure-class routing (§9.4); full jitter; **deadline-bounded**, not attempt-bounded; per-provider circuit breaker |
| **F1.5** | Duplicate side effects | Same verdict written twice; downstream inconsistency | `qa.idempotency.replay` at zero after a retry storm | Retries occurred but no replays were suppressed ⇒ the key is wrong or absent | Idempotency key from `(run_id, rule_id, step_index)`; make it a required attribute on mutating tools (§10.1) |
| **F1.6** | Aggregation on partial results | Document marked complete with missing verdicts | `coverage_ratio < 1.0` **and** document status = success | Success status coexisting with incomplete coverage | Aggregation MUST assert completeness; typed **degraded** result rather than silent truncation |
| **F1.7** | Collector trace loss | Traces incomplete; audit gaps | Collector queue depth, `..._trace_dropped_too_early`, refused spans | Gaps correlate with collector memory pressure, **not** application errors | Size tail-sampling buffers for long agent traces; alert on collector drops as a **reliability** signal (§2.6) |
| **F1.8** | Baggage loss | Spans missing `qa.run.id` / `qa.rule.id`; joins fail | Correlation-key null rate | Usually appears at an async or thread-pool boundary | Context propagation test in CI; assert non-null correlation keys at span creation |

### F1.1 expanded — the silent rule drop

**Why it deserves the expansion:** it is the only failure in this catalogue that produces *no error and
no wrong answer*. It produces an *absence*, and absences are invisible to every signal that measures
what happened rather than what should have happened.

A checklist of 60 rules expands, 57 dispatch successfully, 3 fail to enqueue because of a transient
broker error that the orchestrator logs at WARN and moves past. The document aggregates 57 passes.
Every verdict is correct. Every latency is normal. The document is certified compliant against a
checklist it was never fully checked against.

**In a compliance system this is the highest-severity control-plane failure and it will not page you
unless you build the coverage assertion.**

- Emit `expected_rule_count` at checklist build. Emit `completed_rule_count` at aggregation.
- Assert equality. On inequality: fail the document, do not degrade it.
- SLO it at 99.95% (§4.2) — tighter than run completion, because the failure is silent.
- Test it: chaos-inject dispatch failures in staging and confirm the document *fails* rather than
  quietly shrinking.

---

## 5.2 F2 — Provider and model infrastructure failures

| ID | Failure | Symptom | Primary signals | Discriminator | Mitigation |
|---|---|---|---|---|---|
| **F2.1** | **Silent model version change** | Quality shifts; **no deploy**; nothing in the change log | `qa.model.response_model_mismatch`; `qa.verdict.mix_divergence`; step-count distribution shift | `gen_ai.request.model ≠ gen_ai.response.model`, or response model changed with request model constant | **Pin snapshot identifiers, never floating aliases.** Alert on any change in the `response.model` distribution. Keep a frozen canary eval set running continuously against production config |
| **F2.2** | Provider systemic degradation | Latency up, errors up, fleet-wide | `time_per_output_chunk` rising; 529/5xx rate; `error_class=systemic` | **ITL rises before error rate.** Correlated across all callers — not one tenant, not one rule | Per-provider circuit breaker; failover to secondary; degrade to defensive escalation rather than degrading quality |
| **F2.3** | Rate limiting (429) | Latency up; throughput capped | 429 rate; `qa.retry.attempts`; `qa.queue.wait` | 429s are **transient** and carry `retry-after`. Honouring it beats backoff arithmetic | Honour `retry-after`; token-bucket client-side admission control; per-tenant quotas so one run cannot starve others |
| **F2.4** | Context overflow | Invocations abort mid-loop | `termination_reason = context_overflow`; `context_utilisation` → 1.0 | Correlates with document size or step count, not with provider health | Context budget per step; summarise-and-carry rather than accumulate; chunk large documents |
| **F2.5** | **Context degradation (pre-overflow)** | Quality drops with no error at all | `context_utilisation` high band; quality metrics sliced by utilisation decile | 🔴 Quality is a function of context occupancy **well below the hard limit** | Slice every quality metric by context-utilisation decile — you cannot see this otherwise. Cap effective context below the model limit. Compact deliberately |
| **F2.6** | Output truncation | Verdicts malformed or partial | `qa.model.finish_reason{reason=length}` | `finish_reason = length` — often mis-triaged as a parsing bug | Raise max output tokens; shorten required output; **assert on finish reason before parsing** |
| **F2.7** | Cross-region / capacity routing shift | Latency profile changes; nothing else does | `server.address` distribution; TTFT by region | TTFT changes while ITL is stable ⇒ routing/queueing, not decode | Pin region where contractually possible; alert on endpoint-distribution shift |

### F2.5 expanded — context degradation is the quality bug that looks like nothing

Model quality does not fall off a cliff at the context limit; it erodes as occupancy rises. An agent at
80% context utilisation is measurably worse than the same agent at 20%, with **identical** infrastructure
telemetry: same latency band, same token rates, no errors, no truncation.

This failure is invisible unless you deliberately look for it, and there is exactly one way to look:

> **Slice every quality metric by `qa.agent.context_utilisation` decile.**

If quality degrades monotonically across deciles, you have a context-engineering problem, not a model
problem — and buying a bigger model will not fix it, while compaction, retrieval tightening or rule
decomposition will. This single slice has more diagnostic power per unit of effort than almost anything
else in this document, and it is one query.

Related: agents accumulate context across steps, so `context_utilisation` and `qa.agent.steps` are
correlated. A loop (§5.3) therefore *causes* context degradation, which degrades the reasoning, which
extends the loop. **This feedback loop is the mechanism behind most "the agent got confused and
spiralled" reports** — and it is why a step budget is a quality control, not just a cost control.

---

## 5.3 F3 — Cognition plane: trajectory and convergence

| ID | Failure | Symptom | Primary signals | Discriminator | Mitigation |
|---|---|---|---|---|---|
| **F3.1** | **Loop / step repetition** (MAST FM-1.3, 15.7%) | Cost and latency up; verdict eventually arrives or budget exhausts | `qa.agent.repeated_tool_call_ratio`; `steps` right tail; `step_budget_utilisation` → 1.0 | **Identical tool + identical args, ≥2×.** Distinguishes a loop from legitimate multi-step work | Detect repeats in the harness and inject a "you already called this, here is the result" observation; hard step budget; alert on repetition ratio, not just step count |
| **F3.2** | Budget exhaustion (MAST FM-1.5, 12.4%) | No verdict; escalation | `termination_reason = step_budget_exhausted` rate | Distinct from `deadline_exceeded` — hit the *step* ceiling, not the clock | Investigate *why*: usually F3.1 or a rule that genuinely needs more steps. Do not just raise the budget — that converts a fast failure into a slow expensive one |
| **F3.3** | **Premature termination** (MAST FM-3.1, 6.2%) | Confident verdict, thin evidence | Low `steps` **and** low `evidence_count` **and** `verdict_reached` | 🔴 Looks like **efficiency**. It is the most dangerous-looking-good signature in the system | Minimum-evidence gate before a verdict is accepted; require ≥1 verified citation for evidence-required rules |
| **F3.4** | **Reasoning–action mismatch** (MAST FM-2.6, 13.2%) | Rationale describes work the agent did not do | `reasoning_action_consistency` eval; rationale mentions tools absent from the span tree | Compare the rationale text against the actual `execute_tool` children — **partly deterministic** | Cheap deterministic pre-check: extract tool mentions from the rationale, diff against the span tree, flag mismatches for judge review |
| **F3.5** | Scope drift (MAST FM-1.1 / FM-2.3) | Agent validates something adjacent to the rule | `scope_adherence`; false-fail spike in one `rule_category` | Concentrated in a `rule_category`, often after a rule-text edit | Tighten rule text; add negative examples; add a scope assertion to the output guardrail |
| **F3.6** | **Manufactured certainty** (MAST FM-2.2, transformed) | `insufficient_evidence` rate near zero | `insufficient_evidence` rate ↓ while `evidence.verified_ratio` ↓ | 🔴 **Both falling together.** Either alone is ambiguous; together it is diagnostic | Make `insufficient_evidence` a first-class, *rewarded* outcome; include such cases in the eval set; check the prompt does not penalise uncertainty |
| **F3.7** | Tool-argument malformation | Tool errors that are not the tool's fault | `qa.agent.tool_arg_invalid`; `qa.tool.arg_validation = invalid_schema` | Tool returns a validation error, not a runtime error. **Spikes right after tool-schema changes** | Version tool schemas; hash them into the change manifest; treat a schema change as a prompt change with an eval gate |
| **F3.8** | Bimodal trajectory population | `steps` histogram has two peaks | `qa.agent.steps` distribution shape | Two behaviours in one metric — means are meaningless | Find the splitting dimension (`rule_category`, doc type, cache state, model). **Then split the metric.** Never average across a bimodal population |

### F3.3 expanded — premature termination is the failure that looks like success

A rule evaluated in 2 steps with 1 citation, returning a confident `pass`, is either an easy rule
handled efficiently or a hard rule handled negligently. **The metrics cannot tell these apart without
context.** That is why:

- `qa.agent.steps` and `qa.evidence.count` must always be read **together**, and both sliced by
  `rule_category`. Establish the expected band per category; alert on downward drift, not just upward.
- A minimum-evidence gate is the mitigation, and it must be **deterministic**: a verdict on an
  evidence-required rule with zero verified citations is rejected by the guardrail, full stop.
- This mode is a **strong candidate for adversarial probing**: seed documents where the evidence is
  deliberately buried deep and measure whether the agent finds it or terminates early. Track the catch
  rate as a standing quality metric, not a one-off test.

Note the interaction with cost pressure: any optimisation that reduces steps or tokens — a cheaper
model, a tighter prompt, a lower step budget — pushes the system toward this failure mode.
**Never ship a cost optimisation without re-measuring `qa.evidence.count` and false-pass rate.** Cost
optimisations in agentic systems are quality changes wearing a finance disguise.

---

## 5.4 F4 — Grounding and evidence failures

| ID | Failure | Symptom | Primary signals | Discriminator | Mitigation |
|---|---|---|---|---|---|
| **F4.1** | **Fabricated citation** | Verdict cites text that does not exist | `qa.evidence.fabricated` > 0 | Deterministic substring/offset check fails | Deterministic verification on **every** citation; reject the verdict at the guardrail; **any sustained non-zero rate is a P2 incident** |
| **F4.2** | Stale-version citation | Citation resolves in a *different* version of the document | `qa.evidence.stale_version` | Text exists — in the wrong version | Pin `qa.document.version` through the whole path; verify citations against the pinned version only |
| **F4.3** | Retrieval miss | Evidence exists but was never retrieved | `qa.retrieval.empty`; `insufficient_evidence` rate up; `retrieval.top_score` distribution down | Retrieval returns nothing or low scores; the document *does* contain the clause | Hybrid retrieval; query expansion; **index-version alerting**; re-rank |
| **F4.4** | **Index regression** | Broad quality drop after an index rebuild | `retrieval.top_score` distribution shift; `retrieval.index_version` change in the manifest | 🔴 Quality change **correlates with index version, not with code or prompt version** | Treat the index as a deployable artifact: version it, canary it, eval-gate it. Keep the previous index warm for instant rollback |
| **F4.5** | Evidence–verdict mismatch | Citation is real but does not support the verdict | `evidence_sufficient` judge eval; overturn rate on non-escalated verdicts | Citation verifies deterministically but is semantically irrelevant. **This is the residual that judges are genuinely needed for** | Judge eval on a sample; over-sample the PASS verdicts |
| **F4.6** | Chunk-boundary truncation | Evidence split across chunks; agent sees half a clause | `retrieval.returned` normal, `top_score` normal, quality poor for long clauses | Concentrated on rules whose evidence is structurally long (tables, multi-paragraph clauses) | Overlapping chunks; structure-aware chunking; whole-clause retrieval for evidence-required rules |

---

## 5.5 F5 — Verification failures

The MAST FC3 cluster (~24% of observed failures), plus the guardrail equivalents.

| ID | Failure | Symptom | Primary signals | Discriminator | Mitigation |
|---|---|---|---|---|---|
| **F5.1** | No verification performed (FM-3.2, 8.2%) | Verdict emitted with no check | Guardrail span **absent**; `evidence_verified = 0` | The span is missing, not failing. **Only visible because guardrails are spans (§2.1)** | Make the guardrail structurally unskippable; assert its presence in the aggregation step |
| **F5.2** | Incorrect verification (FM-3.3, 9.1%) | Guardrail passes something a human overturns | Overturn rate on **non-escalated** verdicts | The check ran and was wrong — worse than no check, because it manufactured confidence | Test guardrails against a labelled adversarial set; version them into the change manifest; measure guardrail precision/recall like any classifier |
| **F5.3** | Guardrail over-trigger | Escalation rate spikes; throughput collapses | `termination_reason = guardrail_blocked` rate; `escalation.rate` | Escalation rises with **no** change in verdict quality | Tune with a labelled set, never by feel; measure guardrail false-positive rate explicitly |
| **F5.4** | Guardrail bypass on error path | Verdicts escape checks when a guardrail errors | Guardrail error rate; verdicts with no guardrail span | 🔴 **Fail-open.** The most dangerous default in the system | **Guardrails MUST fail closed** → route to `escalate`, never to `pass`. Test the error path explicitly |

> **F5.4 is worth a governance rule of its own.** Write down, for every guardrail, what happens when the
> guardrail *itself* fails. If the answer is "the verdict proceeds", you have built a control that
> disappears exactly when the system is unhealthy — which is when you need it most. In a compliance
> system, a failed check MUST resolve to human review.

---

## 5.6 F6 — Economic failures

| ID | Failure | Symptom | Primary signals | Discriminator | Mitigation |
|---|---|---|---|---|---|
| **F6.1** | **Cache collapse** | Cost up 3–5×; latency up; quality unchanged | `qa.cache.hit_ratio` falls off a cliff | 🔴 Cost/latency move; **quality does not**. Almost always follows a prompt or tool-schema change | Byte-stable tool serialisation; no timestamps/nonces in the cacheable prefix; dimension hit ratio by `prompt_version`; alert on failure to recover after a deploy |
| **F6.2** | Loop-driven cost blowout | `cost.per_verdict` p99 explodes; mean barely moves | `cost.per_verdict` p99; `qa.agent.steps` right tail | **p99 moves, mean does not.** A small population of runaway invocations | Step budget; loop detection (F3.1); alert on the p99, never the mean |
| **F6.3** | Retry-attributed spend | Cost up during degradation, over and above the incident | `qa.cost.retry_attributed`; `qa.retry.attempts` | Cost rise tracks provider error rate | Sunk-token awareness — do not re-bill a partially streamed generation; deadline-bounded retries |
| **F6.4** | Reasoning-token blowout | Output cost up; visible output length unchanged | `qa.tokens.reasoning_ratio` | Reasoning tokens are billed but not displayed — **invisible unless instrumented** | Capture `gen_ai.usage.reasoning.output_tokens`; budget reasoning effort per rule category |
| **F6.5** | Model-tier drift | Cost creeps up over weeks with no single change | Cost per verdict by `model`; model mix distribution | Gradual mix shift toward an expensive tier, often via a fallback path firing more often | Alert on model-mix distribution; make fallback-to-expensive-model an explicitly counted event |

---

## 5.7 F7 — Consequence plane: the human layer

| ID | Failure | Symptom | Primary signals | Discriminator | Mitigation |
|---|---|---|---|---|---|
| **F7.1** | **Rubber-stamp review** | Reviews complete fast, overrides approach zero | `qa.review.dwell` ↓ + `override.rate` ↓ + `agreement` ↑ | 🔴 **All three together.** Individually each reads as improvement (§3.5.1) | Adversarial probes with a tracked catch rate; require structured override reasons; **show consequence before recommendation** in the review UI; route by risk, not by default |
| **F7.2** | Reviewer queue starvation | Old items never reviewed; SLA breaches | `escalation.queue.age` **max**, not mean | Mean age looks fine while the oldest items age indefinitely — a priority-inversion signature | Age-based priority boost; alert on oldest-item age; cap queue depth and shed upstream by raising the agent's confidence bar |
| **F7.3** | Escalation collapse | Escalation rate drifts down; throughput improves | `escalation.rate` ↓ with `false_pass_rate` flat-or-unknown | Often follows a threshold change **or** a confidence-calibration drift. Looks like the automation getting better | Two-sided escalation-calibration SLO (§4.3); thresholds in the change manifest |
| **F7.4** | Escalation flood | Reviewer queue saturates; latency SLO breaks | `escalation.rate` ↑; `queue.depth` ↑ | Human capacity is a hard ceiling and does not autoscale | Pre-agreed defensive/normal profiles; prioritise by rule severity; **admission control at intake**, not at the queue |
| **F7.5** | Override signal loss | Overrides happen but teach nothing | `override.reason` cardinality ≈ 1, or free-text only | Reviewers overriding without structured reasons | Structured `reason_code` enum, mandatory; feed overrides into the eval set as labelled cases. **This is your cheapest source of high-quality labels** |
| **F7.6** | Reviewer drift | Two reviewers, two standards | Override rate by `reviewer_role`/cohort; inter-reviewer agreement on shared cases | Variance across reviewers on the *same* case type | Periodic shared-case calibration rounds; publish inter-reviewer agreement |

---

## 5.8 F8 — Evaluation plane failures (the observer breaks)

The most under-appreciated group. When the evaluation plane fails, **every other metric in this document
becomes unreliable at once** — and it fails silently, because nothing evaluates the evaluator.

| ID | Failure | Symptom | Primary signals | Discriminator | Mitigation |
|---|---|---|---|---|---|
| **F8.1** | **Judge drift** | Eval scores move; system unchanged | `qa.eval.score` shift with `qa.change.*` constant; `human_judge_agreement` falling | 🔴 Scores move with **no** change in the change manifest | Pin the judge model snapshot; version the judge in the manifest; run a **frozen golden set** through the judge on a schedule and alert on score shift |
| **F8.2** | Judge instability | Same input, different verdicts | `qa.eval.judge_agreement` < 0.8 | Re-running frozen inputs produces different scores | Fix the rubric first (most of it); raise N; move deterministic sub-checks out of the judge |
| **F8.3** | Eval-set contamination | Evals improve, production does not | Divergence between eval score and production quality | Offline metric and online metric decouple | Hold out a set never used for prompt iteration; rotate; keep production-sampled cases as the primary source |
| **F8.4** | **Goodhart drift** | Optimised metric improves, outcome worsens | Eval score ↑ while `override.rate` or `reopened` ↑ | 🔴 The **inverse correlation** between the optimised metric and the outcome metric | Keep at least one outcome metric that is *not* an optimisation target (`qa.review.reopened` is ideal — it is downstream and expensive to game) |
| **F8.5** | Coverage collapse | Quality SLI silently based on almost nothing | `qa.eval.coverage` falling | The rate looks stable because the denominator shrank | **Publish sample size and CI with every quality number** — §4.4.1 makes this non-optional |
| **F8.6** | Calibration decay | Confidence stops predicting correctness | `qa.confidence.calibration_error` rising | Escalation routing becomes near-random while looking principled | Recalibrate on a schedule; block threshold changes when ECE exceeds a bound |

### F8.4 expanded — Goodhart drift, and the metric you must not optimise

Every quality metric you optimise stops measuring quality, at a rate proportional to how hard you
optimise it. This is not cynicism; it is the predictable result of iterating prompts against a score.

The defence is structural: **maintain at least one outcome metric that nobody is allowed to target.**

`qa.review.reopened` — verdicts overturned later by downstream audit or a customer challenge — is the
best candidate in this system. It is slow, expensive, downstream of everything, and effectively
impossible to game from inside the pipeline. Watch for the signature: eval scores improving over a
quarter while `reopened` is flat or rising. That divergence means your evaluation plane has drifted from
the thing you actually care about, and every green dashboard between them is decoration.

---

# 6. Diagnostic playbooks

## 6.0 The triage tree — first 120 seconds

Answer these in order. **Do not skip to the interesting hypothesis.** The boring answer is more likely
and much cheaper to rule out.

```
1. Did anything change?  →  qa.change.* on recent runs vs the previous stable window
   │  Compare all seven surfaces: prompt, model, tool schema, rule set, index, judge, thresholds.
   │  A change in ANY of them, including one nobody called a "deploy", explains most incidents.
   └─ CHANGED → §7.5 rollback decision. Do not debug forward past a known change.

2. Is the control plane healthy?  →  coverage_ratio, queue depth/wait, retry mean, tool error rate
   └─ NO → §6.1 (latency) or §6.7 (provider). Classic SRE. Stop looking at the model.

3. Is the agent behaving normally?  →  termination_reason distribution, steps, context_utilisation
   └─ NO → §6.3 (convergence). Behavioural, deterministic, fast.

4. Is the evidence sound?  →  evidence.verified_ratio, fabricated count, retrieval.top_score
   └─ NO → §6.4 (grounding).

5. Is the evaluation plane trustworthy?  →  judge_agreement, eval coverage, human_judge_agreement
   └─ NO → §6.8. FIX THIS FIRST. Every quality number is currently unreliable.

6. Only now: is quality actually worse?  →  false_pass/false_fail with CIs, mix divergence
   └─ §6.2.
```

**Step 5 before step 6 is deliberate.** More than once you will find that the "quality incident" is an
evaluation incident, and hours spent debugging the agent were spent debugging a working system with a
broken thermometer.

---

## 6.1 Playbook — document latency regression

**Trigger:** `qa.document.duration` p95 or p99 above objective.

**First 5 minutes — decompose before you theorise:**

| Component | Query | Verdict |
|---|---|---|
| Queue wait | `qa.queue.wait` p95 vs `qa.rule.duration` p95 | Wait ≫ work ⇒ **F1.2 fan-out saturation** |
| Straggler count | `qa.fanout.straggler_count` | Elevated ⇒ **F1.3 straggler tail** |
| Rule count per document | `qa.rule.dispatched` per document | Rose ⇒ rule-set change, not a regression (check manifest) |
| Steps per invocation | `qa.agent.steps` p95 | Rose ⇒ **F3.1 loop** — a *cognition* problem presenting as latency |
| TTFT | `time_to_first_chunk` p95 | Rose alone ⇒ prefill/queueing at the provider (**F2.7**) |
| ITL | `time_per_output_chunk` p95 | Rose ⇒ **F2.2 provider degradation**. Earliest reliable provider signal |
| Retry mean | `qa.retry.attempts` mean | Rose ⇒ **F1.4 retry storm** |

**Ranked causes:** ① more steps per invocation (cognition, not infra) ② provider ITL degradation
③ fan-out saturation ④ retry amplification ⑤ rule-count growth ⑥ document population shift (bigger docs).

**Containment, in order of preference:** enforce per-rule deadlines → escalate; shed load via admission
control at intake; fail over the affected provider; **only then** raise concurrency (raising concurrency
against a provider limit moves the queue rather than draining it).

**The trap:** treating this as an infrastructure problem when `qa.agent.steps` moved. If the agent is
taking 40% more steps, no amount of concurrency tuning helps, and adding concurrency will make the
provider-side contention worse.

---

## 6.2 Playbook — verdict quality dropped

**Trigger:** false-pass or false-fail rate above objective, or `qa.verdict.mix_divergence` alarm.

**First 5 minutes:**

1. **Is the evaluation plane sound?** `qa.eval.judge_agreement`, `qa.eval.coverage`,
   `human_judge_agreement`. If any is off → go to §6.8. **Do not proceed.**
2. **Is it a real quality change or an input change?** Compare `qa.verdict.mix_divergence` against
   `qa.document.population_drift`. Both moving ⇒ the documents changed, and the system may be behaving
   correctly on genuinely different input. Only the first moving ⇒ the system changed.
3. **Change manifest diff.** All seven surfaces vs the last-known-good window.
4. **Slice, do not aggregate.** By `rule_category`, `severity`, `model`, `doc_type`,
   **`context_utilisation` decile** (§F2.5), `prompt_version`. Quality regressions are almost always
   concentrated in a slice; the aggregate number is the *last* place a regression becomes visible.
5. **Direction matters more than magnitude.** Which way did the error mix move? A drop in accuracy that
   is *entirely* false-fail is a very different incident from the same drop that is false-pass (§1.6).

**Ranked causes:** ① a change in one of the seven surfaces ② silent model version change (F2.1)
③ retrieval index regression (F4.4) ④ context degradation (F2.5) ⑤ document population shift
⑥ judge drift masquerading as quality drift (F8.1).

**Containment:** roll back the changed surface if identified. If not, **raise escalation** — shift the
risk to humans while you investigate. That is what the escalation lever is for; use it early rather
than as a last resort.

---

## 6.3 Playbook — convergence anomaly (steps, loops, terminations)

**Trigger:** `termination_reason` distribution shift, or `qa.agent.steps` p95 shift.

| Observation | Read as |
|---|---|
| `step_budget_exhausted` ↑ | **F3.2**; check `repeated_tool_call_ratio` for the underlying loop |
| `repeated_tool_call_ratio` ↑ | **F3.1** loop. Check whether the tool became non-deterministic or started returning empty |
| `steps` ↓ **and** `evidence_count` ↓ | 🔴 **F3.3 premature termination.** Highest-priority reading in this table |
| `insufficient_evidence` ↑ | Retrieval or corpus problem (**F4.3**) — usually *not* a model problem |
| `insufficient_evidence` ↓ **and** `evidence.verified_ratio` ↓ | 🔴 **F3.6 manufactured certainty** |
| `context_overflow` ↑ | **F2.4**; check document size distribution first |
| `steps` histogram bimodal | **F3.8**; find the splitting dimension before drawing any conclusion |

**Why start here in most quality incidents:** these signals are deterministic, free, available on 100%
of invocations, and they move **before** sampled quality metrics can reach significance (§4.4.2). In
practice, convergence anomalies are your fastest true-positive quality alarm.

---

## 6.4 Playbook — grounding failure

**Trigger:** `qa.evidence.fabricated` > 0, or `evidence.verified_ratio` below objective.

1. **Fabricated > 0 is a P2 incident.** The system asserted evidence that does not exist, in a
   compliance context. Establish scope immediately: how many verdicts, which rules, which document
   versions, which have been released downstream.
2. Separate the three grounding failures — they have different fixes:
   - **Fabricated** (F4.1): text does not exist anywhere → model failure
   - **Stale version** (F4.2): text exists in another version → **version propagation bug**, a control-plane defect
   - **Mismatch** (F4.5): text exists and verifies but does not support the verdict → reasoning failure
3. Check `retrieval.top_score` distribution and `retrieval.index_version` before blaming the model.
   An index rebuild (F4.4) is a far more common cause of a broad grounding drop than model behaviour.
4. Containment: force `escalate` for all affected rule categories; quarantine affected verdicts;
   re-run with the previous index if the manifest points there.

---

## 6.5 Playbook — cost anomaly

**Trigger:** `qa.cost.per_verdict` or `per_document` above budget.

**Look at the p99 before the mean.** Cost anomalies in agentic systems are usually a small population
of runaway invocations, not a broad shift.

| Signal | Cause | Section |
|---|---|---|
| p99 ≫ mean, `steps` right tail heavy | Loop-driven blowout | F6.2 / F3.1 |
| `cache.hit_ratio` cliff | Cache collapse — check for a recent prompt or tool-schema change | F6.1 |
| `retry.attempts` mean up | Retry-attributed spend during degradation | F6.3 |
| `tokens.reasoning_ratio` up | Reasoning-token blowout | F6.4 |
| model mix shifted | Tier drift, often via a fallback path firing more | F6.5 |
| Cost flat per verdict, total up | **Not an incident** — volume grew. Check `qa.rule.dispatched` | — |

**The discipline that makes this playbook work:** cost per *verdict*, not cost per day. Absolute spend
conflates volume with efficiency and will send you chasing a growth curve.

---

## 6.6 Playbook — escalation rate anomaly

**Both directions are incidents. Neither is self-evidently good news.**

**Escalation ↑ (flood):** check `guardrail_blocked` rate (F5.3 over-trigger), confidence-distribution
shift (calibration decay, F8.6), input population drift, threshold change in the manifest.
Contain by prioritising on `rule.severity` and applying admission control at intake — **not** by
loosening thresholds, which converts a visible problem into an invisible one.

**Escalation ↓ (collapse):** this is the dangerous direction. Check the false-pass rate — if the CI is
too wide to tell, **that itself is the finding**. Check for a threshold change, calibration drift, and
the F7.1 rubber-stamp signature. Contain by restoring the previous threshold profile and raising the
human-review sample rate on PASS verdicts until you have precision on the false-pass estimate.

> The failure of intuition here is worth naming explicitly: a falling escalation rate presents to
> stakeholders as **the automation maturing**. It is equally consistent with the automation quietly
> becoming over-confident. The two-sided SLO in §4.3 exists so that the system, not a stakeholder's
> intuition, decides which one it is.

---

## 6.7 Playbook — provider degradation

**Trigger:** ITL rising, error rate rising, or a provider status advisory.

1. **Classify before you react** (§9.4): transient (429) / systemic (529, 5xx) / terminal (deterministic 4xx).
   The response differs completely, and the most common incident-amplifying mistake is retrying a
   systemic failure as if it were transient.
2. Confirm it is the provider, not you: is it correlated across all tenants and rule categories?
   Is `time_per_output_chunk` up while your own queue wait is flat?
3. **Check your own retry behaviour before escalating to the vendor.** §1.3.2: at a 60% failure rate,
   3-attempt retries nearly double your load on the degraded provider.
4. Contain: honour `retry-after`; trip the per-provider circuit breaker; fail over if a secondary is
   qualified for this workload (**qualified** means it has passed the eval suite — an unqualified
   failover trades an availability incident for a quality incident, which is a bad trade in a
   compliance system); otherwise degrade to defensive escalation.
5. Record `gen_ai.response.id` samples for the vendor support case.

---

## 6.8 Playbook — the evaluation plane is broken

**Trigger:** judge agreement below 0.8, eval coverage falling, or eval scores moving with an unchanged
change manifest.

**Declare that quality metrics are untrusted, and say so on the dashboard.** A quality number from a
broken evaluator is worse than no number, because it will be acted on.

1. Run the **frozen golden set** through the current judge. If scores differ from the recorded baseline
   with no judge-version change → **F8.1 judge drift**, almost always a provider-side model change.
   Pin the snapshot.
2. Run the judge N=5 times on frozen inputs. Agreement < 0.8 → **F8.2**. Fix the rubric before touching
   anything else; vague rubrics are the majority cause.
3. Check `qa.eval.coverage`. If it fell, the rate did not change — the **denominator** did (**F8.5**).
4. Check `human_judge_agreement`. If the judge has drifted from human ground truth, every historical
   quality number needs re-basing, and you must say so rather than quietly re-baselining.
5. While untrusted: fall back to deterministic signals (`evidence.verified_ratio`, `fabricated`,
   termination distribution, mix divergence) and raise the human-review sample rate.

---

# 7. Change safety

## 7.1 Seven change surfaces

In a conventional service, "a change" means a code deploy, and your change-management process is built
around that. In an agentic system **seven independent surfaces alter behaviour**, and most of them do
not look like deploys — which is precisely why they cause incidents that nobody can attribute.

| # | Surface | Changes without a code deploy? | Blast radius | Detected by |
|---|---|---|---|---|
| 1 | **Prompt / instructions** | ✅ often config | Global, immediate | `qa.change.prompt_version`; mix divergence |
| 2 | **Model version** | ✅ **provider-side, without telling you** | Global, immediate | `qa.model.response_model_mismatch`; frozen canary set |
| 3 | **Tool schema / definitions** | ✅ | Global; also **destroys cache** | `tool_schema_hash`; `tool_arg_invalid`; cache hit ratio |
| 4 | **Rule set / checklist** | ✅ business-owned | Scoped to categories | `rule_set_version`; coverage; per-category quality |
| 5 | **Retrieval index / corpus** | ✅ scheduled rebuilds | Global, delayed onset | `retrieval_index_version`; `top_score` distribution |
| 6 | **Judge / evaluator** | ✅ | **Measurement only** — but corrupts every decision built on it | `judge_version`; golden-set drift |
| 7 | **Thresholds / routing config** | ✅ runtime | Escalation mix, human load | `threshold_profile`; escalation rate |

**All seven MUST be in the change manifest on every run (§2.3), and all seven MUST go through the same
release gate.** A rule-set edit made in a business console at 4pm on a Friday is a production change to
a regulated system. Treat it like one — version it, gate it, and make it visible in the same place
engineers look when they ask "did anything change?"

## 7.2 The release gate

Every change to any of the seven surfaces passes the same gate.

```
┌── STATIC ────────────────────────────────────────────────────────────────┐
│ · Schema validation of prompt/rule/tool artifacts                        │
│ · Tool-definition serialisation is byte-stable (cache protection, F6.1)  │
│ · No timestamp/nonce/UUID in the cacheable prefix                        │
│ · Change manifest complete — all seven surfaces present and versioned    │
└──────────────────────────────────────────────────────────────────────────┘
┌── DETERMINISTIC EVAL (fast, free, blocking) ─────────────────────────────┐
│ · Format/schema conformance = 100%                                       │
│ · Citation-exists on the golden set = 100%                               │
│ · Step budget respected on all cases                                     │
│ · Required tools called on cases where they are required                 │
│  ⟶ Any failure BLOCKS. No judgement calls at this layer.                 │
└──────────────────────────────────────────────────────────────────────────┘
┌── JUDGE EVAL (slower, sampled, statistically qualified) ─────────────────┐
│ · Judge stability check first: N=5, agreement ≥ 0.85, else UNSTABLE=FAIL  │
│ · Quality vs baseline on the golden set                                  │
│ · Gate on the UPPER confidence bound (§4.4.5)                            │
│ · Report the MINIMUM DETECTABLE EFFECT in the gate output                │
└──────────────────────────────────────────────────────────────────────────┘
┌── ADVERSARIAL SET (blocking, asymmetric) ────────────────────────────────┐
│ · Known-non-compliant documents that MUST NOT pass                       │
│ · Buried-evidence cases that MUST NOT terminate early                    │
│ · Prompt-injection cases embedded in document text                       │
│  ⟶ FALSE PASS ON THIS SET IS AN ABSOLUTE BLOCK. No override, no waiver.  │
└──────────────────────────────────────────────────────────────────────────┘
┌── ECONOMIC (blocking with a documented waiver path) ─────────────────────┐
│ · Cost per verdict within ±15% of baseline, or explicitly approved       │
│ · Cache hit ratio projected to recover within one deploy cycle           │
│ · Steps per verdict within band (a cost win via fewer steps may be F3.3) │
└──────────────────────────────────────────────────────────────────────────┘
```

**The adversarial set is the gate that matters.** Aggregate quality can improve while false-pass gets
worse (§1.6). A dedicated set of documents that **must not pass** is the only gate that directly
defends the asymmetry, and it is the one to build first if you build only one.

## 7.3 Eval-suite hygiene

- **Golden set** — curated, stable, never used for prompt iteration. Rotate a portion quarterly.
- **Regression set** — every production failure ever found, converted into a permanent case. This set
  only grows. It is your institutional memory and it is the single highest-ROI artifact in the pipeline.
- **Adversarial set** — deliberately hard: buried evidence, near-miss compliance, contradictory clauses,
  documents that superficially resemble compliant ones, injection attempts in document text.
- **Production-sampled set** — refreshed continuously from real traffic, including every human override
  (F7.5 gives you these labelled for free).

**Declare and publish the minimum detectable effect of each set** (§4.4.2). An eval suite whose MDE
nobody knows produces green checkmarks of unknown meaning.

**On flakiness:** measure and publish per-case judge agreement alongside the pass rate. `UNSTABLE`
counts as a failure. A suite that reports "97% pass" while a fifth of its cases flip between runs is
reporting a random variable.

## 7.4 Progressive delivery

| Stage | Traffic | Duration | Gate to advance |
|---|---|---|---|
| **Shadow** | 0% (runs in parallel, output discarded) | 24–48 h | No error-rate delta; cost within band; **verdict agreement with production ≥ threshold** |
| **Canary** | 1–5% | 24–72 h | Behavioural metrics within band; no escalation-rate shift; quality CI overlapping baseline |
| **Ramp** | 25% → 50% → 100% | 24 h per step | Same, plus enough accumulated sample for the quality comparison to have useful power |
| **Bake** | 100% | 7–14 d | Quality SLI stable; `qa.review.reopened` (the ungamed metric) not rising |

**Shadow mode is disproportionately valuable here and under-used.** Because the agent's output is not
acted on, you can run the new configuration against **real production documents** and compare verdicts
directly against the incumbent — the highest-fidelity signal available short of shipping, at zero
correctness risk. The only cost is tokens. In a compliance system, that is a very cheap insurance
premium.

**Ramp slowly, and know why:** quality signals need sample to reach significance (§4.4.2). A canary at
1% for 24 hours may accumulate too few reviewed verdicts to detect anything. Compute the accumulated
sample size and state it in the promotion decision, rather than promoting on elapsed time.

## 7.5 Rollback

**Every surface MUST be independently rollable, in minutes, without a code deploy.**
This is an architectural requirement, not an operational aspiration:

- Prompts: versioned artifacts, selected at runtime
- Models: pinned snapshots, switchable by config
- Tool schemas: versioned, with the previous version retained
- Rule sets: versioned, with the previous version retained
- Retrieval indexes: previous index kept warm and switchable
- Judges: pinned and versioned
- Thresholds: runtime config

**Rollback triggers — pre-agree these, do not deliberate during an incident:**

| Trigger | Action |
|---|---|
| `qa.evidence.fabricated` above baseline | 🔴 Immediate rollback |
| False-pass rate lower CI bound above objective | 🔴 Immediate rollback |
| Adversarial set regression detected post-deploy | 🔴 Immediate rollback |
| Escalation rate outside band by >2× | Rollback or defensive profile |
| Cost per verdict > 2× baseline | Rollback unless explicitly waived |
| Behavioural metrics outside band with quality inconclusive | Rollback — **absence of evidence of harm is not evidence of absence**, and at realistic sample sizes it usually just means you cannot see yet |

**Post-rollback, before re-attempting:** add the failing case to the regression set. A rollback that
does not produce a permanent test case will be repeated.

---

# 8. Evidence, audit and reproducibility

## 8.1 The trace is the audit record

In this system the distinction between "observability data" and "the compliance record" is thin, and
pretending otherwise creates risk in both directions: retention that is too short to satisfy an
auditor, or sensitive content sprayed into a general-access observability backend.

**Resolve it explicitly** (§2.5, §2.6): T0 audit spans are the record and are never sampled;
content lives in the governed evidence store and is referenced, never inlined.

**The questions an auditor or regulator will ask, and the span that answers each:**

| Question | Answered by |
|---|---|
| What decision did the system make about this document? | `qa.verdict` span — value, confidence, evidence refs |
| On what evidence? | `qa.verdict.evidence_*` + evidence store via `qa.content.ref` |
| Was the evidence real? | `qa.evidence.verified_ratio` on that verdict — a recorded deterministic check |
| Which document version? | `qa.document.version` |
| Which rule, in which version? | `qa.rule.id` + `qa.rule.version` |
| What system configuration produced it? | `qa.change.*` manifest — all seven surfaces |
| Did a human review it? | Linked `qa.review` trace — reviewer role, dwell, decision, override reason |
| Why was it or was it not escalated? | `qa.verdict.escalated` + `escalation_reason` + `threshold_profile` |
| How do you know your human oversight is real? | `qa.review.probe.catch_rate` over the period |
| How do you know the system is performing as claimed? | SLO attainment history + eval results per release |

**Note what the last two do for you.** "We have a documented human review procedure" is a policy answer.
"Our human review catches 84% of injected known-bad cases, measured continuously, here is the 90-day
series" is an **evidenced control**. The second is materially stronger in front of an auditor, and it
costs one metric.

## 8.2 Regulatory framing

The EU AI Act's record-keeping provisions are the clearest articulation of what is expected, and they
are worth designing to whether or not they bind you today:

- **Article 12 (Record-keeping)** requires high-risk AI systems to technically allow **automatic
  recording of events (logs) over the lifetime of the system**, sufficient to identify risk situations,
  support post-market monitoring, and enable operational oversight.
- **Article 19 (Automatically generated logs)** requires providers to retain automatically generated
  logs for a period appropriate to the intended purpose, **at least six months** unless other law
  requires longer.
- Applicability for these obligations runs from **2 December 2027** (Annex III systems) or
  **2 August 2028** (Annex I). ⚠️ Dates and interpretation continue to move — **confirm current status
  with your legal/compliance function; do not rely on this document for a regulatory date.**
- **Relevant to a financial-services context:** the Act contemplates entities already subject to EU
  financial-services governance folding these log-retention obligations into their existing internal
  documentation regime rather than standing up a parallel one. That is an integration decision worth
  making early, with compliance, rather than discovering later.

**The design implication is simple and it changes your architecture:** retention of the audit tier is
driven by regulation, not by observability cost. Six months of APM trace retention is expensive and
unnecessary; six months of a compact, structured audit record is cheap and required. **This is the
strongest single argument for the two-store split in §2.5**, and it is the argument to lead with when
someone asks why the evidence store exists separately from Splunk.

Adjacent frameworks worth aligning vocabulary with — NIST AI RMF (Measure and Manage functions map
almost directly onto §3–§4 of this document) and ISO/IEC 42001 — mostly reinforce the same
requirements: measurable performance, documented monitoring, demonstrable human oversight.

## 8.3 The replay contract

**Reproducibility is a design property. If you do not build for it, you will not have it — and in a
compliance system you will eventually be asked for it.**

To re-derive why a verdict was reached, you need:

| Input | Source | Notes |
|---|---|---|
| Document content + version | Evidence store | Immutable, addressed by version |
| Rule text + version | Rule store | Versioned |
| Prompt template + version | Prompt store | Versioned |
| Model snapshot | `qa.change.model_id` | ⚠️ **Provider snapshots are deprecated over time.** Retention of a *reproducible* model is outside your control |
| Sampling parameters | `gen_ai.request.temperature`, `top_p`, `seed` | |
| Tool schema | `tool_schema_hash` + schema store | |
| Retrieval index version | `retrieval_index_version` | ⚠️ Index rebuilds usually destroy the old index — **keep versioned snapshots or accept non-reproducibility** |
| The trajectory that occurred | T0 + T1 spans | |

**Be honest about the limit, internally and externally.** Exact replay is generally **not achievable**
in an agentic system: providers deprecate model snapshots, sampling is stochastic even at temperature 0,
and indexes are rebuilt. What *is* achievable, and what you should commit to, is:

> **Evidentiary reproducibility** — we can show exactly what the system did, on what inputs, under what
> configuration, and why the verdict followed from the evidence. We cannot guarantee that re-running the
> model today produces a byte-identical result.

Write this distinction down and agree it with compliance **before** an audit, not during one. An
organisation that has thought about the limits of reproducibility and documented them reads as
competent; one that discovers the limits under questioning does not.

**Two engineering consequences that follow directly:**

1. **The rationale and evidence are the durable artifact, not the model.** Store them richly enough that
   the verdict is defensible from the record alone, without re-running anything.
2. **Retain versioned retrieval index snapshots** for at least the audit retention period, or accept and
   document that retrieval is not reproducible. This is a real storage cost and it should be an
   explicit, funded decision rather than an accident.

---

# 9. Instrumentation reference

## 9.1 Minimum viable instrumentation

If you implement nothing else from this document, implement this list. It is ordered by
value-per-unit-effort, and the ordering is deliberate — items 1–5 are cheap, deterministic, and catch
the highest-consequence failures.

1. `qa.rule` span per rule, with `rule.id`, `version`, `category`, `severity`
2. `qa.agent.termination_reason` on every agent invocation *(the single highest-information attribute)*
3. `qa.rule.coverage_ratio` per document *(catches the silent rule drop — the failure nothing else sees)*
4. Deterministic citation verification → `qa.evidence.verified_ratio`, `qa.evidence.fabricated`
5. The seven-surface change manifest on the root span
6. `gen_ai.*` token attributes on every model call
7. `qa.verdict.value` and `escalated` with `escalation_reason`
8. `qa.review.*` on the linked review trace, with **structured** override reason codes
9. Tail-based sampling with T0 never sampled
10. Exemplars linking metrics to traces

**Items 1–5 cost days, not weeks, and they cover the failure modes that produce silent compliance
defects. Everything after item 5 improves your ability to diagnose; items 1–5 determine whether you can
detect at all.**

## 9.2 Code patterns

Illustrative Python using the OTel SDK. Adapt to your framework's instrumentation, but **the attribute
names are the contract** (§2) and must not drift.

### 9.2.1 The rule span — the critical unit

```python
from opentelemetry import trace, baggage, context
from opentelemetry.trace import Status, StatusCode

tracer = trace.get_tracer("compliance_qa")

def evaluate_rule(run_id: str, doc: Document, rule: Rule) -> Verdict:
    with tracer.start_as_current_span("qa.rule") as span:
        # Correlation keys — MUST be present on every span in the subtree.
        span.set_attribute("qa.run.id", run_id)
        span.set_attribute("qa.document.id", doc.id)
        span.set_attribute("qa.document.version", doc.version)   # non-negotiable (§2.3)
        span.set_attribute("qa.rule.id", rule.id)
        span.set_attribute("qa.rule.version", rule.version)
        span.set_attribute("qa.rule.category", rule.category)     # low cardinality → metric dimension
        span.set_attribute("qa.rule.severity", rule.severity)
        span.set_attribute("qa.rule.evidence_required", rule.evidence_required)

        # Propagate to children (including across async boundaries — see §9.2.5).
        ctx = baggage.set_baggage("qa.run.id", run_id)
        ctx = baggage.set_baggage("qa.rule.id", rule.id, context=ctx)

        try:
            verdict = _run_agent_loop(doc, rule, ctx)
        except Exception as exc:
            span.set_status(Status(StatusCode.ERROR))
            span.set_attribute("error.type", type(exc).__qualname__)
            raise

        RULE_DURATION.record(span_elapsed_s(span), {
            "rule_category": rule.category,      # bounded
            "severity": rule.severity,           # bounded
            # NOTE: rule.id deliberately absent — cardinality budget (§2.4)
        })
        return verdict
```

### 9.2.2 The agent invocation — convergence instrumentation

```python
def _run_agent_loop(doc, rule, ctx) -> Verdict:
    with tracer.start_as_current_span("invoke_agent rule_validator") as span:
        span.set_attribute("gen_ai.operation.name", "invoke_agent")
        span.set_attribute("gen_ai.agent.name", "rule_validator")
        span.set_attribute("gen_ai.conversation.id", conversation_id)
        span.set_attribute("qa.agent.step_budget", STEP_BUDGET)

        steps, seen_calls, peak_ctx = 0, set(), 0.0
        termination = "verdict_reached"
        deadline = monotonic() + RULE_DEADLINE_S

        while steps < STEP_BUDGET:
            if monotonic() > deadline:
                termination = "deadline_exceeded"
                break

            step = model_step(...)                     # emits a `chat` CLIENT span (§9.2.3)
            steps += 1
            peak_ctx = max(peak_ctx, step.context_utilisation)

            if step.context_utilisation > 0.95:
                termination = "context_overflow"
                break

            for call in step.tool_calls:
                # Loop detection — MAST FM-1.3 (§F3.1). Deterministic, cheap, effective.
                key = (call.name, canonical_json(call.args))
                if key in seen_calls:
                    REPEATED_TOOL_CALL.add(1, {"rule_category": rule.category})
                    step.inject_observation(
                        f"You already called {call.name} with these arguments. "
                        f"Prior result: {seen_calls_result[key]}"
                    )
                    continue
                seen_calls.add(key)
                execute_tool(call, ctx)               # emits an `execute_tool` span (§9.2.4)

            if step.has_verdict:
                break
        else:
            termination = "step_budget_exhausted"

        # THE attribute. One enum value, enormous diagnostic leverage (§2.3.1).
        span.set_attribute("qa.agent.termination_reason", termination)
        span.set_attribute("qa.agent.steps_used", steps)
        span.set_attribute("qa.agent.context_utilisation_peak", peak_ctx)

        AGENT_TERMINATIONS.add(1, {
            "termination_reason": termination,
            "rule_category": rule.category,
        })
        AGENT_STEPS.record(steps, {"rule_category": rule.category,
                                   "termination_reason": termination})
        return step.verdict if termination in ("verdict_reached",
                                               "insufficient_evidence") else None
```

### 9.2.3 The model call — token and version instrumentation

```python
def model_step(messages, tools) -> Step:
    with tracer.start_as_current_span(f"chat {MODEL_SNAPSHOT}",
                                      kind=trace.SpanKind.CLIENT) as span:
        span.set_attribute("gen_ai.operation.name", "chat")
        span.set_attribute("gen_ai.provider.name", PROVIDER)       # NOT gen_ai.system (§2.2)
        span.set_attribute("gen_ai.request.model", MODEL_SNAPSHOT) # pinned snapshot, never an alias
        span.set_attribute("gen_ai.request.temperature", TEMPERATURE)

        resp = client.messages.create(model=MODEL_SNAPSHOT, messages=messages, tools=tools)

        span.set_attribute("gen_ai.response.model", resp.model)
        span.set_attribute("gen_ai.response.id", resp.id)
        span.set_attribute("gen_ai.usage.input_tokens", resp.usage.input_tokens)
        span.set_attribute("gen_ai.usage.output_tokens", resp.usage.output_tokens)
        span.set_attribute("gen_ai.response.finish_reasons", [resp.stop_reason])

        # Cache accounting — drives §3.6 economics. Attribute names are provider-specific;
        # normalise to the gen_ai.* names at emission, not at query time.
        if (cr := getattr(resp.usage, "cache_read_input_tokens", None)) is not None:
            span.set_attribute("gen_ai.usage.cache_read.input_tokens", cr)
        if (cc := getattr(resp.usage, "cache_creation_input_tokens", None)) is not None:
            span.set_attribute("gen_ai.usage.cache_creation.input_tokens", cc)

        # Silent version change detector (§F2.1).
        if resp.model != MODEL_SNAPSHOT:
            MODEL_MISMATCH.add(1, {"requested": MODEL_SNAPSHOT, "served": resp.model})

        # Truncation is a correctness bug, not a performance one (§F2.6).
        if resp.stop_reason == "max_tokens":
            FINISH_REASON.add(1, {"reason": "length", "model": MODEL_SNAPSHOT})

        # Content: reference, never inline, in production (§2.5).
        if CAPTURE_CONTENT:
            span.set_attribute("qa.content.ref", evidence_store.put(messages, resp))

        return Step.from_response(resp)
```

### 9.2.4 Tools — idempotency and argument validation

```python
def execute_tool(call, ctx, step_index: int):
    with tracer.start_as_current_span(f"execute_tool {call.name}") as span:
        span.set_attribute("gen_ai.operation.name", "execute_tool")
        span.set_attribute("gen_ai.tool.name", call.name)

        if TOOLS[call.name].mutating:
            # Deterministic, replay-safe key (§F1.5).
            key = f"{baggage.get_baggage('qa.run.id', ctx)}:" \
                  f"{baggage.get_baggage('qa.rule.id', ctx)}:{step_index}"
            span.set_attribute("qa.tool.idempotency_key", key)

        # Separate "model called the tool wrong" from "the tool broke" (§F3.7).
        try:
            args = TOOLS[call.name].schema.validate(call.args)
            span.set_attribute("qa.tool.arg_validation", "valid")
        except SchemaError as e:
            span.set_attribute("qa.tool.arg_validation", "invalid_schema")
            TOOL_ARG_INVALID.add(1, {"tool_name": call.name, "reason": e.code})
            return ToolResult.invalid(str(e))

        result = TOOLS[call.name].run(args, idempotency_key=key)
        span.set_attribute("qa.tool.result_cardinality", len(result))
        return result
```

### 9.2.5 Verdict, deterministic verification, and the evaluation event

```python
def materialise_verdict(raw, doc: Document, rule: Rule) -> Verdict:
    with tracer.start_as_current_span("qa.verdict") as span:
        # DETERMINISTIC citation verification. Microseconds. Never drifts.
        # Build this before any LLM judge (§1.5, §3.3).
        verified = sum(1 for c in raw.citations
                       if doc.contains_at(c.text, c.locator, version=doc.version))
        fabricated = len(raw.citations) - verified

        span.set_attribute("qa.verdict.id", raw.verdict_id)
        span.set_attribute("qa.verdict.value", raw.value)
        span.set_attribute("qa.verdict.evidence_count", len(raw.citations))
        span.set_attribute("qa.verdict.evidence_verified", verified)
        span.set_attribute("qa.verdict.grounded",
                           verified >= 1 or not rule.evidence_required)

        if fabricated:
            EVIDENCE_FABRICATED.add(fabricated, {"rule_category": rule.category})

        # Guardrails FAIL CLOSED — escalate, never pass (§F5.4).
        if rule.evidence_required and verified == 0:
            raw = raw.override(value="escalate", reason="ungrounded")

        span.set_attribute("qa.verdict.escalated", raw.value == "escalate")
        if raw.value == "escalate":
            span.set_attribute("qa.verdict.escalation_reason", raw.escalation_reason)

        # Evaluation results ride the OTel event, one per evaluator (§2.3 / §3.4).
        for ev in run_evaluators(raw, doc, rule):
            span.add_event("gen_ai.evaluation.result", {
                "gen_ai.evaluation.name": ev.name,          # required by the convention
                "gen_ai.evaluation.score.value": ev.score,
                "gen_ai.evaluation.score.label": ev.label,
                "gen_ai.evaluation.explanation": ev.explanation,
                "gen_ai.response.id": raw.response_id,
                "qa.eval.judge_version": ev.judge_version,  # ours — needed for §F8.1
            })
        return raw
```

### 9.2.6 Linking the asynchronous human review

```python
def record_review(verdict_id: str, verdict_span_context, decision, dwell_ms, reviewer):
    link = trace.Link(verdict_span_context)          # link, NOT a parent (§2.1)
    with tracer.start_as_current_span("qa.review", links=[link]) as span:
        span.set_attribute("qa.verdict.id", verdict_id)
        span.set_attribute("qa.review.reviewer_role", reviewer.role)  # role, not identity
        span.set_attribute("qa.review.dwell_ms", dwell_ms)
        span.set_attribute("qa.review.decision", decision.value)
        span.set_attribute("qa.review.override", decision.overrides_agent)
        if decision.overrides_agent:
            # STRUCTURED reason codes. Free labelled training data (§F7.5).
            span.set_attribute("qa.review.override_reason_code", decision.reason_code)
        REVIEW_DWELL.record(dwell_ms / 1000, {"reviewer_role": reviewer.role,
                                              "verdict": decision.original_verdict})
```

## 9.3 Collector pipeline

```yaml
receivers:
  otlp: { protocols: { grpc: {}, http: {} } }

processors:
  # 1. Normalise convention drift in ONE place (§2.2).
  transform/genai_normalise:
    trace_statements:
      - context: span
        statements:
          - set(attributes["gen_ai.provider.name"], attributes["gen_ai.system"])
              where attributes["gen_ai.provider.name"] == nil
                and attributes["gen_ai.system"] != nil
          - set(attributes["gen_ai.usage.input_tokens"], attributes["gen_ai.usage.prompt_tokens"])
              where attributes["gen_ai.usage.input_tokens"] == nil
                and attributes["gen_ai.usage.prompt_tokens"] != nil

  # 2. Redact BEFORE anything leaves. Allow-list, never deny-list (§2.5).
  redaction:
    allow_all_keys: false
    allowed_keys: [service.name, service.version, deployment.environment.name,
                   qa.run.id, qa.document.id, qa.document.version, qa.rule.id,
                   qa.rule.category, qa.rule.severity, qa.verdict.value,
                   qa.agent.termination_reason, qa.content.ref,
                   gen_ai.operation.name, gen_ai.provider.name, gen_ai.request.model,
                   gen_ai.response.model, gen_ai.usage.input_tokens,
                   gen_ai.usage.output_tokens]
    blocked_values: ['(?i)\b[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}\b']

  # 3. Tail sampling — T0 always, T1 on interest, T2 low (§2.6).
  tail_sampling:
    decision_wait: 60s              # long agent traces need a long window
    num_traces: 200000              # SIZE THIS. Under-sizing silently drops traces.
    policies:
      - name: t0_audit_always
        type: string_attribute
        string_attribute: { key: qa.span.tier, values: ["t0"] }
      - name: errors_always
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: escalated_always
        type: boolean_attribute
        boolean_attribute: { key: qa.verdict.escalated, value: true }
      - name: slow_always
        type: latency
        latency: { threshold_ms: 30000 }
      - name: baseline_sample
        type: probabilistic
        probabilistic: { sampling_percentage: 5 }

  batch: { timeout: 5s, send_batch_size: 1024 }

exporters:
  signalfx:          { send_otlp_histograms: true }   # required for Splunk AI Agent Monitoring
  otlphttp/dynatrace: { endpoint: "${DT_ENDPOINT}/api/v2/otlp" }
  otlphttp/llm_store: { endpoint: "${LLM_STORE_OTLP}" }

service:
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [transform/genai_normalise, redaction, tail_sampling, batch]
      exporters:  [signalfx, otlphttp/dynatrace, otlphttp/llm_store]
    metrics:
      receivers:  [otlp]
      processors: [transform/genai_normalise, batch]
      exporters:  [signalfx, otlphttp/dynatrace]
```

**Operate the collector as a tier-1 dependency.** With tail sampling on the compliance path, collector
failure is audit-record loss (§F1.7). Monitor queue depth, refused spans and dropped traces, alert on
them, and size `num_traces` against measured agent trace volume — not against a default.

## 9.4 Failure-class routing

Stock circuit breakers assume failures are transient, independent, cheap to retry, and idempotent.
LLM calls violate all four: a 429 or 529 hits your whole fleet at once, every retry re-bills tokens,
and a partially streamed generation has already cost you money. Classify before you react.

```python
class FailureClass(Enum):
    TRANSIENT = "transient"   # 429, connection reset → honour retry-after, retry
    SYSTEMIC  = "systemic"    # 529, 5xx           → circuit break + full jitter + failover
    TERMINAL  = "terminal"    # deterministic 4xx  → NEVER retry; fix or degrade

def classify(exc) -> FailureClass:
    if exc.status == 429 or isinstance(exc, ConnectionResetError):
        return FailureClass.TRANSIENT
    if exc.status in (529,) or 500 <= exc.status < 600:
        return FailureClass.SYSTEMIC
    return FailureClass.TERMINAL

def call_with_policy(fn, deadline_s: float, provider: str):
    """DEADLINE-bounded, not attempt-bounded. Attempt counts ignore how much time
    the caller actually has, which is the number that matters (§F1.4)."""
    end, sunk_tokens = monotonic() + deadline_s, 0
    while monotonic() < end:
        if breaker[provider].is_open():
            raise ProviderUnavailable(provider)
        try:
            return fn()
        except Exception as exc:
            cls = classify(exc)
            sunk_tokens += getattr(exc, "tokens_consumed", 0)   # you were billed for these
            TOOL_ERRORS.add(1, {"error_class": cls.value, "provider": provider})
            if cls is FailureClass.TERMINAL:
                raise
            if cls is FailureClass.SYSTEMIC:
                breaker[provider].record_failure()
            sleep(min(full_jitter_backoff(), end - monotonic()))
    raise DeadlineExceeded(sunk_tokens=sunk_tokens)
```

**Per-provider breakers, never a global one.** A global breaker converts one provider's degradation into
a total outage. **Full jitter, never plain exponential** — synchronised retries across a fleet turn a
short provider blip into a long one.

## 9.5 The telemetry contract test

This is the test that catches convention drift on a dependency bump (§2.2). It is cheap and it is the
only thing standing between an instrumentation-library upgrade and a silently broken dashboard.

```python
def test_telemetry_contract(in_memory_exporter):
    run_fixture_document(rules=[FIXTURE_RULE])
    spans = {s.name: s for s in in_memory_exporter.get_finished_spans()}

    rule = spans["qa.rule"]
    for attr in ("qa.run.id", "qa.document.id", "qa.document.version",
                 "qa.rule.id", "qa.rule.version", "qa.rule.category", "qa.rule.severity"):
        assert attr in rule.attributes, f"contract violation: {attr} missing (§2.3)"

    agent = next(s for s in spans.values() if s.name.startswith("invoke_agent"))
    assert agent.attributes["qa.agent.termination_reason"] in TERMINATION_REASONS

    chat = next(s for s in spans.values() if s.name.startswith("chat"))
    assert "gen_ai.provider.name" in chat.attributes          # NOT gen_ai.system
    assert "gen_ai.usage.input_tokens" in chat.attributes     # NOT prompt_tokens

    # No document content escaped into span attributes (§2.5).
    for s in spans.values():
        for v in s.attributes.values():
            assert FIXTURE_SECRET not in str(v), "content leaked into telemetry"
```

## 9.6 Anti-patterns

| Anti-pattern | Why it hurts | Instead |
|---|---|---|
| One span for the whole agent invocation | The trajectory — the actual diagnostic content — is invisible | Span per step, per tool call, per guardrail |
| `qa.rule.id` as a metric dimension | Cardinality explosion (§2.4) | `rule_category` in metrics; `rule.id` in traces; join with exemplars |
| Prompts inlined as span attributes in prod | Sensitive content in a general-access backend; attribute-limit truncation | Externalise, reference (§2.5) |
| Head sampling on the verdict path | Audit-record gaps | T0 unsampled, tail sampling elsewhere |
| Alerting on mean latency | Heavy-tailed; the mean is meaningless (§1.3.1) | p95/p99, and the p99/p95 ratio |
| Alerting on aggregate accuracy | Hides the error asymmetry (§1.6) | Separate false-pass and false-fail objectives |
| A quality number without a CI | Unfalsifiable; invites acting on noise | Publish n and the Wilson bounds (§4.4.1) |
| LLM judge for a deterministic check | Costs money, drifts, weaker signal | Assertion (§1.5) |
| Guardrails failing open | The control vanishes exactly when needed | Fail closed → escalate (§F5.4) |
| Retry by attempt count | Ignores the caller's actual deadline | Deadline-bounded with full jitter (§9.4) |
| Global circuit breaker | One provider's blip = total outage | Per-provider breakers |
| Raising the step budget to fix budget exhaustion | Converts a fast failure into a slow expensive one | Diagnose the loop (§F3.1) |
| Only reviewing escalated verdicts | Blind to false passes — the errors that matter | Over-sample PASS verdicts (§1.6) |
| Floating model aliases | Silent version change (§F2.1) | Pinned snapshots in the manifest |
| Treating a threshold change as "not a deploy" | Ungated production change to a regulated system | All seven surfaces through the gate (§7.1) |

---

# 10. Rules for the IDE assistant

**This section is normative and is written to be applied directly.** When suggesting, generating or
reviewing code in this repository, apply these rules. Cite the section number when you flag a violation
so the engineer can read the reasoning rather than just take the instruction.

## 10.1 MUST

| # | Rule | § |
|---|---|---|
| M1 | Every model call MUST be wrapped in a span with `gen_ai.operation.name`, `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.response.model`, and both token counts | 2.3 |
| M2 | Every agent invocation MUST set `qa.agent.termination_reason` from the closed enum, on **every** exit path including exceptions | 2.3.1 |
| M3 | Every span in the request path MUST carry `qa.run.id`, `qa.document.id`, `qa.document.version`, `qa.rule.id` — propagated via baggage, including across async and thread-pool boundaries | 2.3 |
| M4 | The root span MUST carry all seven change-manifest attributes | 2.3 / 7.1 |
| M5 | Model identifiers MUST be pinned snapshots. Floating aliases are prohibited | F2.1 |
| M6 | Citations MUST be verified **deterministically** against the pinned `qa.document.version` before a verdict is accepted | 3.3 |
| M7 | Guardrails MUST fail closed — on guardrail error, the verdict routes to `escalate`, never to `pass` | F5.4 |
| M8 | Mutating tool calls MUST carry an idempotency key derived from `(run_id, rule_id, step_index)` | F1.5 |
| M9 | Retries MUST be deadline-bounded with full jitter, and MUST classify the failure (transient / systemic / terminal) before retrying. Terminal failures MUST NOT be retried | 9.4 |
| M10 | Circuit breakers MUST be per-provider | 9.4 |
| M11 | Every automated evaluator MUST emit a `gen_ai.evaluation.result` event with `gen_ai.evaluation.name` and the judge version | 3.4 |
| M12 | Any new dimensioned metric MUST include a series-count estimate in the PR description | 2.4 |
| M13 | Document content, prompts and rationales MUST NOT be set as span attributes in production; use `qa.content.ref` | 2.5 |
| M14 | Aggregation MUST assert `completed_rule_count == expected_rule_count` and fail the document on mismatch | F1.1 |
| M15 | Any published quality rate MUST be accompanied by its sample size and confidence interval | 4.4.1 |
| M16 | `insufficient_evidence` MUST be an available verdict wherever evidence can be absent | 1.6 / F3.6 |
| M17 | Tool definitions MUST serialise byte-stably; a canonical-serialisation test is required | F6.1 |
| M18 | A change to any of the seven surfaces MUST pass the release gate, including threshold and rule-set changes | 7.1 |

## 10.2 SHOULD

| # | Rule | § |
|---|---|---|
| S1 | Emit cache read/write token counts wherever the provider exposes them | 3.6 |
| S2 | Emit `qa.agent.context_utilisation`; slice quality metrics by its decile | F2.5 |
| S3 | Detect repeated identical tool calls in the harness and inject the prior result rather than re-executing | F3.1 |
| S4 | Prefer a deterministic assertion over an LLM judge wherever an assertion can express the check | 1.5 |
| S5 | Use structured `reason_code` enums for human overrides, never free text alone | F7.5 |
| S6 | Dimension `qa.cache.hit_ratio` by `prompt_version` so post-deploy recovery is visible | F6.1 |
| S7 | Assert `finish_reason != "length"` before parsing a model response | F2.6 |
| S8 | Attach exemplars to metrics so a chart links to the trace that produced the datapoint | 2.4 |
| S9 | Record `gen_ai.response.id` for provider support escalations | 2.3 |
| S10 | Give every guardrail its own span, even when it is a fast local check | 2.1 |

## 10.3 NEVER

| # | Rule | § |
|---|---|---|
| N1 | NEVER head-sample the verdict path | 2.6 |
| N2 | NEVER use `qa.run.id`, `qa.document.id` or `qa.verdict.id` as a metric dimension | 2.4 |
| N3 | NEVER alert on aggregate accuracy without the directional false-pass and false-fail rates | 1.6 |
| N4 | NEVER alert on mean latency for model or agent operations | 1.3.1 |
| N5 | NEVER retry a terminal (deterministic 4xx) failure | 9.4 |
| N6 | NEVER let a guardrail fail open | F5.4 |
| N7 | NEVER emit `qa.verdict.confidence` without also publishing calibration error | 3.4.2 |
| N8 | NEVER raise a step budget as the fix for budget exhaustion without diagnosing the loop | F3.2 |
| N9 | NEVER use `gen_ai.system` or `gen_ai.usage.prompt_tokens` — those names are superseded | 2.2 |
| N10 | NEVER build an SLO on a `gen_ai.*` attribute alone; SLOs read `qa.*` | 2.2 |
| N11 | NEVER ship a cost optimisation without re-measuring `qa.evidence.count` and the false-pass rate | F3.3 |
| N12 | NEVER treat an unstable judge (self-agreement < 0.8) result as a measurement | 4.4.4 |

## 10.4 Diff review checklist

When reviewing a change in this repository, work through these in order:

**Telemetry contract**

- [ ] New spans carry the required correlation keys, including across async boundaries
- [ ] New attributes are in `qa.*` (ours) or a current `gen_ai.*` name — no superseded names
- [ ] No unbounded dimension added to a metric; series-count estimate present for new dimensions
- [ ] No content in span attributes on a production path
- [ ] All exit paths — including exception paths — set `termination_reason`

**Reliability**

- [ ] Retries are deadline-bounded, jittered, and failure-classified
- [ ] Mutating tools carry an idempotency key
- [ ] Circuit breakers are per-provider
- [ ] Timeouts compose: the sum of child deadlines does not exceed the parent's

**Correctness**

- [ ] Citations verified deterministically against the pinned document version
- [ ] Guardrails fail closed
- [ ] `insufficient_evidence` reachable wherever evidence can be absent
- [ ] Verdict schema validated before persistence; `finish_reason` checked before parsing

**Change safety**

- [ ] Does this change any of the seven surfaces? If yes: versioned, in the manifest, gated
- [ ] Does this alter the cacheable prefix or tool serialisation? If yes: cache-impact note required
- [ ] Does this reduce steps, tokens or model tier? If yes: **quality re-measurement required** — this
      is a quality change wearing a cost disguise
- [ ] Rollback path exists and does not require a code deploy

**Statistics**

- [ ] Quality claims carry sample size and confidence interval
- [ ] Eval gates report their minimum detectable effect
- [ ] Judge stability checked before judge output is used in a decision

## 10.5 Suggestion priors

When there is a judgement call and the code does not settle it, prefer in this order:

1. **Deterministic over probabilistic.** An assertion beats a judge. Always.
2. **Fail closed over fail open.** In a compliance system, uncertainty routes to a human.
3. **Explicit over implicit.** A `termination_reason` enum beats inferring intent from a null.
4. **Bounded over unbounded.** Every loop has a step budget; every call has a deadline; every queue has
   a depth limit.
5. **Observable over efficient.** A guardrail that is a span costs a few microseconds and buys a signal
   you cannot otherwise get.
6. **Versioned over current.** Anything that can change behaviour gets a version in the manifest.
7. **Escalate over guess.** `insufficient_evidence` is always a better answer than a manufactured one.

---

# 11. Maintenance contract

This document is read by an assistant that will treat it as authority. **A stale line here becomes
wrong code, silently, at scale.** That makes maintenance a correctness requirement rather than
housekeeping.

## 11.1 Review triggers

Review and update this document when **any** of these occur — not on a calendar alone:

- The OTel GenAI semantic conventions change status, rename an attribute, or publish a tagged release
  (they are in Development and have already renamed core attributes at least three times — §2.2)
- Any of the seven change surfaces changes structurally (a new tool class, a new agent, a new rule category)
- A backend changes its ingestion requirements or supported conventions
- An incident reveals a failure mode not in §5 — **add it, with its discriminator**
- An SLO target is changed
- The regulatory position changes

## 11.2 Ownership

| Section | Owner |
|---|---|
| §1 mental model, §5 failure catalogue, §6 playbooks | SRE |
| §2 telemetry contract, §9 instrumentation | SRE + application engineering |
| §3 metrics, §4 SLOs | SRE + compliance product owner (quality SLOs are **jointly** owned) |
| §7 change safety | SRE + release management |
| §8 audit | Compliance + SRE |
| §10 assistant rules | SRE, updated whenever §2–§9 change |

## 11.3 Keeping it honest

- **Every failure mode added to §5 MUST include a discriminator.** A failure mode without one adds
  noise to triage rather than signal.
- **Every number added MUST carry a provenance tier** (§0.3). Unattributed constants are how a document
  like this rots into folklore.
- **When an SLO target changes, record why in the commit message.** Targets that drift without recorded
  reasoning become unchallengeable and eventually meaningless.
- **Delete what is wrong immediately.** A known-wrong line left in place because "we'll fix it in the
  next review" will be read by an assistant hundreds of times before that review.
- **§10 is derived from §2–§9.** If you change a rule in §10 without a corresponding change upstream,
  you have created a contradiction that an assistant will resolve unpredictably.

---

# Appendix A — Attribute registry

**Ownership legend:** `OTEL` = OpenTelemetry GenAI convention (Development status — see §2.2 for the
pinning and coalescing rules); `QA` = ours, stable by our own decree.

| Attribute | Own | Type | Span | Req |
|---|---|---|---|---|
| `service.name` / `service.version` / `deployment.environment.name` | OTEL | string | all | MUST |
| `qa.pipeline.version` | QA | string | all | MUST |
| `qa.run.id` | QA | string | all | MUST |
| `qa.document.id` | QA | string | doc subtree | MUST |
| `qa.document.version` | QA | string | doc subtree | MUST |
| `qa.rule.id` / `.version` / `.category` / `.severity` | QA | string | rule subtree | MUST |
| `qa.rule.evidence_required` | QA | bool | `qa.rule` | SHOULD |
| `qa.verdict.id` | QA | string | `qa.verdict`, `qa.review` | MUST |
| `qa.tenant.id` | QA | string | all | SHOULD |
| `qa.change.prompt_version` | QA | string | root | MUST |
| `qa.change.model_id` | QA | string | root | MUST |
| `qa.change.tool_schema_hash` | QA | string | root | MUST |
| `qa.change.rule_set_version` | QA | string | root | MUST |
| `qa.change.retrieval_index_version` | QA | string | root | MUST |
| `qa.change.judge_version` | QA | string | root | MUST |
| `qa.change.threshold_profile` | QA | string | root | MUST |
| `qa.span.tier` | QA | string | all | MUST (drives sampling) |
| `qa.content.ref` | QA | string | content-bearing | COND |
| `gen_ai.operation.name` | OTEL | string | agent/model/tool | MUST |
| `gen_ai.provider.name` | OTEL | string | model | MUST |
| `gen_ai.request.model` / `gen_ai.response.model` | OTEL | string | model | MUST |
| `gen_ai.usage.input_tokens` / `.output_tokens` | OTEL | int | model | MUST |
| `gen_ai.usage.cache_read.input_tokens` / `.cache_creation.input_tokens` | OTEL | int | model | SHOULD |
| `gen_ai.usage.reasoning.output_tokens` | OTEL | int | model | SHOULD |
| `gen_ai.response.finish_reasons` | OTEL | string[] | model | MUST |
| `gen_ai.response.id` | OTEL | string | model | SHOULD |
| `gen_ai.request.temperature` / `.top_p` / `.seed` | OTEL | number | model | SHOULD |
| `gen_ai.agent.name` | OTEL | string | `invoke_agent` | MUST |
| `gen_ai.conversation.id` | OTEL | string | `invoke_agent` | SHOULD |
| `gen_ai.tool.name` / `.type` | OTEL | string | `execute_tool` | MUST / SHOULD |
| `qa.agent.step_budget` / `.steps_used` | QA | int | `invoke_agent` | MUST |
| `qa.agent.termination_reason` | QA | enum | `invoke_agent` | MUST |
| `qa.agent.context_utilisation_peak` | QA | double | `invoke_agent` | SHOULD |
| `qa.tool.idempotency_key` | QA | string | `execute_tool` (mutating) | MUST |
| `qa.tool.arg_validation` | QA | enum | `execute_tool` | MUST |
| `qa.tool.result_cardinality` | QA | int | `execute_tool` | SHOULD |
| `qa.retrieval.query_type` / `.k` / `.returned` / `.top_score` / `.index_version` | QA | mixed | retrieval | MUST for `.returned`, `.index_version` |
| `qa.verdict.value` / `.evidence_count` / `.evidence_verified` / `.grounded` / `.escalated` | QA | mixed | `qa.verdict` | MUST |
| `qa.verdict.confidence` | QA | double | `qa.verdict` | SHOULD (only if calibrated) |
| `qa.verdict.escalation_reason` | QA | enum | `qa.verdict` | COND |
| `qa.review.reviewer_role` / `.dwell_ms` / `.decision` / `.override` | QA | mixed | `qa.review` | MUST |
| `qa.review.override_reason_code` | QA | enum | `qa.review` | COND |
| `error.type` | OTEL | string | any errored | COND |

**Events:** `gen_ai.evaluation.result` (`gen_ai.evaluation.name` required; `.score.value`, `.score.label`,
`.explanation`, `gen_ai.response.id`; plus `qa.eval.judge_version`) ·
`gen_ai.client.inference.operation.details` (**opt-in, keep off in production** — §2.5).

# Appendix B — Metric registry (index)

| Plane | Metrics | § |
|---|---|---|
| Control | `qa.run.duration` `qa.run.count` `qa.document.duration` `qa.rule.duration` `qa.rule.dispatched` `qa.rule.completed` `qa.rule.coverage_ratio` `qa.fanout.concurrency` `qa.fanout.straggler_count` `qa.queue.depth` `qa.queue.wait` `qa.tool.duration` `qa.tool.errors` `qa.retry.attempts` `qa.deadline.exceeded` `qa.idempotency.replay` | 3.1 |
| Cognition (behaviour) | `qa.agent.steps` `qa.agent.step_budget_utilisation` `qa.agent.terminations` `qa.agent.tool_calls` `qa.agent.inference_calls` `qa.agent.repeated_tool_call_ratio` `qa.agent.context_utilisation` `qa.agent.context_overflow` `qa.agent.tool_arg_invalid` `qa.model.response_model_mismatch` `qa.model.finish_reason` + `gen_ai.client.*` | 3.2 |
| Cognition (grounding) | `qa.evidence.count` `qa.evidence.verified_ratio` `qa.evidence.fabricated` `qa.evidence.stale_version` `qa.grounded_verdict_ratio` `qa.retrieval.empty` `qa.retrieval.top_score` `qa.retrieval.returned` | 3.3 |
| Quality | `qa.verdict.count` `qa.verdict.false_pass_rate` `qa.verdict.false_fail_rate` `qa.verdict.accuracy` `qa.eval.score` `qa.eval.judge_agreement` `qa.eval.human_judge_agreement` `qa.eval.coverage` `qa.confidence.calibration_error` `qa.verdict.mix_divergence` `qa.document.population_drift` | 3.4 |
| Consequence | `qa.escalation.rate` `qa.escalation.queue.depth` `qa.escalation.queue.age` `qa.review.dwell` `qa.review.override.rate` `qa.review.agreement` `qa.review.override.reason` `qa.review.probe.catch_rate` `qa.review.reopened` | 3.5 |
| Economic | `qa.cost.per_verdict` `qa.cost.per_document` `qa.cost.retry_attributed` `qa.cache.hit_ratio` `qa.cache.write_amortisation` `qa.tokens.per_verdict` `qa.tokens.reasoning_ratio` | 3.6 |

# Appendix C — Failure mode index

| Group | IDs | Theme |
|---|---|---|
| F1 | 1.1–1.8 | Control plane: silent rule drop, fan-out saturation, stragglers, retry storm, duplicates, partial aggregation, collector loss, baggage loss |
| F2 | 2.1–2.7 | Provider: silent version change, systemic degradation, rate limiting, context overflow, **context degradation**, truncation, routing shift |
| F3 | 3.1–3.8 | Trajectory: loops, budget exhaustion, **premature termination**, reasoning–action mismatch, scope drift, **manufactured certainty**, tool-arg malformation, bimodality |
| F4 | 4.1–4.6 | Grounding: fabricated citation, stale version, retrieval miss, **index regression**, evidence–verdict mismatch, chunk truncation |
| F5 | 5.1–5.4 | Verification: none, incorrect, over-trigger, **fail-open bypass** |
| F6 | 6.1–6.5 | Economic: cache collapse, loop blowout, retry spend, reasoning tokens, tier drift |
| F7 | 7.1–7.6 | Human: **rubber-stamp**, starvation, escalation collapse, escalation flood, override signal loss, reviewer drift |
| F8 | 8.1–8.6 | Evaluation: judge drift, instability, contamination, **Goodhart drift**, coverage collapse, calibration decay |

**Bold = the modes most likely to cause a silent, high-consequence compliance defect in this
architecture.** If you are prioritising detection work, start there.

# Appendix D — Adoption roadmap

| Phase | Duration | Deliverables | Exit criterion |
|---|---|---|---|
| **0 — Skeleton** | 2 weeks | Span tree (§2.1); correlation keys; change manifest; `termination_reason`; `coverage_ratio` | You can answer "did every rule run, and how did each agent end?" |
| **1 — Deterministic correctness** | 3 weeks | Citation verification; grounding metrics; guardrails as spans, failing closed; verdict schema validation | You can detect a fabricated citation within one aggregation window |
| **2 — Economics & behaviour** | 2 weeks | Token/cache accounting; cost per verdict; loop detection; retry classification; per-provider breakers | You can attribute a cost spike to a cause in under 15 minutes |
| **3 — Evaluation plane** | 4 weeks | Golden/regression/adversarial sets; judge stability gate; human review sample **over-sampling PASS**; eval events | You can state your false-pass rate **with a confidence interval** |
| **4 — SLOs & budgets** | 2 weeks | Two budgets; burn-rate alerting with behavioural proxies; error-budget policy; defensive escalation profile | You have exercised the defensive profile in a game day |
| **5 — Change safety** | 3 weeks | Seven-surface gating; shadow + canary; independent rollback for all seven; regression-set automation | You can roll back any surface in under 5 minutes without a deploy |
| **6 — Audit & assurance** | 3 weeks | Two-store split; retention policy; replay contract documented; adversarial review probes | You can answer every question in §8.1 from the record alone |

**Do not reorder phases 1 and 3.** Deterministic correctness is cheaper, faster and more reliable than
the evaluation plane, and building the evaluation plane first produces expensive machinery that
measures things an assertion could have proved (§1.5).

# Appendix E — Glossary

| Term | Meaning here |
|---|---|
| **Attenuation factor (a)** | `1 − α_J − β_J` for a judge; the factor by which judge noise shrinks an observed effect (§4.4.3) |
| **Convergence** | Whether an agent terminates for a good reason within budget (§1.4) |
| **Containment** | Whether guardrails and escalation held (§1.4) |
| **Defensive profile** | Runtime configuration that escalates all critical verdicts to humans; the graceful degradation mode (§4.6) |
| **Discriminator** | The observation separating a failure mode from its lookalikes (§5.0) |
| **Evidentiary reproducibility** | Ability to show what happened and why, without byte-identical replay (§8.3) |
| **False pass** | Non-compliant document verdicted compliant. The consequential error (§1.6) |
| **Green dashboard paradox** | All infrastructure signals healthy, business outcome wrong (§1.1) |
| **MDE** | Minimum detectable effect — the smallest regression an eval suite can detect (§4.4.2) |
| **Rubber-stamp signature** | Falling dwell + falling override + rising agreement (§3.5.1) |
| **Seven surfaces** | The seven things that change agent behaviour independently of a code deploy (§7.1) |
| **T0 / T1 / T2** | Telemetry retention tiers: audit / diagnostic / bulk (§2.6) |
| **Three planes** | Control (deterministic) / cognition (stochastic) / consequence (human) (§1.2) |
| **UNSTABLE** | A judge result with self-agreement < 0.8; counts as a failure (§4.4.4) |

# Appendix F — Sources

**Standards and specifications (P1)**

- [OpenTelemetry GenAI semantic conventions repository](https://github.com/open-telemetry/semantic-conventions-genai) — spans, metrics, events; Development status, no tagged releases
- [GenAI events (incl. `gen_ai.evaluation.result`)](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-events.md)
- [OpenTelemetry GenAI semantic conventions (redirect notice)](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [OpenInference semantic conventions](https://github.com/Arize-ai/openinference/blob/main/spec/semantic_conventions.md) — span kinds and namespaces
- [EU AI Act Article 12 — Record-keeping](https://artificialintelligenceact.eu/article/12/)
- [EU AI Act Article 19 — Automatically generated logs](https://artificialintelligenceact.eu/article/19/)

**Research (P1)**

- [Why Do Multi-Agent LLM Systems Fail? (MAST)](https://arxiv.org/abs/2503.13657) — 3 categories, 14 modes, κ=0.88 · [full text](https://arxiv.org/html/2503.13657v3)
- [TRAIL: Trace Reasoning and Agentic Issue Localization](https://arxiv.org/abs/2505.08638) — trace error taxonomy; ~11% best joint accuracy on error localisation · [full text](https://arxiv.org/html/2505.08638v2)

**Vendor documentation (P2)**

- [Splunk AI Agent Monitoring — key concepts](https://help.splunk.com/en/splunk-observability-cloud/observability-for-ai/splunk-ai-agent-monitoring/key-concepts-in-ai-agent-monitoring)
- [Splunk AI Agent Monitoring — setup](https://help.splunk.com/en/splunk-observability-cloud/observability-for-ai/splunk-ai-agent-monitoring/set-up-ai-agent-monitoring)
- [Dynatrace AI Observability via OpenTelemetry](https://docs.dynatrace.com/docs/observe/dynatrace-for-ai-observability/get-started/opentelemetry)
- [Langfuse — AI agent evaluation](https://langfuse.com/resources/engineering/ai-agent-evaluation)
- [Anthropic — Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Anthropic — Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

**Practitioner reporting (P3 — treat numbers as hypotheses, not benchmarks)**

- [The state of the OpenTelemetry GenAI semantic conventions (July 2026)](https://john-hodge.com/blog/opentelemetry-genai-semantic-conventions/) — pinning, coalescing, framework divergence
- [OTel GenAI conventions are not stable yet — what shipped in 2026](https://dev.to/azena-ai/opentelemetrys-genai-semantic-conventions-are-not-stable-yet-heres-what-actually-shipped-in-2026-3mke) — the three renames
- [How OpenTelemetry traces LLM calls, agent reasoning and MCP tools](https://greptime.com/blogs/2026-05-09-opentelemetry-genai-semantic-conventions)
- [LLM tail latency: why your p99 is a disaster when p50 looks fine](https://tianpan.co/blog/2026-05-07-llm-tail-latency-p99-heavy-tailed-distributions)
- [Fault isolation and circuit breaking for LLM agent pipelines](https://ranjankumar.in/fault-isolation-circuit-breaking-llm-agent-pipelines) — failure-class routing
- [The HITL rubber-stamp problem](https://tianpan.co/blog/2026-04-15-human-in-the-loop-rubber-stamp) — dwell time, override rate, automation bias
- [Prompt cache hit rate: the production metric your cost dashboard is missing](https://tianpan.co/blog/2026-04-20-prompt-cache-hit-rate-production-metric)
- [Your evals are flaky too](https://dev.to/saurav_bhattacharya/your-evals-are-flaky-too-stop-trusting-a-pass-rate-you-cant-reproduce-6pk) — judge self-agreement, UNSTABLE as failure
- [The death of the four golden signals](https://devops.com/the-death-of-the-four-golden-signals-designing-telemetry-for-non-deterministic-infrastructure/) — the argument this document deliberately rejects in favour of extension (§1.4)

---

*End of document. §11 governs how it changes.*
