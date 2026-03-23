# Vibe Coding Playbook — MVP Demo Sprint

> **Mindset shift**: You're not building a product. You're building a
> DEMO that proves a product should exist. Every decision filters through:
> "Does this make the demo better?" If no → skip it.

---

## THE 3-MINUTE DEMO MENTAL MODEL

Your management audience has a 3-minute attention window. Here's what they
need to see, feel, and remember:

```
MINUTE 1 — THE PROBLEM (you talking, tool gathering data)
  "When an alert fires, our SREs spend 30+ minutes switching between
  Splunk, Dynatrace, Confluence, and Jira to diagnose the issue."
  → MEANWHILE: tool is pulling real data from all 4 in parallel
  → Each tool lights up green on screen. "That just took 3 seconds."

MINUTE 2 — THE MAGIC (Claude generating the runbook)
  "Now the system synthesizes all that data and generates an executable
  diagnosis and remediation plan."
  → YAML runbook appears. Diagnosis summary in plain English.
  → "This is not a template. This was generated based on what
     it actually found in the logs and metrics just now."

MINUTE 3 — THE SAFETY (execution with approval)
  "Watch it execute. Each step runs against our real tools..."
  → Steps light up green one by one
  → Remediation step pauses: "AWAITING APPROVAL"
  → "Nothing writes to production without a human saying yes."
  → Click approve. Verification runs. ✅ RESOLVED.
  → "45 seconds. Diagnosis, plan, execution, verification."
```

### What Management Remembers After
1. It pulls REAL data from OUR tools (not a mock)
2. It generates the plan dynamically (not a template)
3. It won't break anything without approval (safe)
4. It's fast (seconds vs. 30+ minutes)

---

## COPILOT SESSION STRATEGY

### Session Setup (Do This Once)

Create a `.github/copilot-instructions.md` in your repo:

```markdown
# Project: SRE Agent Engine MVP

You are helping build an SRE incident automation tool.
The canonical spec is in MVP_DEMO_SPEC.md at the project root.
ALWAYS read the relevant section before implementing.

Key constraints:
- Python 3.12, FastAPI, async everything
- MCP Python SDK for tool connections
- Anthropic Claude SDK for LLM calls
- Frontend: single HTML file, no React, no build step
- This is a demo MVP — happy path only, no error recovery

When I ask you to implement a file:
1. Read the spec section for that file
2. Implement exactly as specified
3. Include type hints and docstrings
4. No placeholder comments — implement the full logic
```

### The Golden Rule: REAL DATA or NOTHING

The single biggest differentiator for your demo is that it uses REAL data
from your actual Splunk, Dynatrace, Confluence, and Jira instances. Every
time Copilot suggests a mock or placeholder, push back:

```
BAD: "I'll create some sample log data for testing"
GOOD: "Connect to Splunk and search for actual ERROR logs from cds-api-gw"

BAD: "Here's a hardcoded metrics response"
GOOD: "Pull live metrics from Dynatrace for the actual service"
```

Mock data is invisible to management. Real data makes them lean forward.

---

## DAY-BY-DAY PLAYBOOK

### DAY 1: The SID Gate (Most Important Day)

**Goal**: Prove you can connect to MCP servers from Python on your VDI.

```
Time allocation:
  30 min — SID passthrough test
  90 min — Client pool implementation
  60 min — First real tool call
  60 min — Second server + parallel calls

FIRST 30 MINUTES — DO NOT SKIP THIS:
─────────────────────────────────────
Prompt to Copilot:
  "I have an mcp.json from DevGPT/Cline that configures MCP servers
  for Splunk, Dynatrace, Confluence, and Jira. These use SID-based
  auth from my Windows VDI session.

  Write a minimal 30-line Python script that:
  1. Reads mcp.json
  2. Starts the Splunk MCP server process
  3. Creates a ClientSession
  4. Calls list_tools()
  5. Prints the available tools

  Use the official mcp Python SDK (from 'mcp' package)."

OUTCOME A — It works:
  Great. SID passthrough confirmed. Proceed to full client pool.

OUTCOME B — Auth error:
  Prompt: "SID passthrough failed with error: [paste error].
  Modify the script to use an API token instead.
  The token is in the SPLUNK_API_TOKEN env var."

OUTCOME C — MCP server won't start:
  This is the hardest blocker. Options:
  1. Check if npx is available on your VDI
  2. Try running the MCP server command directly in terminal
  3. If the DevGPT MCP servers use a custom binary, find its path
  4. Last resort: skip MCP, use direct REST API calls instead
     Prompt: "MCP servers won't start on my VDI. Create a
     compatibility layer that uses direct REST API calls to
     Splunk/Dynatrace but wraps them in the same ToolResponse
     interface so the rest of the app doesn't know the difference."
```

