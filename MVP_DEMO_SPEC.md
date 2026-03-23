# SRE Agent Engine — MVP Demo Spec

> **Purpose**: Get a working, visually impressive demo in front of management
> within ~8 working days. This spec is RUTHLESSLY scoped to only what the
> demo needs. Everything else is deferred.
>
> **The demo pitch**: "Watch an alert fire. In 10 seconds, the system pulls
> data from 4 tools, generates a diagnosis and executable plan, and walks
> through remediation — pausing for human approval before any action."
>
> **Feed this file to GitHub Copilot (Claude Opus 4.6 / Sonnet) as context.**

---

## 0. WHAT WE'RE BUILDING (AND WHAT WE'RE NOT)

### In Scope (Demo MVP)
```
✅ MCP connectivity to Splunk, Dynatrace, Confluence, Jira (via mcp.json / SID)
✅ Strategist: parallel data gathering → Claude → dynamic YAML runbook
✅ Sequential executor: runs steps one-by-one (no DAG, no parallel groups)
✅ Approval gate: web-based approve/reject before remediation
✅ Single-page web UI: live execution stream + generated runbook viewer
✅ ONE golden-path demo scenario (rehearsed, polished)
```

### NOT in Scope (Deferred)
```
❌ DAG execution / parallel groups — sequential is fine for demo
❌ Crash recovery / state persistence — if it crashes, restart
❌ React / Vite / build toolchain — single HTML file served by FastAPI
❌ Docker Compose — runs locally on VDI
❌ MCP server registry CRUD — hardcoded in mcp.json
❌ Run history / search / pagination — one run at a time
❌ Abstractive summarization — extractive only
❌ Retry logic / error recovery — happy path only
❌ Audit logging — not needed for demo
❌ Cost tracking — mention verbally
❌ Write operations to real infra — mock the K8s remediation
```

### The Demo Scenario (Script This Exactly)
```
SCENARIO: "CDS API Gateway error rate spike"

1. Presenter clicks "Trigger Incident" button
   → app_name: cds-api-gw, environment: prod, severity: CRITICAL

2. UI shows "GATHERING INTELLIGENCE..." with live progress
   → Splunk: searches for ERROR logs (REAL data)
   → Dynatrace: pulls error rate + latency metrics (REAL data)
   → Confluence: searches for related runbooks (REAL data)
   → Jira: checks recent deployments/changes (REAL data)
   Each tool lights up green as it completes (~3 seconds total)

3. UI shows "GENERATING RUNBOOK..." with spinner
   → Claude Sonnet processes all signals (~4 seconds)
   → Generated YAML runbook appears in a code viewer
   → Diagnosis summary shown in plain English

4. UI shows "EXECUTING RUNBOOK..." with step-by-step trace
   → Step 1: Deep log analysis (DiagnosticAgent → Splunk) ✅
   → Step 2: Deployment correlation (DiagnosticAgent → Jira) ✅
   → Step 3: Root cause decision (DecisionAgent → Claude) ✅
   → Step 4: Remediation proposed ⚠️ AWAITING APPROVAL

5. Approval modal appears with full context:
   "Proposed action: Rollback deployment build-4891"
   "Root cause: OOM from memory leak (confidence: 0.84)"
   "Rollback plan: Re-deploy build-4891 if regression"
   → Presenter clicks APPROVE

6. UI shows remediation executing (MOCKED — no real K8s call)
   → Step 5: Verification check (DiagnosticAgent → Dynatrace) ✅
   → Status: RESOLVED

Total demo time: ~45 seconds of execution + 30 seconds of presenter talking
```

---

## 1. ENVIRONMENT & AUTH

```
AVAILABLE (via mcp.json from DevGPT/Cline):
  - Splunk MCP       → search, get_alerts
  - Dynatrace MCP    → get_problems, get_metrics, get_events
  - Confluence MCP    → search_pages, get_page
  - Jira MCP          → search_issues, get_issue

ALSO AVAILABLE (direct API — use if MCP is flaky for any tool):
  - Confluence REST API
  - ServiceNow REST API (read-only)
  - Splunk REST API

AUTH:
  - Primary: SID passthrough from VDI session via mcp.json
  - Fallback: API tokens in .env

STACK:
  - Python 3.12, FastAPI, uvicorn
  - Anthropic Claude SDK (Sonnet 4 for planning + decisions)
  - MCP Python SDK (official)
  - SQLite (minimal — just current run state)
  - Frontend: Single HTML file with Tailwind CDN + SSE (NO React, NO build step)
```

### mcp.json (adapt from your DevGPT/Cline config)
```jsonc
{
  "mcpServers": {
    "splunk": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-splunk"],
      "env": { "SPLUNK_URL": "${SPLUNK_BASE_URL}", "SPLUNK_AUTH": "sid" }
    },
    "dynatrace": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-dynatrace"],
      "env": { "DT_URL": "${DYNATRACE_BASE_URL}", "DT_AUTH": "sid" }
    },
    "confluence": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-confluence"],
      "env": { "CONFLUENCE_URL": "${CONFLUENCE_BASE_URL}", "CONFLUENCE_AUTH": "sid" }
    },
    "jira": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-jira"],
      "env": { "JIRA_URL": "${JIRA_BASE_URL}", "JIRA_AUTH": "sid" }
    }
  }
}
```

