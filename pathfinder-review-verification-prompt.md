# Prompt: verify the CC Pathfinder architecture review against the codebase

Paste everything below the line into your IDE assistant. Fill the four bracketed
placeholders first. Attach or point it at the review document if you have it;
the finding register is embedded here so the prompt still works if you don't.

---

You are performing an evidence-based verification pass on an architecture review.

A design review of CC Pathfinder was produced from a diagram alone, before any
code was inspected. It contains 20 findings and 10 open decisions. Your job is
to determine which of them survive contact with the actual implementation, and
to find what the diagram-level review could not see.

## Inputs

- Codebase root: `[REPO PATHS — e.g. ./pathfinder-core, ./pathfinder-agents, ./infra]`
- Related repos, if separate: `[eSign/CDS/CX integration repos, IaC repo, policy repo]`
- Design docs, ADRs, runbooks: `[PATHS OR CONFLUENCE SPACE]`
- Environment scope: `[which environment the code reflects — dev, UAT, prod]`

## Rules you must not break

1. **The review is a set of hypotheses, not conclusions.** It was written without
   code access. Do not confirm a finding because it appears in the register.
   Confirm it only when you can point at code, config, or a manifest that
   demonstrates it. Your default posture toward each finding is "unverified".

2. **Cite evidence for every verdict.** Every confirmed or closed finding needs
   at least one `path/to/file.ext:line-range` reference, or an explicit statement
   that you searched and found nothing. Verdicts without file references are
   invalid output.

3. **Distinguish absence from ignorance.** "This control does not exist in the
   codebase" and "I could not locate this control" are different verdicts. Use
   `MITIGATED`, `CONFIRMED`, or `UNVERIFIABLE` precisely. If a component lives in
   a repo you cannot read, say so and mark it `UNVERIFIABLE — out of scope`.

4. **Do not infer intent from names.** A class called `PolicyEngine` proves
   nothing about whether authorization is deterministic. A module called
   `AuthMiddleware` proves nothing about whether end-user identity reaches the
   downstream call. Read the implementation, follow the call path, and report
   what it actually does.

5. **You are expected to disagree with the review.** If a finding is wrong,
   overstated, or already solved, say so plainly and show why. A verification
   pass that confirms everything is a failed verification pass. State explicitly
   how many findings you closed or downgraded.

6. **Do not modify code.** This is a read-and-report task. Propose changes in the
   report; make none in the repo.

## Evidence to gather, by theme

Trace these paths through the actual code before writing any verdict.

**Identity and authorization (F-03, F-06, F-12, F-19)**
- Follow one user request end to end from the chat handler to a downstream API
  call. Record every place credentials are attached or swapped.
- Is the downstream call made with the end user's token, a delegated token, or a
  service account? Find the credential source in code, not in a diagram.
- Is any authorization decision made by a model call? Search for prompt templates
  containing words like "allowed", "permitted", "can the user".
- Are result sets filtered before or after retrieval? Find the query construction.
- What do denial responses say? Find the error and refusal message strings.

**Write paths and blast radius (F-11, F-17)**
- Enumerate every downstream call and classify it read or write. This resolves
  decision D-01, which roughly a third of the review depends on.
- For each write: is there a dry-run mode, an idempotency key, an approval gate,
  a rate limit, a circuit breaker, a rollback path? Report the seven-point status
  per capability.
- Does anything write back to ServiceNow? Under whose identity, and is agent
  authorship marked in the payload?

**Agent structure (F-01, F-02, F-05, F-08)**
- Where does inference actually happen? List every model call site with its
  purpose.
- Does retrieved document text enter the context of any call that selects or
  parameterises a tool? Trace the variable, do not assume from structure.
- How is tool or capability knowledge represented — hardcoded, config, a
  registry, or generated from downstream schemas? Who owns each source file?
- What happens on low-confidence or empty retrieval? Find the branch. If there
  is no branch, that is the finding.

**Knowledge base (F-10, F-20)**
- How many distinct corpora and indexes exist? What is the ingestion and reindex
  trigger? Is there document-level access control, provenance metadata, or
  tombstoning of superseded content?
- Are prompts, model versions, index snapshots, and policy rules versioned as
  release artefacts with a rollback path?

**Reliability and cost (F-04, F-13, F-14, F-18)**
- Build the dependency list from code and manifests, then answer: which
  dependencies are shared with the systems Pathfinder is meant to diagnose?
- Is there instrumentation of the agent itself — spans per hop, tool traces,
  token cost, retrieval quality? Name the library and the export target.
- Are telemetry queries on-demand per turn or pre-aggregated? Are they cached or
  rate-limited?