**REST OF DAY 1:**
```
After SID works, implement step by step:

1. "Implement backend/config.py per the MVP spec."
   → Review, test: can you import settings?

2. "Implement backend/mcp/normalizer.py per the spec."
   → Review, test: normalize a dummy response

3. "Implement backend/mcp/client_pool.py per the spec.
   Here's what list_tools() returned from my Splunk server: [paste].
   Here's the actual mcp.json I'm using: [paste]."
   → Test: connect + call_tool works

4. "Now call the Splunk search tool with this query:
   index=app source=cds-api-gw level=ERROR | head 20
   Show me the normalized response."
   → THIS IS YOUR DAY 1 WIN: real Splunk data in your Python terminal.
```

### DAY 2: All Four Servers

```
Goal: Connect all 4 MCP servers, parallel calls working.

Session flow:
  1. Connect Dynatrace (same pattern as Splunk)
  2. Connect Confluence
  3. Connect Jira
  4. Implement call_tools_parallel
  5. Implement tool_registry.py

Test prompt:
  "Call all four MCP servers in parallel:
  - Splunk: search for ERROR logs from cds-api-gw
  - Dynatrace: get_problems for cds-api-gw
  - Confluence: search for 'cds api gateway runbook'
  - Jira: search for recent issues on cds-api-gw
  Print each response with its latency."

WIN: All 4 return real data. Total time < 5 seconds.

IF A SERVER FAILS:
  Don't block on it. The demo works with 2-3 servers.
  A server that's down during demo is actually a good story:
  "Notice it gracefully handled the Confluence timeout and
  continued with the other sources."
```

### DAY 3: Data Models + Strategist Gathering

```
Goal: All models defined, strategist can gather intelligence.

Session flow:
  1. "Implement backend/models/schemas.py per the spec.
     All models in one file."
  2. "Implement the gather_intelligence() method in strategist.py.
     Here are the actual tool names from my MCP servers: [paste from Day 2].
     Make the queries realistic for a CDS API Gateway error scenario."
  3. Test: run gather_intelligence, print all tool outputs.

WIN: Gather returns structured ToolResponses from real servers.

IMPORTANT: Study the actual data coming back from your tools.
  - What do Splunk ERROR logs look like for your services?
  - What format does Dynatrace return metrics in?
  - What Confluence pages exist for your runbooks?
  This informs the system prompt on Day 4.
```

### DAY 4: The Strategist LLM Call (HIGHEST VALUE DAY)

```
Goal: Claude generates a sensible YAML runbook from real data.

THIS DAY DETERMINES WHETHER THE DEMO WORKS.

Session flow:

  MORNING — System Prompt:
  1. "Here's the strategist system prompt template from the spec.
     I need to customize it for my environment."

  Then ADD your real knowledge:
  - Actual service names and their dependencies
  - Actual SLO thresholds
  - Common failure modes YOU'VE seen in the last 6 months
  - Actual escalation policies
  - The actual tool names from your MCP servers

  AFTERNOON — The Planning Call:
  2. "Implement generate_runbook() in strategist.py.
     Use Claude's tool_use with GeneratedRunbook as the schema.
     Here's a real set of tool outputs from Day 3: [paste]."

  3. Run it. Read the generated YAML carefully.

  ITERATE ON THE PROMPT, NOT THE CODE:
  - If Claude hallucinates tool names → make the Available Tools section
    more explicit (copy exact tool names from registry)
  - If the diagnosis is shallow → add more domain knowledge to the prompt
  - If there are too many steps → add "Keep runbooks to 4-8 steps"
  - If remediation steps lack approval_required → the safety override
    in the code should catch this, but also add it to the prompt rules

  Run it 5 times with slightly different scenarios. Evaluate each.

WIN: Claude generates a YAML runbook that you'd actually follow as an SRE.
     This is the "holy shit" moment. If this works, the demo will work.
```

### DAY 5: Agents + Executor