---

## 2. PROJECT STRUCTURE (Minimal)

```
sre-agent-engine/
├── backend/
│   ├── __init__.py
│   ├── main.py                    # FastAPI app + SSE endpoint + static serving
│   ├── config.py                  # Pydantic Settings
│   │
│   ├── mcp/
│   │   ├── __init__.py
│   │   ├── client_pool.py         # MCP connection manager
│   │   ├── tool_registry.py       # Tool discovery + read/write tags
│   │   └── normalizer.py          # Standardize responses
│   │
│   ├── engine/
│   │   ├── __init__.py
│   │   ├── strategist.py          # Gather intelligence + generate runbook
│   │   ├── executor.py            # Sequential step execution (NO DAG)
│   │   └── agents.py              # DiagnosticAgent, DecisionAgent, RemediationAgent
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   ├── schemas.py             # ALL models in ONE file (MVP simplicity)
│   │   └── db.py                  # Minimal SQLite (just current run)
│   │
│   └── prompts/
│       └── strategist_system.md   # Domain expertise prompt
│
├── frontend/
│   └── index.html                 # SINGLE FILE — Tailwind CDN + SSE + Alpine.js
│
├── mcp.json                       # From DevGPT/Cline
├── .env                           # API keys + server URLs
├── pyproject.toml
└── README.md
```

**Key simplification**: No separate api/ directory. All endpoints live in `main.py`.
No run history. No WebSocket (use SSE — simpler, one-direction is enough for demo).
All models in one file. Frontend is ONE HTML file with Tailwind CDN and Alpine.js
for reactivity — zero build step.

---

## 3. ALL MODELS IN ONE FILE (backend/models/schemas.py)

```python
"""
All data models for the MVP. One file. No over-engineering.
"""
from pydantic import BaseModel, Field
from typing import Any, Literal
from datetime import datetime
from enum import Enum
import uuid, yaml

# ── Tool Permission ──
class ToolPermission(str, Enum):
    READ = "read"
    WRITE = "write"

WRITE_TOOLS: set[str] = {
    "restart_deployment", "scale_replicas", "rollback_deployment",
    "create_issue", "update_issue", "transition_issue",
    "create_page", "update_page", "create_ticket", "update_ticket",
    "page_oncall", "post_to_channel",
}

# ── MCP ──
class ToolSchema(BaseModel):
    name: str
    server: str
    description: str
    parameters: dict[str, Any] = {}
    permission: ToolPermission = ToolPermission.READ

class ToolResponse(BaseModel):
    tool: str
    server: str
    status: Literal["success", "error", "timeout"]
    data: Any = None
    error_message: str | None = None
    latency_ms: int = 0
    token_estimate: int = 0

# ── Incident ──
class IncidentContext(BaseModel):
    app_name: str
    environment: str
    severity: str = "CRITICAL"
    alert_source: str | None = None
    description: str | None = None
    l1_diagnosis: str | None = None        # from teammate's L1 tool

# ── Generated Runbook ──
class RunbookStepTool(BaseModel):
    server: str
    method: str
    params: dict[str, Any] = {}

class RunbookStep(BaseModel):
    id: str
    type: Literal["diagnostic", "decision", "remediation", "close", "escalate"]
    description: str
    tool: RunbookStepTool | None = None
    llm_prompt: str | None = None
    output_schema: dict[str, str] | None = None
    branches: dict[str, str] = {}
    approval_required: bool = False
    rollback_plan: str | None = None
    risk: str | None = None
    on_failure: str | None = None
    success_criteria: dict[str, Any] = {}
    wait_before: str | None = None
    timeout: str = "30s"

class GeneratedRunbook(BaseModel):
    diagnosis_summary: str
    steps: list[RunbookStep]
    confidence: float = 0.0
    generated_at: datetime = Field(default_factory=datetime.utcnow)

    def to_yaml(self) -> str:
        return yaml.dump(
            self.model_dump(mode="json", exclude_none=True),
            default_flow_style=False, sort_keys=False
        )

# ── Run State ──
class RunStatus(str, Enum):
    GATHERING = "gathering"
    PLANNING = "planning"
    EXECUTING = "executing"
    AWAITING_APPROVAL = "awaiting_approval"
    RESOLVED = "resolved"
    ESCALATED = "escalated"
    FAILED = "failed"

class StepStatus(str, Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    AWAITING_APPROVAL = "awaiting_approval"
    APPROVED = "approved"
    SKIPPED = "skipped"

class StepResult(BaseModel):
    step_id: str
    status: StepStatus
    description: str = ""
    output: Any = None
    error: str | None = None
    latency_ms: int = 0

class RunState(BaseModel):
    """The single current run. No history needed for MVP."""
    id: str = Field(default_factory=lambda: f"run_{uuid.uuid4().hex[:8]}")
    incident: IncidentContext
    status: RunStatus = RunStatus.GATHERING
    runbook_yaml: str | None = None
    diagnosis_summary: str | None = None
    step_results: list[StepResult] = []
    current_step_index: int = 0
    pending_approval_step: str | None = None
    created_at: datetime = Field(default_factory=datetime.utcnow)
```

