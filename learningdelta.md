You want maximum retention, not a pleasant chat. Good. Treat this walkthrough like you’re reverse-engineering a production system you’ll soon own. Here’s the mental model + checklist that forces the builder to reveal what matters.

---

## 1) The mental model (how to “see” the system)

Picture it as 6 layers. If you can map the tool into these layers, you’ll remember it and contribute safely.

1. **User experience layer**
   How users define workflows, trigger them, pass inputs, see outputs.

2. **Control plane (definition + orchestration)**
   Where workflow definitions live, versioning, validation, scheduling, routing, dependency graph.

3. **Data plane (execution runtime)**
   The engine that runs steps: state machine, retries, timeouts, fan-out/fan-in, concurrency limits.

4. **Connectors / integrations**
   Splunk, Cortex, HTTP endpoints, web scraping. Auth, rate limits, pagination, data shapes.

5. **State / storage**
   Run state, logs, artifacts, secrets, templates, audit events. What’s persisted, for how long.

6. **Safety & governance**
   RBAC, approvals, allowlists, sandboxing, egress rules, PII controls, auditability.

Your job in the walkthrough is to “label every major piece” into one of these 6 buckets. If something doesn’t fit, it’s a hidden risk.

---

## 2) The retention method: build a “one-page map” live

During the walkthrough, take notes in this structure (literally headings):

* **How a workflow is defined**
* **How it is triggered**
* **How a run executes**
* **How integrations are called**
* **How state is stored**
* **How it’s observed**
* **How it’s secured**
* **How it fails**
* **How we deploy changes**
* **How we test changes**

If you force yourself to fill these slots, you’ll retain 80%+.

---

## 3) Mental checklist for the walkthrough (what to extract)

### A) Product boundaries (or you’ll assume wrong things)

* What is the tool *not* meant to do?
* Is it for read-only workflows or can it do actions/remediation?
* Is it meant for L1/L2 operations or general automation?

### B) Workflow DSL / definition model

* Is a workflow stored as JSON/YAML/DB rows?
* Is there a formal schema? Validation?
* Are there reusable components (subflows, templates, macros)?
* Versioning: can two versions run simultaneously? rollback?

### C) Triggers

* Supported triggers: manual, webhook, schedule, event-driven?
* Can it integrate with ServiceNow/Jira/Email today or later?
* How are inputs passed (schema, types, defaults)?
* Idempotency: can the same trigger create duplicate runs?

### D) Execution engine (most important for SRE sanity)

* Execution model: DAG vs linear steps vs state machine?
* Concurrency controls: per workflow/per tenant/global?
* Retries: per step? exponential backoff? jitter? max attempts?
* Timeouts: per step + per run?
* Fan-out/fan-in supported? parallel steps?
* Cancellation: can you abort a run safely?
* Reentrancy: can it resume from step N after crash?

### E) Integrations (Splunk/Cortex/HTTP/Scrape)

For **each connector**, force answers to:

* Auth method (service account, SSO, tokens) and where secrets live
* Network path (egress allowlist, proxies, DNS)
* Rate limits + backpressure behavior
* Pagination handling (Splunk results, API pages)
* Data normalization: does the platform transform results or is it raw?
* Error mapping: how are connector errors surfaced to the workflow?

**Splunk-specific**

* How does it query (search/jobs API, streaming)?
* How does it handle large result sets (limit, sampling, summarization)?
* How do you prevent expensive searches from melting Splunk?

**Scraping**

* Static-only guaranteed? Any JS rendering?
* How does it parse (CSS selectors, XPath)?
* What happens when markup changes (fragility handling)?

### F) State, storage, and audit

* Where is run state stored? (DB type, schema)
* What’s persisted:

  * inputs, intermediate results, outputs?
* Retention policy for run logs/artifacts
* Can you replay runs? export traces?
* Audit: who ran what, when, using what inputs?

### G) Observability (the platform must be observable itself)

* Metrics:

  * run count, success rate, duration, step retries, connector latencies
* Logs:

  * per-run logs, step logs, correlation IDs
* Tracing:

  * do workflows emit trace context?
* Alerting for the platform:

  * queue buildup, error spikes, dead letter behavior

If they can’t show platform dashboards, it’s not an enterprise tool yet, it’s a hobby with a UI.

### H) Security & governance

* AuthN: how users log in (SSO, IDA)
* AuthZ:

  * role-based access to workflows, connectors, secrets
* Secret management:

  * where, how rotated
* Data governance:

  * PII handling, masking, redaction
* Egress controls:

  * endpoint allowlists, domain allowlists for scraping
* Multi-tenancy:

  * separation between teams/services?

### I) Deployment & contribution model (since you’ll contribute)

* Repo structure and architecture boundaries
* Local dev setup (docker-compose? k8s? mocks?)
* CI/CD pipeline and environments (dev/stage/prod)
* Backward compatibility rules for workflow definitions
* How they test connectors (mocks vs integration tests)
* Release strategy:

  * feature flags?
  * migration approach?

### J) Failure modes (force them to tell you the ugly parts)

Ask directly:

* “Top 5 production incidents this tool has caused or suffered?”
* “What breaks most often: connectors, storage, UI, runtime?”
* “What’s the biggest scaling bottleneck today?”
* “What’s the worst possible misuse by an engineer?”

This gets you real truth fast.

---

## 4) The “golden path” questions (ask these in order)

If you only ask 10 questions, ask these:

1. **Walk me through one workflow run end-to-end** (trigger → steps → outputs).
2. **Where are workflows stored and versioned?**
3. **What is the execution model and how do retries/timeouts work?**
4. **How do you enforce concurrency and prevent runaway workflows?**
5. **How do Splunk queries run and how do you cap cost/results?**
6. **What does a run’s state look like in storage and how long is it retained?**
7. **How do you observe the platform (metrics/logs/traces)? Show dashboards.**
8. **How do auth, secrets, and access control work?**
9. **How do we safely deploy changes and roll back?**
10. **What are known pain points and biggest tech debt items?**

That set alone will make you dangerous in a good way.

---

## 5) A simple note-taking template you can paste into Confluence

Use this during the walkthrough:

* **Purpose / Scope**
* **Architecture sketch (components + data flow)**
* **Workflow definition model**
* **Triggers**
* **Execution engine**
* **Connectors**

  * Splunk
  * Cortex
  * HTTP endpoints
  * Scraping
* **State & storage**
* **Observability**
* **Security & governance**
* **Deployment & contribution**
* **Known limitations / tech debt**
* **Open questions / follow-ups**
* **Immediate contribution opportunities (low-risk)**

---

## 6) One last piece of advice you’ll thank me for later

Don’t let the builder walk you through UI features first. That’s how you leave with “it has buttons.”
You want:

**Run lifecycle → execution model → connectors → state → safety → deploy.**

Everything else is decoration.

Use this checklist like a scalpel and you’ll retain the tool’s “shape” in your head, not just a pile of facts.
