You are an expert software architect and SRE assistant with full read-only access to this GitHub repository (<REPO_NAME>). 

Your job is to produce a HIGH-LEVEL but HOLISTIC map of this codebase that I can use for:
- Understanding core responsibilities and architecture
- Debugging production issues
- Designing reliability and observability improvements

Follow ALL of these rules STRICTLY:

1. SAFETY / REDACTION
   - Do NOT output any secrets, credentials, API keys, tokens, certificates, private hostnames, IPs, account numbers, emails, or any other PII.
   - If you encounter anything that looks like a secret, hostname, or internal identifier, REDACT it as `<REDACTED>` or describe it generically (e.g. “internal CM endpoint”, “Pithos object-store host”).
   - Do NOT print full environment variable values. You may mention env var NAMES, but not their real values.

2. EVIDENCE & HONESTY
   - Base your description on what is ACTUALLY present in the code, configs, and docs in this repo.
   - Clearly distinguish between:
     - “Explicit in code” → say: **[explicit]**
     - “Inferred from patterns / naming / structure” → say: **[inferred]**
   - When possible, cite the most important file paths you used, in backticks, so I can jump to them.

3. SCOPE & STRUCTURE OF OUTPUT
   Produce your answer in the following sections:

   ## 1. Repository Overview
   - In 5–10 sentences, explain what this repo does in business terms and technical terms.
   - Mention the main domains (e.g. document store, search, distribution, integrations, etc.).
   - Identify the primary runtime environments (e.g. Spring Boot service, Python app, ECS, Kubernetes, etc.). Mark each statement as **[explicit]** or **[inferred]**.

   ## 2. Main Components & Modules
   - List the main services / modules / packages in this repo and their responsibilities.
   - For each component, specify:
     - What it does
     - Key entrypoints (controllers/handlers/jobs)
     - Key dependencies (e.g. CM client, Pithos client, queues, DB client)
   - For each component, list 2–5 important files with their paths (no code, just path + short description).

   ## 3. Core Flows / Use Cases
   - For this repo, describe the core flows it implements. Focus on:
     - For amcds: 
       - Upload, update, and download flows (CDS-STORE)
       - Search flows (CDS-FIND)
       - Any routing logic between different backends (e.g. CM vs Pithos, IPB vs PBA, GUDI vs non-GUDI).
     - For cds-distribute:
       - How distribution is triggered
       - How distribution rules are determined
       - How physical distribution or downstream systems are called.
   - For each flow, describe:
     - The typical HTTP/API entrypoint
     - The sequence of major steps (validation → routing → storage/distribution → response)
     - The main modules/classes involved.
   - Represent each core flow in a short textual sequence (e.g. `Controller -> Service -> Adapter -> Client -> External system`) and reference key files.

   ## 4. Configuration, Feature Flags & Environment
   - Summarize where configuration is managed (e.g. `application.yml`, `*.properties`, `config/`, feature flag libraries).
   - Identify:
     - How environments (dev/qa/prod) are separated (by files, profiles, env vars, etc.).
     - Any feature flags or config switches that affect:
       - Routing (e.g. CM vs Pithos, IPB vs PBA).
       - Behaviour toggles (sync vs async, retries, timeouts).
   - Do NOT output actual values. Only mention:
     - Config/prop file paths
     - Env var / key NAMES

   ## 5. External Integrations
   - List the main external systems this repo talks to (e.g. CM, Pithos, S3, queues, auth/identity, distribution vendors).
   - For each integration:
     - Describe its purpose
     - Identify the client/adapter modules used
     - Mention where timeouts/retries/circuit breakers (if any) are configured.
   - Redact any sensitive endpoints or identifiers.

   ## 6. Observability & Error Handling
   - Describe how logging is done (frameworks, typical log pattern).
   - Describe how metrics and tracing are integrated (e.g. Prometheus metrics, Dynatrace, OpenTelemetry spans, request attributes for consumer FIDs, etc.).
   - Identify:
     - Where errors are mapped to HTTP responses
     - Any global exception handlers or filters.

   ## 7. Reliability & Risk Hotspots (Opinionated)
   - Based on the code structure, list 5–10 potential reliability / operability hotspots, such as:
     - Tight coupling to a specific backend
     - Weak error handling around external calls
     - Lack of timeouts/retries
     - Complex routing logic spread across multiple layers
   - Mark these clearly as **[inferred]** and tie each to specific files or patterns.

   ## 8. Quick Navigation Guide
   - Finish with a cheat sheet:
     - “Start here for upload flow” → `path/to/file`
     - “Start here for search flow” → `path/to/file`
     - “Start here for distribution flow” → `path/to/file`
     - “Start here for routing logic between CM/Pithos” → `path/to/file`
     - “Start here for external integrations config” → `path/to/file`
     - “Start here for tracing/metrics” → `path/to/file`

