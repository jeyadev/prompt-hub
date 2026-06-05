# CDS Splunk Data Dictionary

> **This file is the lever.** The mode's output quality is capped by how completely and
> accurately this is filled in — not by the model. Treat it as a living artifact: update it as
> you confirm real index/sourcetype/field names, and re-sync it into `.roo/rules-splunk-sre/`
> and `.clinerules/`. Anything left `<PLACEHOLDER>` will surface as `<PLACEHOLDER>` in generated
> SPL — by design, so you never ship a guessed index name to prod Splunk.

---

## 1. Indexes
| Placeholder token | Real index name | Contains | Retention | Accelerated DM? |
|---|---|---|---|---|
| `<CDS_APP_INDEX>` | `<fill>` | CDS application logs (12 services) | `<fill>` | `<fill>` |
| `<CDS_QUEUE_INDEX>` | `<fill>` | PBA / IPB queue + ticket state | `<fill>` | `<fill>` |
| `<CDS_DEPLOY_INDEX>` | `<fill>` | Release / deploy events | `<fill>` | — |
| `<CDS_INFRA_INDEX>` | `<fill>` | Host / container / GKP metrics | `<fill>` | `<fill>` |

## 2. Sourcetypes
| Placeholder token | Real sourcetype | Index | Format (json/kv/raw) | CIM-compliant? |
|---|---|---|---|---|
| `<CDS_APP_SOURCETYPE>` | `<fill>` | `<CDS_APP_INDEX>` | `<fill>` | `<fill>` |
| `<CDS_QUEUE_SOURCETYPE>` | `<fill>` | `<CDS_QUEUE_INDEX>` | `<fill>` | `<fill>` |
| `<CDS_DEPLOY_SOURCETYPE>` | `<fill>` | `<CDS_DEPLOY_INDEX>` | `<fill>` | — |

## 3. Key fields (confirm real names — these are ASSUMED until verified)
| Assumed field | Real field name | Meaning | Indexed? | Sensitive? |
|---|---|---|---|---|
| `service` | `<fill>` | Service emitting the event | `<fill>` | no |
| `status` | `<fill>` | HTTP/app status code | `<fill>` | no |
| `duration_ms` | `<fill>` | Request latency in ms | `<fill>` | no |
| `endpoint` | `<fill>` | API path | `<fill>` | maybe |
| `doc_id` | `<fill>` | Document identifier | `<fill>` | **likely — hash before display** |
| `correlation_id` | `<fill>` | Cross-service trace id | `<fill>` | no |
| `action` | `<fill>` | e.g. write_ok / read / 404 | `<fill>` | no |
| `error_class` | `<fill>` | Categorised error | `<fill>` | no |
| `backend` | `<fill>` | mongo / pithos / s3 | `<fill>` | no |
| `queue_name` | `<fill>` | PBA / IPB / ... | `<fill>` | no |
| `queue_depth` | `<fill>` | Items in queue | `<fill>` | no |
| `ticket_age_hours` | `<fill>` | Age of oldest ticket | `<fill>` | no |

## 4. Services (12 critical — fill the rest)
| Service | Role | Pithos-migrating? | Tier-1 dashboard? |
|---|---|---|---|
| cds-find | Document retrieval | yes | yes |
| cds-store | Document write/store | yes | yes |
| `<fill>` | `<fill>` | `<fill>` | `<fill>` |
| _… add remaining services …_ | | | |

## 5. SLOs / thresholds (drive ALL dashboard RAG colors and error-budget math)
| Token | Service | Objective | Window | Notes |
|---|---|---|---|---|
| `<SLO_TARGET>` | cds-find | `<fill, e.g. 0.999>` availability | 28d | good = status<500 |
| `<LATENCY_OBJECTIVE_MS>` | cds-find | p95 < `<fill>` ms | 28d | |
| `<AGING_THRESHOLD_HOURS>` | PBA queue | oldest ticket < `<fill>` h | rolling | drives queue-hygiene RAG |
| `<DEPTH_THRESHOLD>` | PBA queue | depth < `<fill>` | rolling | |
| `<PITHOS_RAMP_TARGET>` | cds-store/find | planned ops/min at current ramp phase | per phase | volume-vs-ramp panel |

## 6. Macros to define in Splunk (reuse, prevent SLI drift)
| Macro | Definition (fill once confirmed) |
|---|---|
| `cds_app_base` | `index=<CDS_APP_INDEX> sourcetype=<CDS_APP_SOURCETYPE>` |
| `cds_valid_events` | `<good/valid event filter for SLI denominator>` |
| `cds_server_errors` | `status>=500` |

## 7. Environment facts the mode must know
| Question | Answer |
|---|---|
| Dashboard Studio or Simple XML for new builds? | `<fill — default Studio>` |
| Splunk version | `<fill>` |
| Splunk MCP reachable from VDI? (`:8089`) | `<UNCONFIRMED — see README landmine 1>` |
| MCP auth model (OAuth 2.1 / token / SID) | `<UNCONFIRMED — see README landmine 2>` |
| Read-only role name for MCP service account | `<fill>` |
| Sandbox/dev Splunk available for validation? | `<fill>` |
