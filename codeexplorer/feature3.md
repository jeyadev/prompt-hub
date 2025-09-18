Proposal: Checkpoint YAML & Alert Orchestration

Feature 3 — From “module flow” to monitorable checkpoints (conceptual design)


---

1) Objective

Create a single source of truth—a checkpoint.yaml per module—that captures the operational contract of that module’s flow: the key checkpoints (nodes) in the request path and the monitoring intents (metrics, logs, traces, alerts, routing).
This file is committed with the code and produced from your CodeExplorer’s flow.flow.json. A separate Terraform module will later compile these YAMLs into concrete monitors and alerts across Dynatrace/Splunk (and others).

Outcome: Every PR that changes a module can also update its checkpoint.yaml, keeping monitoring definitions versioned, reviewable, and in lockstep with the code.


---

2) Artifacts & Locations (concept)

docs/
  flows/
    modules/
      <module-name>/
        flow.mmd
        flow.svg
        flow.flow.json
        checkpoint.v1.yaml     # <-- this proposal
schemas/
  checkpoint.v1.schema.json    # JSON Schema for linting (future)


---

3) Checkpoint YAML — v1 Schema (concept)

# docs/flows/modules/orders/checkpoint.v1.yaml
version: 1
service: storefront
module: orders
env: shared                      # logical default; overridable in per-env overlays
owners:
  team: checkout-core
  slack: "#checkout-alerts"
  pagerduty: "PD_SERVICE_ORDERS"
metadata:
  tags: ["critical-path", "pci-scope"]
  runbook: "RUNBOOK.md#orders-module"
  notes: "Generated from flow.flow.json {{semantic_hash}}; hand-edited fields allowed."

slos:
  # Optional SLI/SLO anchors (used by alert compilers)
  - name: "orders-create-availability"
    sli:
      type: events_ratio
      numerator: dynatrace: "builtin:service.errors.server.rate?entitySelector=type(SERVICE),tag(service:storefront),requestName('POST /orders')"
      denominator: dynatrace: "builtin:service.request.count?entitySelector=type(SERVICE),tag(service:storefront),requestName('POST /orders')"
    objective: "99.9%/28d"
    page_policy:
      fast_burn: { window: "1h", budget_fraction: 0.02 }
      slow_burn: { window: "6h", budget_fraction: 0.05 }

checkpoints:
  - id: cp.entry.createOrder
    kind: entrypoint              # entrypoint|call|db|cache|queue|filesystem|compute|external|featureflag
    description: "HTTP POST /orders"
    source_refs:
      file: "src/orders/handlers/CreateOrderController.java"
      symbol: "CreateOrderController#create"
      span_name: "POST /orders"
    selectors:
      dynatrace:
        entitySelector: "type(SERVICE),tag(service:storefront),tag(module:orders)"
        requestName: "POST /orders"
      splunk:
        dataset: "prod_app_logs"
        query_hint: "source=nginx access method=POST path=/orders"
    monitors:
      metrics:
        - id: m.latency.p99
          provider: dynatrace
          metric: "builtin:service.response.time"
          scope: { requestName: "POST /orders" }
          objective: { percentile: 0.99, target_ms: 400 }
        - id: m.error.rate
          provider: dynatrace
          metric: "builtin:service.errors.server.rate"
          scope: { requestName: "POST /orders" }
          objective: { max_ratio: 0.005 }
      logs:
        - id: l.error.patterns
          provider: splunk
          search: |
            index=prod_app sourcetype=app:error module=orders ("CreateOrderException" OR "ValidationError")
          intent: "cluster_by_template_and_alert_on_novelty"
      traces:
        - id: t.critical_path
          provider: dynatrace
          selector: { spanName: "POST /orders" }
          intent: "emit_quantiles: [p95,p99], attribute_census: [error.kind]"

  - id: cp.call.paymentAuth
    kind: external
    description: "Call to PaymentService.authorize"
    source_refs:
      file: "src/orders/services/PaymentGateway.java"
      symbol: "PaymentGateway#authorize"
      span_name: "PaymentService/Authorize"
    selectors:
      dynatrace:
        entitySelector: "type(SERVICE),tag(service:payment)"
        requestName: "Authorize"
      splunk:
        dataset: "prod_app_logs"
        query_hint: "message=\"payment authorize\" OR class=PaymentGateway"
    monitors:
      metrics:
        - id: m.payment.latency.p99
          provider: dynatrace
          metric: "builtin:service.response.time"
          scope: { serviceTag: "service:payment", requestName: "Authorize" }
          objective: { percentile: 0.99, target_ms: 250 }
        - id: m.payment.error.rate
          provider: dynatrace
          metric: "builtin:service.errors.server.rate"
          scope: { serviceTag: "service:payment", requestName: "Authorize" }
          objective: { max_ratio: 0.01 }
      logs:
        - id: l.payment.declines
          provider: splunk
          search: |
            index=prod_app sourcetype=payment status=DECLINED OR error_code=*
          intent: "threshold: > 50 declines/5m -> ticket"

  - id: cp.db.ordersWrite
    kind: db
    description: "INSERT into orders"
    selectors:
      dynatrace:
        entitySelector: "type(DATABASE_SERVICE),tag(db:orders)"
    monitors:
      metrics:
        - id: m.db.errors
          provider: dynatrace
          metric: "builtin:service.errors.total.rate"
          scope: { entityTag: "db:orders" }
          objective: { max_ratio: 0.002 }
      logs:
        - id: l.db.deadlocks
          provider: splunk
          search: |
            index=infra_db sourcetype=postgresql error="deadlock detected"
          intent: "page_on_first_occurrence"