- Is there a latency budget, a timeout policy, or streaming in the response path?

**Evaluation (F-09, F-16)**
- Is there a test suite covering agent behaviour, a golden set, or any offline
  evaluation? Is it wired into CI as a gate, or is it advisory?
- Is there a bounded, declared list of supported intents anywhere in code or
  config?

**Session and scope (F-15)**
- Where is conversation state stored, what is its retention, and who can read it?

## Verdict vocabulary

Assign exactly one to each finding:

- `CONFIRMED` — the code exhibits the problem. Cite it.
- `PARTIALLY MITIGATED` — a control exists but is incomplete or bypassable. State
  the gap and the bypass.
- `MITIGATED` — the code already solves this. Cite the control and say why it is
  sufficient.
- `NOT APPLICABLE` — the review misread the design. Explain what the design
  actually does.
- `UNVERIFIABLE` — the relevant code is out of scope, generated, or absent from
  the repos you can read. Say which repo or artefact would settle it.

Where the evidence justifies it, propose a severity change with
`SEVERITY 2 → 1` or `SEVERITY 1 → 3` and one sentence of justification. Findings
that turn out to be more dangerous in implementation than on paper matter more
than findings that are already handled.

## New findings

The review saw a diagram. You are seeing code. Report every material problem the
diagram could not expose, at minimum across: secret and credential handling,
error handling and retry semantics, input validation, dependency and supply chain
risk, concurrency and race conditions, resource lifecycle and leaks, timeout and
cancellation, logging of sensitive data, test coverage of the paths that carry
risk, and any prompt-construction pattern that concatenates untrusted input.

Number these `N-01` onward, in the same shape as the existing findings:
statement, why it matters, concrete failure scenario, remediation, question to
answer, plus file references.

## Decisions to resolve from evidence

For each, answer `RESOLVED — <answer>` with citations, or
`STILL OPEN — <what artefact would settle it>`.

- D-01 Does the Service Orchestrator ever write?
- D-02 What is the intent inventory, and what share needs dynamic planning?
- D-03 Is the Rule agent a model call or a rules engine?
- D-04 Can an end-user identity reach eSign, CDS, and CX?
- D-05 Who owns freshness for each knowledge corpus?
- D-06 What does Pathfinder say when it does not know?
- D-07 What is the availability target relative to its dependencies?
- D-08 Who authors authorization policy?
- D-09 What is the golden set, and who curates it?
- D-10 What happens to an in-flight agent action during a change freeze?

## Output format

```
## Coverage
Repos and paths read. What you could not read and why. Rough percentage of the
runtime system this pass actually covers. Be honest about the gaps — an
overstated coverage claim invalidates every verdict below it.

## Summary
Counts by verdict. Number of findings closed or downgraded. Number of new
findings. The three things that most change the original review's conclusions.

## Finding verdicts
One block per finding, F-01 through F-20, each with:
  Verdict, evidence (file:line references), reasoning, revised remediation.

## New findings
N-01 onward.

## Decision resolutions
D-01 through D-10.

## Revised top risks
The five highest-risk items after this pass, in order, with the single next
action for each. This list may differ substantially from the original ranking.
That is the point of the exercise.
```

## Finding register

F-01 Three workloads (knowledge, state, operate) fused into one plane — sev 1
F-02 Retrieved document content can steer tool selection — sev 1
F-03 No end-user identity propagation, confused deputy — sev 1
F-04 Shares a failure domain with the estate it diagnoses — sev 1
F-05 Rule agent centralises capability knowledge owned by other teams — sev 1
F-06 Non-deterministic authorization if the Rule agent is a model — sev 1
F-07 Incident work forced through a chat turn — sev 2
F-08 No verification step and no defined refusal — sev 2
F-09 No eval harness, no feedback capture, no regression gates — sev 2
F-10 Four corpora with four owners drawn as one store — sev 2
F-11 Write path undefined — sev 2
F-12 Post-retrieval filtering leaks through synthesis — sev 2
F-13 No observability of the agent itself — sev 3
F-14 No latency or cost budget — sev 3
F-15 No session state or episodic memory — sev 3
F-16 Unbounded scope means unbounded evaluation surface — sev 3
F-17 ServiceNow write-back semantics undefined — sev 2
F-18 Telemetry query pattern, cost, and rate limits unspecified — sev 3
F-19 Denial messages leak existence and ownership — sev 2
F-20 No versioning or rollback story for prompts, models, and indexes — sev 3

Begin with the coverage section. Do not write any verdict before you have read
the code that supports it.