```
Goal: Full pipeline working without UI (Python script).

Session flow:
  1. "Implement backend/engine/agents.py per the spec."
     Test: DiagnosticAgent calls Splunk, DecisionAgent calls Claude,
     RemediationAgent returns mock result.

  2. "Implement backend/engine/executor.py per the spec.
     Sequential execution, approval gate via asyncio.Event."

  3. Write a test_pipeline.py script:
     "Create a script that:
     1. Initializes pool, registry, strategist, executor
     2. Creates an IncidentContext for cds-api-gw CRITICAL
     3. Runs gather_intelligence
     4. Runs generate_runbook
     5. Prints the YAML
     6. Runs the executor (with stdin approval for remediation)
     7. Prints final status"

  Run it end-to-end. Fix any integration issues.

WIN: Full pipeline works from Python script. You can demo this in terminal.
     (Terminal demo is your backup if the UI isn't ready.)
```

### DAY 6: FastAPI + SSE Backend

```
Goal: Full pipeline accessible via HTTP with real-time event streaming.

Session flow:
  1. "Implement backend/main.py per the spec.
     Endpoints: /api/trigger, /api/stream/{id}, /api/approve/{id}, /api/status/{id}"

  2. Test with curl:
     Terminal 1: curl -N http://localhost:8000/api/stream/test
     Terminal 2: curl -X POST http://localhost:8000/api/trigger \
                   -H "Content-Type: application/json" \
                   -d '{"app_name":"cds-api-gw","environment":"prod","severity":"CRITICAL"}'
     → Watch events stream in Terminal 1
     Terminal 2: curl -X POST http://localhost:8000/api/approve/run_xxx \
                   -d '{"decision":"approved"}'

WIN: Full pipeline works via HTTP. Events stream in real-time.
```

### DAY 7: The Frontend (Visual Polish Day)

```
Goal: Beautiful single-page UI that makes the demo impressive.

THIS IS WHERE THE DEMO IS WON OR LOST VISUALLY.

Prompt to Copilot:
  "Create frontend/index.html — a single-file web UI for the SRE Agent Engine.

  Tech: Tailwind CSS (CDN), Alpine.js (CDN) for reactivity, SSE for live events.
  NO React. NO build step. Just one HTML file.

  Theme: Dark, professional, terminal-inspired. Think SRE war-room dashboard.

  Layout:
    HEADER: App name + status indicator (pulsing dot: green/yellow/red)

    LEFT PANEL (30%):
      - Trigger button: 'Run Diagnosis' with app name + env dropdowns
      - Intelligence gathering progress (4 tools, each lights up when done)
      - Run status badge

    CENTER PANEL (40%):
      - Execution trace: step-by-step list
      - Each step: icon (🔍/🧠/⚠️/✅) + description + status badge
      - Steps animate in as they start
      - Currently-running step has a pulse animation
      - Completed steps have green checkmark
      - The approval step shows an APPROVE / REJECT button pair

    RIGHT PANEL (30%):
      - Tab 1: Diagnosis Summary (plain English)
      - Tab 2: Generated Runbook (YAML in a <pre> block with syntax highlighting)

    APPROVAL MODAL (overlay):
      - Shows when remediation needs approval
      - Displays: proposed action, diagnosis context, risk level, rollback plan
      - Big green APPROVE and red REJECT buttons

  SSE Integration:
    - Connect to /api/stream/{run_id} on trigger
    - Handle events: tool_started, tool_completed, status_change,
      planning_started, planning_completed, step_started, step_completed,
      approval_requested, run_completed, error

  Animations:
    - Steps fade in with a slide-up animation
    - Tool indicators pulse while loading, solid green when done
    - Status transitions have smooth color changes
    - The YAML runbook types out character by character (optional, nice touch)

  CRITICAL: This must look polished enough for a VP-level audience.
  No 'lorem ipsum'. No ugly default styling. Think Bloomberg terminal meets
  modern SaaS dashboard."

After Copilot generates it:
  1. Open in browser: http://localhost:8000/
  2. Click Trigger. Watch the demo flow.
  3. Adjust timing, colors, animations.
  4. The approval modal MUST be prominent and clear.

POLISH CHECKLIST:
  [ ] Status indicator pulses during active run
  [ ] Tools light up one-by-one (not all at once)
  [ ] YAML appears after planning completes (not before)
  [ ] Steps animate in as they start
  [ ] Approval modal has clear context and big buttons
  [ ] "RESOLVED" end state feels satisfying (maybe a subtle confetti?)
  [ ] Responsive enough for your screen resolution
  [ ] Font sizes readable from 6 feet away (presenting on screen)
```