---

## 4. CONFIG (backend/config.py)

```python
from pydantic_settings import BaseSettings
from typing import Optional

class Settings(BaseSettings):
    anthropic_api_key: str
    model: str = "claude-sonnet-4-20250514"

    # Token budget for strategist context
    max_context_tokens: int = 12000

    # MCP
    mcp_config_path: str = "mcp.json"
    mcp_call_timeout: int = 30

    # Auth fallbacks (if SID passthrough fails)
    splunk_api_token: Optional[str] = None
    dynatrace_api_token: Optional[str] = None
    confluence_api_token: Optional[str] = None
    jira_api_token: Optional[str] = None

    # Demo mode: mock remediation instead of calling real write tools
    demo_mode: bool = True

    class Config:
        env_file = ".env"

settings = Settings()
```

---

## 5. MCP LAYER (backend/mcp/)

### 5a. Client Pool (backend/mcp/client_pool.py)

```python
"""
MCP connection manager. Starts server processes from mcp.json,
maintains client sessions, handles tool calls.

MVP SIMPLIFICATION:
- No reconnection logic (restart app if connection drops)
- No health checks (assume servers are up for demo)
- SID passthrough is the primary auth path
"""
import json, asyncio, time
from pathlib import Path
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from backend.config import settings
from backend.mcp.normalizer import normalize_response
from backend.models.schemas import ToolResponse

class McpClientPool:

    def __init__(self, config_path: str = None):
        self.config_path = config_path or settings.mcp_config_path
        self.configs: dict = {}
        self.sessions: dict[str, ClientSession] = {}
        self._read_streams = {}
        self._write_streams = {}

    async def load_config(self):
        """Read mcp.json and parse server configs."""
        raw = json.loads(Path(self.config_path).read_text())
        self.configs = raw.get("mcpServers", {})

    async def connect(self, server_name: str):
        """Start one MCP server process and create a client session."""
        cfg = self.configs[server_name]
        server_params = StdioServerParameters(
            command=cfg["command"],
            args=cfg.get("args", []),
            env=cfg.get("env", {})
        )
        # stdio_client returns (read_stream, write_stream)
        # We need to keep these alive, so store the context
        read, write = await stdio_client(server_params).__aenter__()
        session = ClientSession(read, write)
        await session.initialize()
        self.sessions[server_name] = session
        self._read_streams[server_name] = read
        self._write_streams[server_name] = write

    async def connect_all(self):
        """Connect to all servers. Skip failures (log them, don't crash)."""
        await self.load_config()
        for name in self.configs:
            try:
                await self.connect(name)
                print(f"  ✓ Connected to {name}")
            except Exception as e:
                print(f"  ✗ Failed to connect to {name}: {e}")

    async def call_tool(self, server: str, tool: str, params: dict = None) -> ToolResponse:
        """Call a tool on an MCP server. Returns normalized ToolResponse."""
        if server not in self.sessions:
            return ToolResponse(
                tool=tool, server=server, status="error",
                error_message=f"Server '{server}' not connected"
            )
        session = self.sessions[server]
        start = time.monotonic()
        try:
            result = await asyncio.wait_for(
                session.call_tool(tool, params or {}),
                timeout=settings.mcp_call_timeout
            )
            latency = int((time.monotonic() - start) * 1000)
            return normalize_response(server, tool, result, latency)
        except asyncio.TimeoutError:
            return ToolResponse(
                tool=tool, server=server, status="timeout",
                error_message=f"Timeout after {settings.mcp_call_timeout}s",
                latency_ms=settings.mcp_call_timeout * 1000
            )
        except Exception as e:
            return ToolResponse(
                tool=tool, server=server, status="error",
                error_message=str(e)
            )

    async def call_tools_parallel(self, calls: list[dict]) -> list[ToolResponse]:
        """Fire multiple tool calls concurrently."""
        tasks = [
            self.call_tool(c["server"], c["tool"], c.get("params"))
            for c in calls
        ]
        return await asyncio.gather(*tasks)

    async def list_tools(self, server: str) -> list[dict]:
        """Discover tools available on a server."""
        if server not in self.sessions:
            return []
        result = await self.sessions[server].list_tools()
        return [{"name": t.name, "description": t.description,
                 "parameters": t.inputSchema} for t in result.tools]
```

### 5b. Tool Registry (backend/mcp/tool_registry.py)

