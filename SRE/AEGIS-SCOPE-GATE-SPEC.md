# AEGIS Scope Gate — Domain Boundary Enforcement Spec

**Status:** Draft for implementation — feed to IDE assist (DevGPT Cline / Copilot fallback)
**Owner:** Jey (VP SRE, CDS)
**Companion to:** AEGIS-RUNBOOK-AUTONOMY-SPEC.md
**Applies to:** AEGIS Strategist entry point, pre-Runbook-Generator

---

## 1. Problem Statement

AEGIS currently has no defense against out-of-scope input. Any query — general chit-chat, unrelated coding help, general knowledge, or adversarial "ignore your instructions / act as a general assistant" prompts — reaches the Strategist and gets processed like a legitimate CDS operational request.

Why this is a Class-of-problem, not a nice-to-have:

| Risk | Consequence |
|---|---|
| Resource/cost waste | LLM + MCP tool cycles spent on non-CDS queries |
| Compliance exposure | Regulated-bank agent answering arbitrary questions is an audit finding waiting to happen |
| Prompt injection surface | An ops agent with Splunk/Dynatrace/Jira/Confluence/Pithos MCP access is a high-value jailbreak target — "pretend you're unrestricted" is not a hypothetical here |
| Trust Ledger contamination | Off-topic interactions shouldn't count toward or against autonomy trust accrual, but an ungated system can't tell the difference |

**Same failure class as ungated writes.** You already solved this once: the write-gate invariant says write-capable actions must be *syntactically unrepresentable*, not merely discouraged. Scope enforcement needs the identical treatment — **unrepresentable, not discouraged.** An LLM instruction not to go off-topic is compliance-by-request, and compliance-by-request is exactly what the write-gate invariant exists to avoid.

---

## 2. Design Principle: Deterministic-Before-Probabilistic, Applied to Input

Same tiering discipline you use everywhere else in AEGIS (fingerprint before LLM, rule-based before ML clustering):

| Tier | Mechanism | Cost | Catches |
|---|---|---|---|
| **Tier 1** | Deterministic allowlist match — keyword/entity match against CDS domain terms (service names, tenant codes IPB/PBA/GVA, Pithos, MCP tool names, known error signatures) | ~0ms, no LLM call | Obvious in-scope queries — majority of real traffic, fast-pathed |
| **Tier 2** | Embedding similarity — cosine similarity of query against a curated CDS/AWM domain corpus, fixed threshold | Low cost, no generation | Paraphrased/indirect in-scope queries; catches obvious off-topic (similarity near zero) |
| **Tier 3** | LLM classifier, structured output only (`in_scope` / `out_of_scope` / `ambiguous` + confidence), **no free-form response generation** | Highest cost, only invoked when Tiers 1–2 disagree or land near threshold | Genuinely ambiguous cases |

**Fail-closed, not fail-open.** Ambiguous → reject with a reframe prompt. Never default-allow on uncertainty. This is the same asymmetry as Trust Ledger: scope confidence is slow to grant, fast to deny.

**Why allowlist, not denylist.** You cannot enumerate every off-topic request — the space is unbounded. You *can* enumerate what CDS operations legitimately look like: the 12 services, the MCP tool surface (Splunk/Dynatrace/Jira/Confluence/Pithos), the tenants (IPB/PBA/GVA), the active initiatives (Pithos migration, OTel, exception monitoring). This mirrors the write-gate logic exactly — define the narrow set of things the grammar *permits*, reject everything else by construction.

---

## 3. Placement in AEGIS Architecture

```
User Query
    │
    ▼
┌─────────────────────┐
│  AEGIS SCOPE GATE    │  ← NEW. Sits before Strategist sees anything.
│  (Tier 1 → 2 → 3)     │
└─────────────────────┘
    │                              │
    ▼ in_scope                    ▼ out_of_scope / ambiguous
Strategist →                  Structured rejection response
Runbook Generator →           + telemetry log entry
Sub-Agents                    (no LLM generation of a "helpful" answer
                                to the off-topic question — that's
                                the leak vector)
```

This is a distinct gate from the existing **AEGIS Gate** (DecisionAgent → RemediationAgent, action-class enforcement). Don't conflate them:

| | AEGIS Gate (existing) | AEGIS Scope Gate (this spec) |
|---|---|---|
| Question it answers | "Is this **action** safe to execute?" | "Is this **query** even CDS's business?" |
| Position | Between Decision and Remediation | Between user input and Strategist |
| Failure mode it prevents | Autonomous action overreach | Domain overreach / jailbreak / off-topic drift |