### DAY 8: Rehearsal (DO NOT SKIP)

```
Goal: Demo runs perfectly 5 times in a row. You have a backup plan.

MORNING — Reliability:
  1. Run the full demo 5 times.
  2. Time each run. Target: 45-60 seconds of tool execution.
  3. Note any flakiness (MCP timeouts, slow responses).
  4. For flaky tools: add a longer timeout or adjust the query.

AFTERNOON — Backup Plans:
  1. Record a screen capture video of the demo working perfectly.
     → If live demo fails, play this video.
  2. Prepare a "degraded demo" script:
     → "If Dynatrace is slow, I'll show the Splunk + Jira data
        and mention 'normally Dynatrace responds in 1 second'"
  3. Pre-cache one successful run's YAML output.
     → If Claude returns something weird, you can show the cached version.

EVENING — Talking Points:
  Practice the narration for each phase:

  TRIGGER: "I'm simulating a CRITICAL alert on our CDS API Gateway.
  In production, this would be triggered by a Dynatrace webhook."

  GATHERING: "Watch — it's pulling real data from our Splunk, Dynatrace,
  Confluence, and Jira. These are our actual production tools, not mocks."

  PLANNING: "Now Claude analyzes everything and generates an executable
  runbook. This isn't a template — it's dynamically created based on
  what it found."

  EXECUTING: "Each step executes against our real tools. Watch the
  diagnostic steps run..."

  APPROVAL: "Here's the critical part — it proposes a remediation but
  PAUSES for human approval. We see the full context: what it found,
  what it wants to do, and the rollback plan. Nothing touches production
  without a human saying yes."

  APPROVE: [click] "And now it verifies the fix worked."

  RESOLVE: "45 seconds. From alert to resolution. What currently takes
  our SREs 30+ minutes of switching between 4 tools."

  PITCH: "This is L2 on our maturity model. [Teammate]'s L1 tool proved
  that LLMs can diagnose our incidents. This takes it further — it
  generates the plan AND executes it, with human approval gates.
  The data you just saw was real. The runbook was real. The only thing
  that was mocked was the actual K8s remediation call — which we gate
  behind approval in production anyway."
```

---

## FALLBACK STRATEGIES

### If MCP doesn't work at all on VDI:
```
Use direct REST APIs instead. Create a compatibility wrapper:

Prompt: "Create a DirectApiPool class that has the same interface as
McpClientPool but uses httpx to call REST APIs directly:
- Splunk: POST to /services/search/jobs with the search query
- Dynatrace: GET /api/v2/problems and /api/v2/metrics/query
- Confluence: GET /rest/api/content/search
- Jira: GET /rest/api/2/search

The call_tool and call_tools_parallel signatures stay the same.
The Strategist and Executor don't know the difference."
```

### If Claude generates bad runbooks:
```
Two fixes:
1. Iterate on the system prompt (80% of the time, this is the issue)
2. Provide a few-shot example in the system prompt:
   "Here's an example of a good runbook for reference: [paste a hand-crafted one]"
```

### If the demo is too slow (>90 seconds):
```
1. Reduce intelligence gathering calls (2 tools instead of 4)
2. Pre-warm the MCP connections (connect on app start, not on trigger)
3. Use a smaller context window (reduce max_context_tokens)
4. For the demo, hardcode the Splunk query to return fewer results
```

### If live demo crashes:
```
Play the backup video. Say: "Let me show you the recording from this
morning's test run — we had a network hiccup just now."
Nobody will judge you for having a backup. They WILL judge you for
scrambling live.
```

---

## AFTER THE DEMO SUCCEEDS

When management says "what's next?", have this ready:

```
"What you saw today is the MVP — happy path, one scenario, demo mode.
Here's the roadmap to production:

NEXT 2 WEEKS:
  - DAG-based parallel execution (faster for complex runbooks)
  - Full React dashboard with run history
  - Integration with [teammate]'s L1 tool as an input signal
  - Connect to real K8s for remediation (with approval gates)

NEXT MONTH:
  - Crash recovery (resume interrupted runs)
  - Audit logging for compliance
  - Confidence-based auto-approval for well-known patterns
  - Slack integration for approval flow

NEXT QUARTER:
  - Multi-service support (beyond CDS gateway)
  - Runbook learning loop (use resolved incidents to improve future plans)
  - SLO-aware prioritization
  - Pilot on production with shadow mode"
```

This turns "cool demo" into "funded initiative."