```python
"""
Discovers tools from connected MCP servers.
Classifies each as read/write.
Formats tool list for injection into Claude's prompt.
"""
from backend.models.schemas import ToolSchema, ToolPermission, WRITE_TOOLS

class ToolRegistry:

    def __init__(self, pool):
        self.pool = pool
        self.tools: dict[str, list[ToolSchema]] = {}

    async def discover_all(self):
        """Discover tools from all connected servers."""
        for server_name in self.pool.sessions:
            raw_tools = await self.pool.list_tools(server_name)
            self.tools[server_name] = [
                ToolSchema(
                    name=t["name"],
                    server=server_name,
                    description=t.get("description", ""),
                    parameters=t.get("parameters", {}),
                    permission=(
                        ToolPermission.WRITE if t["name"] in WRITE_TOOLS
                        else ToolPermission.READ
                    )
                )
                for t in raw_tools
            ]

    def is_write_tool(self, server: str, tool: str) -> bool:
        return tool in WRITE_TOOLS

    def get_tools_for_prompt(self) -> str:
        """Format all tools for Claude's system prompt."""
        lines = ["## Available MCP Tools\n"]
        for server, tools in self.tools.items():
            lines.append(f"### {server}")
            for t in tools:
                perm = "🔒 WRITE" if t.permission == ToolPermission.WRITE else "READ"
                lines.append(f"- **{t.name}** [{perm}]: {t.description}")
                if t.parameters:
                    params_str = ", ".join(
                        f"{k}: {v.get('type','any')}"
                        for k, v in t.parameters.get("properties", {}).items()
                    )
                    lines.append(f"  Parameters: ({params_str})")
            lines.append("")
        return "\n".join(lines)
```

### 5c. Normalizer (backend/mcp/normalizer.py)

```python
"""Standardize MCP responses into ToolResponse objects."""
import json
from backend.models.schemas import ToolResponse

def normalize_response(server: str, tool: str, raw, latency_ms: int) -> ToolResponse:
    """Convert raw MCP result to ToolResponse."""
    try:
        # MCP results typically have .content which is a list of content blocks
        if hasattr(raw, 'content'):
            data = []
            for block in raw.content:
                if hasattr(block, 'text'):
                    data.append(block.text)
            data = "\n".join(data) if data else str(raw)
        else:
            data = str(raw)

        return ToolResponse(
            tool=tool, server=server, status="success",
            data=data, latency_ms=latency_ms,
            token_estimate=len(str(data)) // 4
        )
    except Exception as e:
        return ToolResponse(
            tool=tool, server=server, status="error",
            error_message=f"Normalization error: {e}",
            latency_ms=latency_ms
        )
```

---

## 6. ENGINE (backend/engine/)

### 6a. Strategist (backend/engine/strategist.py)