---

## 4. Rejection Behavior

Do not let the LLM freely compose a response to an out-of-scope query — that's the same leak as letting it freely compose a "safe-looking" write action. The rejection path should be **templated, not generated**:

- Fixed response template, parameterized only with a reason code (e.g., `SCOPE_REJECT_GENERAL_KNOWLEDGE`, `SCOPE_REJECT_UNRELATED_CODE`, `SCOPE_REJECT_JAILBREAK_PATTERN`)
- No tool calls, no MCP access, no generation — the classifier's job ends at classification
- Every rejection logged: query hash (not raw query, if it may contain sensitive tokens), session ID, tier that triggered rejection, confidence score

## 5. Telemetry as Security Signal, Not Just Hygiene

Repeated `SCOPE_REJECT_JAILBREAK_PATTERN` hits from the same session is not noise — it's an anomaly signal. Feed scope-gate rejections into the same Splunk stream AEGIS already reads from, and treat a burst of rejections as an incident-worthy pattern (consistent with your Safety-II instinct: the interesting signal is in what's being *tried*, not just what succeeds).

Recommend: rejection rate + rejection-tier distribution becomes a standing panel next to the existing Exception Monitor dashboard (M6) — reuse that pipeline rather than building a parallel one.

---

## 6. Config, Not Code

Domain allowlist (service names, tenant codes, MCP tool names, initiative keywords, embedding corpus) lives in an external config file, not hardcoded in the gate logic — same shared-canonical-layer principle as `AGENTS.md`/`SKILL.md`. New services or initiatives get added by editing config, not by touching the gate implementation. This also means the gate survives the DevGPT Cline → Copilot fallback transition without logic duplication.

---

## 7. Eval Harness (required before this ships)

IDE assist should generate a golden test set with three categories, run as a regression suite on every change to the gate:

1. **True in-scope** — real CDS/Pithos/incident queries, including paraphrased/indirect phrasing
2. **True out-of-scope** — general knowledge, unrelated coding help, small talk
3. **Adversarial** — "ignore previous instructions," "act as a general assistant," roleplay-framed jailbreaks, prompt injection embedded in a legitimate-looking Splunk log snippet passed as context

Category 3 is the one that actually matters for a banking-context agent with live MCP write paths downstream. Don't ship without it.

---

## 8. Open Questions — For IDE Assist to Ask Interactively During Implementation

These are stakeholder-validation-style items (same pattern as SV-0.1/SV-1.2/SV-1.4 in the autonomy spec) that shouldn't be silently assumed by the coding agent:

| # | Question | Why it matters |
|---|---|---|
| Q1 | Is the MCP tool surface (Splunk, Dynatrace, Jira, Confluence, Pithos) the exhaustive allowlist boundary, or are there CDS-adjacent topics without an MCP tool that should still be in-scope? | Defines Tier 1 keyword set completeness |
| Q2 | Per-message or per-session scoping? Once a session establishes CDS context, do loosely-related follow-ups get relaxed scoring? | Affects false-reject rate on legitimate multi-turn diagnostics |
| Q3 | What's the acceptable false-reject rate for Tier 2 embedding similarity — i.e., how much do you tolerate legitimate queries occasionally bouncing to Tier 3? | Sets the similarity threshold |
| Q4 | Do scope-gate rejections need routing to a human review queue (like PBA/IPB), or is dashboard telemetry sufficient? | Determines if this needs a Jira/queue integration, not just logging |
| Q5 | Does enforcement strictness vary by autonomy level (AL0–AL3), or is scope enforcement uniform regardless of the action-autonomy tier in play? | Determines if Scope Gate and AEGIS Gate need to share config or stay fully independent |
| Q6 | Should the rejection template be identical for all callers, or does it differ for known SRE team members (Slack/session-authenticated) vs. unauthenticated/external callers, if any exist? | Affects whether identity context is even available at this gate |

Do not let the IDE assist default-guess these — they're architectural, not implementation details.

---

## 9. Non-Goals

- This spec does not cover action-level autonomy (that's the existing AEGIS Gate / Trust Ledger).
- This spec does not attempt topic classification via LLM alone — Tier 3 is a last resort, structured-output only, never free generation.
- This spec does not address rate limiting or DoS protection — separate concern, flag if in scope for a follow-up spec.