4. STYLE
   - Be concise but not superficial.
   - Prefer bullet points and short paragraphs.
   - Assume the reader is an SRE/architect who cares about flows, dependencies, and failure modes more than low-level syntax.

Now, scan the entire repository and generate this structured overview.


## Prompt 1: Get all controllers in the repo

Use this in `amcds` first, then `cds-distribute`.

```text
You are an expert Java/Spring codebase navigator with full read-only access to this repository.

Task:
Scan the ENTIRE repository and find all Spring MVC controllers.

What to include:
- Classes annotated with @RestController or @Controller.
- For each class, give:
  - fully-qualified class name
  - file path (relative to repo root)
  - annotation used (@RestController or @Controller)
  - a one-line description of what it seems to handle (based on class name and package only).

STRICT RULES:
- Do NOT output any secrets, credentials, sample data, hostnames, or environment-specific values.
- Only output identifiers and file paths that are directly in the code.
- If you are unsure about a description, keep it generic.

Output format:
Return a markdown table with columns:
`component_type | class_fqn | file_path | annotation | short_description`

Where:
- component_type must be exactly `controller`.
- class_fqn is the fully-qualified class name.
- file_path is the relative path like `src/main/java/.../DocumentController.java`.

Now scan the repository and produce this table.
```

## Prompt 2: Extract endpoints for all controllers


```text
Using ONLY the controllers from the previous result in this repository:

For each controller class:
- Inspect all handler methods that are annotated with any of:
  - @GetMapping, @PostMapping, @PutMapping, @DeleteMapping, @PatchMapping, or @RequestMapping.
- For each handler method, extract:
  - controller class FQN
  - method name
  - HTTP method(s)
  - path pattern(s)
  - file path (relative)
  - any obvious “operation name” if present (e.g. from @Operation(summary=...) or JavaDoc) – optional.

IMPORTANT:
- If multiple HTTP methods or paths are mapped, create one row per combination.
- If the HTTP method is inferred from @RequestMapping without method=, set http_method to `UNKNOWN`.
- Do NOT include request/response body types, parameter names, or example values.
- Do NOT output any secrets, sample data, or hostnames. Only identifiers and static path patterns.

Output format:
Return a CSV-style table inside a code block with columns:

controller_class_fqn,http_method,path_pattern,method_name,file_path,notes

Where:
- http_method is GET/POST/PUT/DELETE/PATCH/UNKNOWN.
- notes is an optional very short description derived from the method name (e.g. "upload document", "search documents").

Only include endpoints actually defined in the code. Do not guess.
```

## Prompt 3: Get all services


```text
Scan the ENTIRE repository for Spring service classes.

What to include:
- Classes annotated with @Service or @Component that clearly act as business logic or integration layers (e.g. *Service, *Manager, *Adapter, *Client).

For each such class, output:
- fully-qualified class name
- file path
- stereotype: `SERVICE` or `ADAPTER` or `CLIENT` (best guess from naming/package)

Strict rules:
- Do NOT output any secrets, credentials, hostnames, or environment values.
- Only output identifiers and file paths.

Output format:
Return a markdown table:

component_type | class_fqn | file_path | stereotype | short_description

Where:
- component_type must be exactly `service_class`.
- short_description is a 1-line guess derived from name/package (e.g. "wraps Content Manager operations" or "Pithos storage adapter").
```

## Prompt 4: Extract public service methods

```text
From the list of service classes you just provided:

For each class:
- Inspect all PUBLIC methods that are part of the business logic (exclude constructors, getters/setters, equals/hashCode/toString, and trivial one-line delegates if obvious).
- For each method, extract:
  - class FQN
  - method name
  - file path
  - a 3–8 word description of what the method appears to do (based only on name and signature).

Rules:
- Do NOT output any parameter names or types that look like PII or secrets (e.g. accountId, ssn, token). If in doubt, omit the parameter list entirely.
- Do NOT include example values or literals from the code.
- Only output identifiers and file paths.

Output format:
Return a CSV-style table inside a code block:

class_fqn,method_name,file_path,short_description

Keep short_description purely functional (e.g. "store document to CM", "distribute document to vendor").
```