```python
"""
THE BRAIN.
1. gather_intelligence() — parallel MCP reads
2. generate_runbook() — Claude planning call → GeneratedRunbook

MVP SIMPLIFICATION:
- Extractive summarization only (truncate, don't LLM-summarize)
- No Opus fallback (Sonnet only)
- No validation retry loop
"""
import asyncio
from pathlib import Path
from anthropic import AsyncAnthropic
from backend.config import settings
from backend.mcp.client_pool import McpClientPool
from backend.mcp.tool_registry import ToolRegistry
from backend.models.schemas import (
    IncidentContext, ToolResponse, GeneratedRunbook, RunbookStep, RunbookStepTool
)

class Strategist:

    def __init__(self, pool: McpClientPool, registry: ToolRegistry):
        self.pool = pool
        self.registry = registry
        self.client = AsyncAnthropic(api_key=settings.anthropic_api_key)

    async def gather_intelligence(
        self,
        incident: IncidentContext,
        event_callback=None
    ) -> list[ToolResponse]:
        """
        Fire parallel read-only MCP calls to gather situation data.
        The calls are tailored to the available servers.
        Emits events for each tool call (for UI streaming).
        """
        calls = []

        # Splunk: search for errors
        if "splunk" in self.pool.sessions:
            calls.append({
                "server": "splunk", "tool": "search",
                "params": {
                    "query": f"index=app source={incident.app_name} level=ERROR | head 50",
                    "timerange": "-30m"
                }
            })

        # Dynatrace: metrics + problems
        if "dynatrace" in self.pool.sessions:
            calls.append({
                "server": "dynatrace", "tool": "get_problems",
                "params": {"entity": incident.app_name}
            })
            calls.append({
                "server": "dynatrace", "tool": "get_metrics",
                "params": {
                    "entity": incident.app_name,
                    "metrics": ["error_rate", "response_time", "throughput"]
                }
            })

        # Confluence: related runbooks
        if "confluence" in self.pool.sessions:
            calls.append({
                "server": "confluence", "tool": "search_pages",
                "params": {"query": f"{incident.app_name} runbook troubleshooting"}
            })

        # Jira: recent changes/deployments
        if "jira" in self.pool.sessions:
            calls.append({
                "server": "jira", "tool": "search_issues",
                "params": {
                    "jql": f'project = SRE AND component = "{incident.app_name}" AND updated >= -7d ORDER BY updated DESC'
                }
            })

        # Fire all in parallel
        if event_callback:
            for c in calls:
                await event_callback("tool_started", {
                    "server": c["server"], "tool": c["tool"]
                })

        responses = await self.pool.call_tools_parallel(calls)

        if event_callback:
            for r in responses:
                await event_callback("tool_completed", {
                    "server": r.server, "tool": r.tool,
                    "status": r.status, "latency_ms": r.latency_ms
                })

        return responses

    def _compress_context(self, responses: list[ToolResponse]) -> str:
        """
        Extractive compression: truncate each response to fit budget.
        MVP: no abstractive summarization.
        """
        budget_per_tool = settings.max_context_tokens // max(len(responses), 1)
        sections = []
        for r in responses:
            if r.status != "success":
                sections.append(f"## {r.server}/{r.tool}: ERROR — {r.error_message}")
                continue
            data_str = str(r.data)
            # Rough truncation: 4 chars ≈ 1 token
            max_chars = budget_per_tool * 4
            if len(data_str) > max_chars:
                data_str = data_str[:max_chars] + "\n... [TRUNCATED]"
            sections.append(f"## {r.server}/{r.tool} (took {r.latency_ms}ms)\n{data_str}")
        return "\n\n".join(sections)

    async def generate_runbook(
        self,
        incident: IncidentContext,
        intelligence: list[ToolResponse],
        event_callback=None
    ) -> GeneratedRunbook:
        """
        Claude planning call. Uses tool_use to enforce structured output.
        Returns a GeneratedRunbook.
        """
        if event_callback:
            await event_callback("planning_started", {})

        # Load system prompt
        system_prompt = Path("backend/prompts/strategist_system.md").read_text()

        # Build user message
        context = self._compress_context(intelligence)
        available_tools = self.registry.get_tools_for_prompt()

        user_message = f"""
## Incident
- App: {incident.app_name}
- Environment: {incident.environment}
- Severity: {incident.severity}
- Description: {incident.description or "No description provided"}
{f"- L1 Tool Diagnosis: {incident.l1_diagnosis}" if incident.l1_diagnosis else ""}

## Intelligence Gathered
{context}

## Available Tools
{available_tools}

Generate a runbook to diagnose and remediate this incident.
"""

        # Define the output tool schema
        runbook_tool = {
            "name": "generate_runbook",
            "description": "Generate a structured incident runbook",
            "input_schema": GeneratedRunbook.model_json_schema()
        }

        response = await self.client.messages.create(
            model=settings.model,
            max_tokens=4096,
            system=system_prompt,
            messages=[{"role": "user", "content": user_message}],
            tools=[runbook_tool],
            tool_choice={"type": "tool", "name": "generate_runbook"}
        )

        # Extract structured output from tool_use block
        for block in response.content:
            if block.type == "tool_use":
                runbook = GeneratedRunbook.model_validate(block.input)

                # SAFETY OVERRIDE: force approval_required on all remediation steps
                for step in runbook.steps:
                    if step.type == "remediation":
                        step.approval_required = True

                if event_callback:
                    await event_callback("planning_completed", {
                        "diagnosis": runbook.diagnosis_summary,
                        "step_count": len(runbook.steps),
                        "yaml": runbook.to_yaml()
                    })

                return runbook

        raise RuntimeError("Claude did not return a tool_use block")
```

### 6b. Agents (backend/engine/agents.py)

```python
"""
Three agent types with hardcoded permission boundaries.
MVP SIMPLIFICATION: no retry logic, no complex error handling.
"""
from anthropic import AsyncAnthropic
from backend.mcp.client_pool import McpClientPool
from backend.mcp.tool_registry import ToolRegistry
from backend.models.schemas import RunbookStep, StepResult, StepStatus
from backend.config import settings

class DiagnosticAgent:
    """Read-only MCP calls. CANNOT call write tools."""

    def __init__(self, pool: McpClientPool, registry: ToolRegistry):
        self.pool = pool
        self.registry = registry

    async def execute(self, step: RunbookStep, context: dict) -> StepResult:
        # SECURITY CHECK
        if step.tool and self.registry.is_write_tool(step.tool.server, step.tool.method):
            raise PermissionError(
                f"DiagnosticAgent BLOCKED from write tool: {step.tool.server}/{step.tool.method}"
            )
        if not step.tool:
            return StepResult(step_id=step.id, status=StepStatus.SKIPPED,
                              description=step.description)

        response = await self.pool.call_tool(
            step.tool.server, step.tool.method, step.tool.params
        )
        return StepResult(
            step_id=step.id,
            status=StepStatus.COMPLETED if response.status == "success" else StepStatus.FAILED,
            description=step.description,
            output=response.data,
            error=response.error_message,
            latency_ms=response.latency_ms
        )


class DecisionAgent:
    """LLM-powered decision. Makes NO MCP calls."""

    def __init__(self):
        self.client = AsyncAnthropic(api_key=settings.anthropic_api_key)

    async def execute(self, step: RunbookStep, context: dict) -> StepResult:
        # Build prompt with prior step outputs
        context_str = "\n".join(
            f"Step '{k}': {str(v)[:2000]}"
            for k, v in context.items()
        )
        prompt = f"""
{step.llm_prompt or "Analyze the following data and determine the root cause."}

## Context from prior steps:
{context_str}

Respond with a JSON object containing your analysis.
"""
        response = await self.client.messages.create(
            model=settings.model,
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}]
        )
        output = response.content[0].text if response.content else "No response"
        return StepResult(
            step_id=step.id, status=StepStatus.COMPLETED,
            description=step.description, output=output
        )


class RemediationAgent:
    """Write-capable. Only runs after approval. MOCKED in demo mode."""

    def __init__(self, pool: McpClientPool):
        self.pool = pool

    async def execute(self, step: RunbookStep, context: dict) -> StepResult:
        if settings.demo_mode:
            # MOCK: simulate successful remediation
            import asyncio
            await asyncio.sleep(2)  # fake execution time for visual effect
            return StepResult(
                step_id=step.id, status=StepStatus.COMPLETED,
                description=step.description,
                output=f"[DEMO MODE] Simulated: {step.description}"
            )

        # Real execution (disabled for demo)
        if not step.tool:
            return StepResult(step_id=step.id, status=StepStatus.SKIPPED,
                              description=step.description)
        response = await self.pool.call_tool(
            step.tool.server, step.tool.method, step.tool.params
        )
        return StepResult(
            step_id=step.id,
            status=StepStatus.COMPLETED if response.status == "success" else StepStatus.FAILED,
            description=step.description, output=response.data,
            error=response.error_message, latency_ms=response.latency_ms
        )
```

