I want to understand all the places in this repository where we are writing to or updating messages in MQ (e.g., producing/publishing/sending messages).  
- Search for code that interacts with MQ clients, producers, or publish/send APIs.  
- Identify the functions, classes, or modules where MQ messages are sent.  
- For each location, summarize: 
  1. The queue/topic name being used (if available).  
  2. The structure or fields of the message being passed.  
  3. Any metadata, headers, or correlation IDs being set.  
  4. Which part of the application triggers this message.  
Return a consolidated report so I can see what information we’re passing into MQ across the repo.




Here’s a ready‑to‑paste Copilot Chat prompt that will:

1. enumerate all endpoints in the repo, 2) trace each endpoint’s full execution flow (DB, MQ, external calls, etc.), and 3) create/update a Markdown report file.




---

Prompt for GitHub Copilot Chat

You are analyzing this repository to produce a complete catalog of HTTP endpoints and their execution flows.

GOAL
==== 
1) Discover ALL HTTP endpoints configured in the codebase.
2) For EACH endpoint, generate a detailed flow of operations the request triggers.
3) Write everything into a single Markdown file at docs/endpoints-flow.md (create the folder/file if they don’t exist), replacing the file content entirely.

SCOPE HINTS
=========== 
Detect endpoints across common frameworks/languages:
- Java/Spring (annotations like @RestController, @Controller, @RequestMapping, @GetMapping, @PostMapping, @PutMapping, @DeleteMapping)
- Node/Express (app.get/post/put/delete, router.METHOD, express.Router())
- Python/FastAPI/Flask (FastAPI .get/.post decorators, Flask @app.route)
- Go (gorilla/mux, chi, net/http handlers)
- Ruby on Rails (routes.rb, controller actions)
- .NET (MapGet/MapPost, ControllerBase with [HttpGet]/[HttpPost] etc.)
Also check OpenAPI/Swagger files, route registries, middleware/filters, API gateway config (if present).

FOR EACH ENDPOINT, CAPTURE
========================== 
- Endpoint signature: HTTP method, path, controller/handler function name, file path, line link (file:line).
- Request contract: path/query/body params, content-type, auth requirements (e.g., JWT, API key, OAuth scopes), rate-limits/feature flags/AB toggles.
- Response contract: status codes, response schema/model, error shapes.
- Middlewares/filters/interceptors: authn/authz, validation, rate limiting, tracing, logging, request shaping.
- Execution FLOW (step-by-step):
  1) Input validation and transformations
  2) Calls to internal services/modules
  3) Database operations (ORM calls, SQL queries) – include table/collection names and key fields if visible
  4) Cache usage (get/set/delete; keys/prefixes)
  5) MQ/stream publishes (topic/queue, headers/correlationId, payload structure)
  6) External API calls (host/path, method, headers set, request/response fields)
  7) Background jobs / async tasks / schedulers triggered
  8) Telemetry: metrics/logs/traces emitted (metric names, labels/tags, log events), and any SLO/SLA implications
  9) Error handling/retries/circuit breakers/idempotency tokens and compensating actions

- Security/Resilience notes: input sanitization, PII handling, secrets usage, timeouts, retry/backoff, circuit breaker, bulkheads, fallbacks.
- Performance notes: obvious N+1s, heavy serialization, large payloads, blocking IO on hot paths.

OUTPUT FORMAT (Markdown)
========================
Rewrite docs/endpoints-flow.md with:
1) Title and generated timestamp.
2) Repo-wide TOC of endpoints in a table:

| Method | Path | Handler | File:Line | Auth | Notes |
|-------:|------|---------|-----------|------|-------|

3) Per-endpoint section with the following structure:

#### {METHOD} {PATH}
**Handler**: `{package.module.Class#method}`  
**File**: `relative/path/to/file.ext:LINE`  
**Auth**: `{none | JWT | OAuth2(scopeA, scopeB) | API key | …}`  
**Request**  
- Path params: `{...}`  
- Query params: `{...}`  
- Body schema: `{...}`  