alerts:
  # Policy snippets the compiler will expand per provider
  - id: a.orders.availability.burn
    based_on: "slos.orders-create-availability"
    severity: page
    routing: { slack: "#checkout-alerts", pagerduty: "PD_SERVICE_ORDERS" }
    annotations:
      summary: "Orders create burning error budget"
      runbook: "RUNBOOK.md#orders-availability"

  - id: a.payment.dependency.regression
    based_on: "checkpoints.cp.call.paymentAuth.monitors.metrics[m.payment.latency.p99]"
    guardrail:
      compare_to: "stable_baseline:2h"
      condition: "ratio > 1.2 for 10m"
    severity: page

Design notes (high level):

checkpoints[] are the nodes from your module flow; each has selectors that let a compiler find the right metric/log/trace in Dynatrace or Splunk.

Monitors describe intent, not vendor-specific minutiae. The Terraform module handles provider-specific plumbing.

alerts[] may refer to slos[] or to a specific monitor by ID—keeping the policy declarative.

Owners/routing live here for one-stop lifecycle.



---

4) Semantics (how to think about it)

Stable IDs: cp.* and m.* IDs are immutable once published; changes are additive or deprecations.

Kinds signal what to watch: entrypoint (SLI candidates), external (dependency risk), db/cache/queue (saturation & errors), compute (CPU-bound paths), etc.

Selectors are just “how to find it” hints; the compiler can enrich or override per environment.

Objectives are targets, not thresholds. Alerting logic derives from SLO (burn-rate) or relative regression (baseline deltas) to curb false positives.



---

5) From flow.flow.json → checkpoint.yaml (mapping)

Flow manifest field	Checkpoint YAML field

nodes[].kind	checkpoints[].kind
nodes[].symbol,file,line	checkpoints[].source_refs
nodes[].span_name	checkpoints[].selectors.dynatrace.requestName
edges[] to foreign module	checkpoints[].kind: external + description
nodes[].attributes (db, queue)	provider-specific selector hints


A lightweight transformer (run locally or in CI) proposes a draft YAML. Humans edit descriptions, owners, objectives, and fine-tune selectors.


---

6) Cadence & Governance (concept)

When it updates

If a PR changes the module’s flow (node added/removed or edge changed), CI proposes updates to checkpoint.v1.yaml.

If a PR changes only code internals without flow change, YAML can remain untouched.


Review rules

checkpoint.v1.yaml falls under SRE + Module Owner review.

Bitbucket PR template gains a section:

[ ] Flow changed → checkpoint YAML regenerated/updated

[ ] Owners/routing verified

[ ] Objectives still valid (or updated)



Validation

Lint against schemas/checkpoint.v1.schema.json (structure & required fields).

Semantic checks (conceptual):

Every entrypoint must have at least one latency and one error-rate monitor.

Every external must carry a dependency guardrail (latency and error).

Owners/routing must be present for any alert with severity: page.




---

7) Provider Intent (kept declarative)

Dynatrace: selectors point to Services/Request Names; metric keys like builtin:service.response.time, builtin:service.request.count, builtin:service.errors.server.rate.

Splunk: search strings + intent hints (cluster_by_template, threshold, novelty); compiler turns them into saved searches, schedules, and alert actions.

Extensibility: keep provider as a key; adding Prometheus/Loki or Datadog later is additive.



---

8) Alerting Strategy (conceptual defaults)

SLO-based paging for entrypoint availability: multi-window/multi-burn (e.g., 2% budget/1h, 5%/6h).

Relative regression for dependencies: canary-style ratio vs. stable baseline (e.g., canary p99 < 1.15× stable over 10m).

Ticket-level signals for DB/queue saturation and error bursts with longer windows to avoid noise.

Seasonality-aware option (future): allow a baseline: holt_winters hint to switch the compiler to seasonality models.



---

9) Environments & Overrides (concept)

Support overlays that change only selectors/routing per env, keeping intent stable:

# docs/flows/modules/orders/checkpoint.v1.overrides-prod.yaml
overrides:
  env: prod
  owners: { slack: "#checkout-prod-alerts" }
  checkpoints:
    - id: cp.entry.createOrder
      selectors:
        dynatrace:
          entitySelector: "type(SERVICE),tag(env:prod),tag(service:storefront),tag(module:orders)"


---

10) Risks & Mitigations

Drift between YAML and reality → tie updates to module-flow changes; nightly audit on main compiles YAML and reports monitors that would change (“plan only”).

Over-specification (stale selectors) → keep selectors minimal and resolvable (tags over hard IDs); compiler can inject current entity IDs at apply-time.

Alert fatigue → defaults lean on SLO burn and relative deltas; warn on “static threshold” monitors without baselines.



---

11) Validation & Simulation (concept)

Plan-only compile: Terraform module runs in “plan” to show created/changed/destroyed monitors from the YAML.

What-if burn: simulate budget burn using last 30 days of metrics to estimate page frequency.

SPL dry-run: run Splunk searches with bounded earliest=-24h latest=now, return hit counts; block obviously noisy rules.

Scorecard: CI posts a small score (0–100) for the YAML completeness (entrypoints covered, dependencies guarded, owners set).



---

12) Ops-Ready Summary (checklist)

checkpoint.v1.yaml exists per module next to flow.flow.json.

IDs are stable; kinds reflect operational intent; owners/routing present.

Entry points have SLO anchors; dependencies have regression guardrails.

Provider selectors are hints; compiler handles vendor specifics.

CI lints structure and semantics; nightly “plan” audits drift without changing prod.



---

If you’re happy with this shape, we can draft the Terraform consumer spec next (inputs/outputs, mapping tables to Dynatrace & Splunk resources, and a plan/report format) so your platform team can implement without ambiguity.