### 6c. Executor (backend/engine/executor.py)

```python
"""
Sequential step executor. No DAG, no parallel groups.
Just runs steps in order with approval gates for remediation.

MVP SIMPLIFICATION: Steps execute in list order.
Decision step branches are followed by matching the next step ID.
"""
import asyncio
from backend.models.schemas import (
    GeneratedRunbook, RunbookStep, RunState, RunStatus,
    StepResult, StepStatus
)
from backend.engine.agents import DiagnosticAgent, DecisionAgent, RemediationAgent
from backend.mcp.client_pool import McpClientPool
from backend.mcp.tool_registry import ToolRegistry

class Executor:

    def __init__(self, pool: McpClientPool, registry: ToolRegistry):
        self.pool = pool
        self.registry = registry
        self.diagnostic = DiagnosticAgent(pool, registry)
        self.decision = DecisionAgent()
        self.remediation = RemediationAgent(pool)

        # Approval gate state
        self._approval_event: asyncio.Event | None = None
        self._approval_decision: str | None = None

    async def execute(
        self,
        run: RunState,
        runbook: GeneratedRunbook,
        event_callback=None
    ):
        """
        Execute runbook steps sequentially.
        Pauses at remediation steps for approval.
        Emits events for UI streaming.
        """
        run.status = RunStatus.EXECUTING
        context = {}  # accumulated step outputs

        for i, step in enumerate(runbook.steps):
            run.current_step_index = i

            if event_callback:
                await event_callback("step_started", {
                    "step_id": step.id,
                    "type": step.type,
                    "description": step.description
                })

            # ── Handle approval gate ──
            if step.type == "remediation" and step.approval_required:
                run.status = RunStatus.AWAITING_APPROVAL
                run.pending_approval_step = step.id

                if event_callback:
                    await event_callback("approval_requested", {
                        "step_id": step.id,
                        "description": step.description,
                        "rollback_plan": step.rollback_plan,
                        "risk": step.risk,
                        "context": {k: str(v)[:500] for k, v in context.items()}
                    })

                # Wait for approval via API
                self._approval_event = asyncio.Event()
                self._approval_decision = None
                await self._approval_event.wait()

                if self._approval_decision == "rejected":
                    result = StepResult(
                        step_id=step.id, status=StepStatus.SKIPPED,
                        description="Rejected by approver"
                    )
                    run.step_results.append(result)
                    if event_callback:
                        await event_callback("step_completed", result.model_dump())

                    # Skip to escalation if available
                    if step.on_failure:
                        continue  # simplified: just continue for MVP
                    run.status = RunStatus.ESCALATED
                    if event_callback:
                        await event_callback("run_completed", {"status": "escalated"})
                    return

                run.status = RunStatus.EXECUTING

            # ── Execute step ──
            try:
                if step.type == "diagnostic":
                    result = await self.diagnostic.execute(step, context)
                elif step.type == "decision":
                    result = await self.decision.execute(step, context)
                elif step.type == "remediation":
                    result = await self.remediation.execute(step, context)
                elif step.type in ("close", "escalate"):
                    result = StepResult(
                        step_id=step.id, status=StepStatus.COMPLETED,
                        description=step.description
                    )
                else:
                    result = StepResult(
                        step_id=step.id, status=StepStatus.SKIPPED,
                        description=f"Unknown step type: {step.type}"
                    )

                # Wait-before support
                if step.wait_before:
                    secs = self._parse_wait(step.wait_before)
                    if event_callback:
                        await event_callback("waiting", {
                            "step_id": step.id, "seconds": secs
                        })
                    await asyncio.sleep(min(secs, 10))  # cap at 10s for demo

            except Exception as e:
                result = StepResult(
                    step_id=step.id, status=StepStatus.FAILED,
                    description=step.description, error=str(e)
                )

            # ── Accumulate context ──
            context[step.id] = result.output
            run.step_results.append(result)

            if event_callback:
                await event_callback("step_completed", result.model_dump())

            # ── Handle branches (for decision steps) ──
            if step.type == "decision" and step.branches and result.output:
                # Simple: try to match output to a branch key
                # In MVP, DecisionAgent returns text. We look for keywords.
                output_lower = str(result.output).lower()
                for branch_key, target_step_id in step.branches.items():
                    if branch_key.lower() in output_lower:
                        # Find target step index and skip to it
                        # (simplified: just log it, sequential continues)
                        break

        # ── Complete ──
        run.status = RunStatus.RESOLVED
        if event_callback:
            await event_callback("run_completed", {"status": "resolved"})

    def submit_approval(self, decision: str):
        """Called by the API when user approves/rejects."""
        self._approval_decision = decision
        if self._approval_event:
            self._approval_event.set()

    def _parse_wait(self, wait_str: str) -> float:
        if wait_str.endswith("m"):
            return float(wait_str[:-1]) * 60
        if wait_str.endswith("s"):
            return float(wait_str[:-1])
        return 30.0
```