You are an expert backend engineer and SRE reviewing this codebase for **incorrect or inconsistent HTTP status code usage**, especially around REST semantics and error handling.

I have an inventory file exported from a previous analysis. It contains the complete list of classes to inspect inside the <module> component of this repository.

===========================================
INVENTORY INPUT
===========================================
- File: <INVENTORY_FILE_PATH> (CSV or XLSX opened as text)
- Columns include:
  - `Class Name`
  - `File Path` (absolute or relative path to the source file)
- This inventory file is the **authoritative list** of classes that must be analyzed.

Your job is to use this inventory to drive a **systematic audit** of all listed <controllers> and <services> inside <module>.

===========================================
ANOMALY DEFINITIONS (Agnostic to domain)
===========================================
Flag any place where HTTP responses violate typical REST semantics, including but not limited to:

1. **Incorrect handling of empty results**
   - Example: <flow> endpoints returning `404` when the response is logically “0 items found.”
   - Expected: `200` with an empty list or object, unless the endpoint explicitly represents a single resource lookup.

2. **Predictable validation errors incorrectly mapped**
   - Validation or schema violations mapped to `5xx` instead of `4xx`.

3. **Business-rule or state-machine failures incorrectly mapped**
   - Invalid state transitions, forbidden operations, quota issues, etc., returned as `5xx` instead of `400/403/409/422`.

4. **2xx responses used incorrectly**
   - Endpoint returns `200` or `201` while embedding an error payload.
   - Endpoint hides internal errors and still emits `2xx`.

5. **Inconsistent status codes across <flow>**
   - Same scenario handled differently in different methods/classes.

6. **Exposing internal exceptions**
   - Returning stack traces/internal messages via raw HTTP responses instead of sanitized error objects.

Treat all framework styles (Spring, JAX-RS, Node, custom frameworks) identically unless the code shows specific conventions.

===========================================
CHECKPOINT / LOG FILE
===========================================
Maintain a progress log so this process can resume if interrupted.

- Log file path: `docs/<module>-http-status-audit.log.jsonl`
- For each processed class, append **one JSON line** in this shape:

  {
    "file": "<file-path-from-inventory>",
    "class": "<Class Name>",
    "status": "done" | "skipped" | "error",
    "anomalies_found": <integer>,
    "notes": "<optional reason or comment>",
    "timestamp": "<ISO-8601>"
  }

Before analyzing, conceptually check if the log exists:
- If a file already has `"status": "done"`, skip re-processing it.
- Only analyze classes not logged yet or logged with `"error"`.

===========================================
OUTPUT FORMAT
===========================================

At the end of the audit run, generate two sections:

### 1. Summary of Observed Anti-Patterns (Markdown)
A short list capturing recurring issues across the <module>.

### 2. Consolidated Anomalies Table (Markdown)
Include **only anomalies discovered in this run**.

Columns:
- File  
- Class  
- Endpoint / Method (HTTP method + route, if detectable)  
- Current Status Code  
- Trigger Condition  
- Why It Appears Anomalous  
- Suggested Behavior / Status Code  
- Confidence (High / Medium / Low)

===========================================
ANALYSIS RULES
===========================================

Use the following baseline REST rules when deciding anomalies:

- **2xx** for successful operations  
  - Particularly, <flow> endpoints returning 0 results should use `200` with an empty array/object unless it is a single-resource lookup.

- **4xx** for anything caused by the caller:  
  - Invalid fields, missing data, malformed inputs, wrong state, forbidden operations, quota exceeded.

- **5xx** only for unexpected server-side failures:  
  - Timeouts, infra errors, unhandled exceptions, dependency outages.

If a situation is ambiguous, include it anyway and mark `Confidence = Low`.

===========================================
EXECUTION RULES
===========================================

1. Parse and iterate through the inventory file in order.  
2. For each entry not marked `"done"` in the log:  
   - Analyze the class for any HTTP status assignment or response builder usage.  
   - Inspect exception mappers, handlers, and utility classes referenced inside.  
   - Collect anomalies.  
   - Append a JSONL entry to the log.  

3. Once all reachable classes for this run are processed, output:
   - Summary (Markdown)
   - Consolidated anomalies table (Markdown)

Do not skip ambiguous cases; just mark them low confidence.

===========================================
BEGIN
===========================================

Start by reading the inventory file content provided in the editor and identifying which files need to be processed for this run.