**Response**  
- Status codes: `200, 400, 401, 404, 500`  
- Body schema: `{...}`  

**Middlewares / Filters**  
- `{name -> purpose}`

**Execution Flow (step-by-step)**  
1. …  
2. …  
3. …

**Data Access**  
- DB reads/writes: `{table/collection, operation, key fields}`  
- Cache ops: `{get/set/del, key patterns}`  

**Messaging / MQ**  
- Publish to: `{queue/topic}`  
- Headers/metadata: `{correlationId, messageId, contentType, custom headers}`  
- Payload fields: `{...}`  

**External Calls**  
- `{METHOD host/path} – headers set: {...}; request fields: {...}; response fields: {...}`  

**Telemetry & SLO Signals**  
- Metrics: `{name(labels)}`  
- Logs: `{event names/levels}`  
- Traces: `{span names, attributes}`  

**Errors & Resilience**  
- Exceptions handled, retry/backoff, timeouts, circuit breakers, idempotency keys, compensations.

**Mermaid Sequence (if enough detail is available)**  
```mermaid
sequenceDiagram
  participant C as Client
  participant GW as API Gateway (if any)
  participant S as {Service/Handler}
  participant DB as {DB/Cache}
  participant MQ as {MQ/Topic}
  participant X as {External API}

  C->>S: {METHOD} {PATH}
  S->>S: Validate & authn/authz
  S->>DB: Read/Write {entity}
  S-->>MQ: Publish {topic} (correlationId=..., headers=...)
  S->>X: {METHOD} {external/path}
  S-->>C: {status} {summary}

DISCOVERY STEPS (what I want you to do)

1. Repo-wide search for routing/endpoint declarations and OpenAPI specs.


2. For each endpoint, jump to its handler and recursively trace calls across files/modules to find DB/MQ/Cache/External interactions and middleware.


3. Extract concrete details from code (variables/constants) whenever possible. If dynamic, state “dynamic (computed from …)”.


4. Include code references as relative/path:line for each significant step.


5. If the repo is large, process in batches and keep appending to the same Markdown (but the final message should present the complete updated file content).



FILE WRITE

Create or update docs/endpoints-flow.md with the generated content. If the editor supports it, insert/update the file directly. If not, print the entire Markdown so I can copy-paste.

IMPORTANT

Be precise; prefer reading constants and config files (YAML/TOML/ENV) to resolve dynamic values (e.g., base URLs, queue names, feature flags).

Note framework-specific features (e.g., Spring AOP interceptors, Express middlewares, FastAPI dependencies) that alter the flow.

Prefer exact names and code-linked evidence over guesses. When uncertain, explicitly mark as “unknown” and show the reference.


---

#### Optional framework-specific booster lines (append as needed)

- **Spring Boot (Java/Kotlin):**  
  “Also scan for: `@RestController`, `@Controller`, `@RequestMapping`, `@GetMapping`, `@PostMapping`, `HandlerMethodArgumentResolver`, `OncePerRequestFilter`, `HandlerInterceptor`, `WebMvcConfigurer`, `RestTemplate/WebClient/FeignClient` usage, `@Transactional`, `JdbcTemplate`, JPA repositories, Resilience4j annotations, and `OpenAPI` docs.”

- **Node/Express/NestJS:**  
  “Also scan for: `express.Router()`, `router.METHOD`, global/route middlewares, `axios/fetch/got` calls, TypeORM/Prisma/Mongoose ops, NestJS `@Controller` & `@Get/@Post` plus `Interceptors/Guards/Pipes`.”

- **Python FastAPI/Flask/Django:**  
  “Also scan for: FastAPI `APIRouter`, dependencies, Pydantic models, Flask blueprints, Django `urls.py` and `views`, `requests/httpx`, ORM calls.”

---

ROLE
You are a senior Spring Boot code-analysis assistant. Produce deterministic, evidence-linked results with file:line citations.

OBJECTIVE
For every HTTP endpoint, generate a detailed “Call Map” that enumerates ALL method calls (direct + transitive), with a short, practical description of what each method does and what it requires. Output one Markdown file per endpoint under docs/endpoints-calls/ and an index README.

WHY / FORMAT CHOICES (for usability)
- Use GitHub-flavored Markdown with <details> blocks for collapsible deep sections.
- Provide two complementary visuals:
  1) A Mermaid "flowchart" to show the high-level call graph (fan-out).
  2) A Mermaid "sequence" diagram to show actual execution order for the happy path.
- Add a "Methods Catalog" table listing each method once with signature, purpose, inputs, preconditions, side effects, errors, and citations.
- Make all method entries include source evidence: relative file path + line number.

SCOPE (SPRING TARGETS)
- Endpoints: @RestController/@Controller with @RequestMapping/@GetMapping/@PostMapping/@PutMapping/@PatchMapping/@DeleteMapping (class + method).
- Trace calls across:
  - Services/Components; @Transactional boundaries and AOP (@Aspect advice).
  - DB: Spring Data repositories, @Query (JPQL/SQL), JdbcTemplate/R2DBC/MyBatis.
  - Cache: @Cacheable/@CachePut/@CacheEvict.
  - Messaging: KafkaTemplate, RabbitTemplate, JmsTemplate (IBM MQ), Spring Cloud Stream.
  - External HTTP: RestTemplate/WebClient/FeignClient (collect base URLs from @Value/@ConfigurationProperties/application.yml).
  - Resilience4j: @Retry/@CircuitBreaker/@RateLimiter/@Bulkhead/@TimeLimiter.
  - Validation: Bean Validation (JSR-380) and custom validators.
  - Security: @PreAuthorize/@PostAuthorize and SecurityFilterChain rules that affect the endpoint.

DISCOVERY STEPS (DO IN ORDER)
1) Find endpoints and resolve full path + method + produces/consumes (class-level + method-level mapping).
2) For each endpoint handler, trace the call tree depth-first (limit to business-relevant paths; ignore trivial getters).
3) At each method, capture:
   - Signature: FQN#method(params) with param types.
   - Purpose: one-sentence plain-English summary from code/comments/names.
   - Requires (inputs & preconditions): non-null params, auth/roles, feature flags, DTO field expectations, validation annotations.
   - Side effects: DB writes, cache mutations, MQ publishes, external HTTP calls.
   - Throws/Errors: exceptions, error responses, retry/backoff, circuit breakers.
   - Transactional context: @Transactional or programmatic tx.
   - Evidence: file:line.
4) Extract config constants from application*.yml/properties and @ConfigurationProperties; if unresolved, show property key and where it’s defined.
5) Build diagrams from the traced graph (flowchart for structure; sequence for typical order).

OUTPUT RULES
Create or overwrite:
- docs/endpoints-calls/README.md   (index)
- docs/endpoints-calls/{slug}.md   (one per endpoint)

Slug rule: "{method_lower}_{path}" with "/" -> "_", ":" removed, "*" -> "star", path params kept as "{id}" (or "_param_" if needed).

PER-ENDPOINT FILE TEMPLATE (use exactly this structure)

# {METHOD} {PATH}
**Handler**: `{package.Class#method}`  
**File**: `relative/path/File.java:LINE`  
**Produces/Consumes**: `...`  
**Auth**: `{none | permitAll | JWT | OAuth2(scopes) | hasRole(...) | @PreAuthorize(...)}`

<details>
<summary><strong>High-Level Call Graph (Mermaid flowchart)</strong></summary>

```mermaid
flowchart TD
  C[Controller {Class#method}] --> S1[{ServiceA#op}]
  S1 -->|DB| R1[(RepositoryX#find...)]
  S1 -->|MQ| MQ1>JmsTemplate/KafkaTemplate: send ...]
  S1 -->|HTTP| X1[ExternalClient#call...]
  C --> S2[{ServiceB#op}]:::optional
  classDef optional stroke-dasharray: 3 3