---

## 7. MAIN APP (backend/main.py)

```python
"""
FastAPI app. Serves:
  - POST /api/trigger — start a new incident run
  - GET  /api/stream/{run_id} — SSE stream of execution events
  - POST /api/approve/{run_id} — approve/reject remediation
  - GET  /api/status/{run_id} — current run state
  - GET  / — serves the single-page frontend

MVP SIMPLIFICATION: one run at a time. No concurrency. No history.
"""
import asyncio, json
from pathlib import Path
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse, StreamingResponse, JSONResponse
from fastapi.staticfiles import StaticFiles
from contextlib import asynccontextmanager

from backend.config import settings
from backend.mcp.client_pool import McpClientPool
from backend.mcp.tool_registry import ToolRegistry
from backend.engine.strategist import Strategist
from backend.engine.executor import Executor
from backend.models.schemas import IncidentContext, RunState, RunStatus

# ── Global State ──
pool: McpClientPool = None
registry: ToolRegistry = None
strategist: Strategist = None
executor: Executor = None
current_run: RunState | None = None
event_queues: list[asyncio.Queue] = []  # SSE subscribers

@asynccontextmanager
async def lifespan(app: FastAPI):
    global pool, registry, strategist, executor
    print("🚀 Starting SRE Agent Engine...")

    pool = McpClientPool()
    await pool.connect_all()

    registry = ToolRegistry(pool)
    await registry.discover_all()
    print(f"📦 Discovered tools from {len(registry.tools)} servers")

    strategist = Strategist(pool, registry)
    executor = Executor(pool, registry)

    yield

    print("🛑 Shutting down...")

app = FastAPI(title="SRE Agent Engine", lifespan=lifespan)


# ── Event Broadcasting ──
async def broadcast_event(event_type: str, data: dict):
    """Push event to all SSE subscribers."""
    event = {"event": event_type, "data": data}
    for q in event_queues:
        await q.put(event)


# ── Trigger Endpoint ──
@app.post("/api/trigger")
async def trigger_incident(incident: IncidentContext):
    global current_run
    current_run = RunState(incident=incident)

    # Run the full pipeline in background
    asyncio.create_task(_run_pipeline(current_run))

    return {"run_id": current_run.id, "status": "started"}


async def _run_pipeline(run: RunState):
    """Full pipeline: gather → plan → execute."""
    try:
        # Phase 1: Gather
        run.status = RunStatus.GATHERING
        await broadcast_event("status_change", {"status": run.status.value})
        intelligence = await strategist.gather_intelligence(
            run.incident, event_callback=broadcast_event
        )

        # Phase 2: Plan
        run.status = RunStatus.PLANNING
        await broadcast_event("status_change", {"status": run.status.value})
        runbook = await strategist.generate_runbook(
            run.incident, intelligence, event_callback=broadcast_event
        )
        run.runbook_yaml = runbook.to_yaml()
        run.diagnosis_summary = runbook.diagnosis_summary

        # Phase 3: Execute
        await executor.execute(run, runbook, event_callback=broadcast_event)

    except Exception as e:
        run.status = RunStatus.FAILED
        await broadcast_event("error", {"message": str(e)})


# ── SSE Stream ──
@app.get("/api/stream/{run_id}")
async def event_stream(run_id: str):
    queue = asyncio.Queue()
    event_queues.append(queue)

    async def generate():
        try:
            while True:
                event = await queue.get()
                yield f"event: {event['event']}\ndata: {json.dumps(event['data'])}\n\n"
        except asyncio.CancelledError:
            pass
        finally:
            event_queues.remove(queue)

    return StreamingResponse(generate(), media_type="text/event-stream")


# ── Approval ──
@app.post("/api/approve/{run_id}")
async def approve_action(run_id: str, request: Request):
    body = await request.json()
    decision = body.get("decision", "rejected")  # "approved" or "rejected"
    executor.submit_approval(decision)
    return {"status": "submitted", "decision": decision}


# ── Status ──
@app.get("/api/status/{run_id}")
async def get_status(run_id: str):
    if not current_run or current_run.id != run_id:
        return JSONResponse(status_code=404, content={"error": "Run not found"})
    return current_run.model_dump(mode="json")


# ── Serve Frontend ──
@app.get("/", response_class=HTMLResponse)
async def serve_frontend():
    return Path("frontend/index.html").read_text()
```

---

## 8. SYSTEM PROMPT (backend/prompts/strategist_system.md)

```markdown
You are an expert Site Reliability Engineer at a major financial institution.
You generate structured incident runbooks.

## Domain Knowledge
- Services: CDS API Gateway (Java, K8s), shared-services in Asset & Wealth Management
- Observability: Splunk (logs), Dynatrace (metrics/APM), Confluence (runbooks), Jira (changes)
- Common failures: OOM, upstream timeouts, cert expiry, config regression, DB pool exhaustion

## Rules
1. Start with diagnostic steps to gather evidence
2. Include a decision step before any remediation
3. EVERY remediation step MUST have approval_required: true
4. EVERY remediation MUST be followed by a verification step
5. ONLY reference tools from the Available Tools section
6. Include rollback_plan for every remediation
7. If confidence < 0.6, escalate instead of remediating
8. Keep runbooks to 4-8 steps (concise, actionable)

## Remediation Priority
1. Rollback (if recent deploy correlated)
2. Rolling restart (if resource exhaustion)
3. Scale up (if load-related)
4. Escalate (if unclear)
```

---

## 9. FRONTEND (frontend/index.html)

> Single HTML file. Tailwind CDN. Alpine.js for reactivity. SSE for live updates.
> Zero build step. Just open in browser.
>
> The UI has 3 panels:
>   LEFT: Trigger form + status indicator
>   CENTER: Live execution trace (step-by-step with status icons)
>   RIGHT: Generated runbook YAML viewer
>
> An approval modal overlays when remediation is proposed.
>
> IMPORTANT: This should look POLISHED. Management judges the demo by the UI.
> Use dark theme, monospace for code/data, clean animations for step transitions.
> The "wow" moment is watching steps light up green one by one.

---

## 10. BUILD ORDER (8 Days)

```
DAY 1 — SID Gate + First MCP Call
  [ ] Copy mcp.json from DevGPT/Cline
  [ ] Implement config.py + .env
  [ ] Implement mcp/normalizer.py
  [ ] Implement mcp/client_pool.py (connect + call_tool only)
  [ ] TEST: connect to Splunk, call search, print response
  [ ] If SID fails: set up API token fallback
  ★ WIN: "I can pull real Splunk data from Python"

DAY 2 — All MCP Servers + Tool Registry
  [ ] Connect Dynatrace, Confluence, Jira
  [ ] Implement mcp/tool_registry.py
  [ ] Implement call_tools_parallel
  [ ] TEST: parallel calls to all 4 servers
  ★ WIN: "I can pull data from 4 tools in parallel in <5s"

DAY 3 — Models + Strategist Gather
  [ ] Implement models/schemas.py (all models)
  [ ] Implement strategist gather_intelligence()
  [ ] TEST: gather returns ToolResponses from real servers
  ★ WIN: "I have real incident data from all tools in one place"

DAY 4 — Strategist Generate (THE KEY DAY)
  [ ] Write prompts/strategist_system.md (YOUR domain expertise)
  [ ] Implement strategist generate_runbook()
  [ ] TEST: feed real data → get YAML runbook back from Claude
  [ ] EVALUATE: is the runbook sensible? Iterate on prompt if not.
  ★ WIN: "Claude generates a runbook I'd actually follow"

DAY 5 — Agents + Executor
  [ ] Implement agents.py (all 3 types)
  [ ] Implement executor.py (sequential execution)
  [ ] TEST: full pipeline via Python script (no API yet)
  ★ WIN: "End-to-end works: gather → plan → execute → approve (stdin)"

DAY 6 — FastAPI + SSE
  [ ] Implement main.py (all endpoints)
  [ ] TEST: trigger via curl, stream events via curl
  [ ] TEST: approval via curl
  ★ WIN: "Full pipeline works via HTTP"

DAY 7 — Frontend
  [ ] Build frontend/index.html (single file, polished)
  [ ] Connect SSE stream, trigger button, approval modal
  [ ] Polish: animations, colors, timing
  ★ WIN: "The demo looks impressive in a browser"

DAY 8 — Rehearsal
  [ ] Run the demo scenario 5 times end-to-end
  [ ] Fix any flakiness
  [ ] Time the demo (target: 45-60 seconds of execution)
  [ ] Prepare talking points for each phase
  [ ] Record a backup video (in case live demo fails)
  ★ WIN: "I can do this demo in my sleep"
```
