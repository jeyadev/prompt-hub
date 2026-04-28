Generate a Mermaid diagram of the AEGIS runtime control flow for an architecture review. Read the actual code in backend/ — do not infer from filenames.

Requirements:

1. Use flowchart TD. One diagram, not multiple.

2. Show the runtime path of a single incident from POST /api/trigger 
   through to run_completed SSE event. Follow actual function calls, 
   not module structure.

3. Group nodes into subgraphs by boundary:
   - "Deterministic Layer" (pattern matcher, cache lookups, registry filters, WRITE_TOOLS check)
   - "Probabilistic Layer" (LLM calls: intake extractor, strategist, agents)
   - "MCP Layer" (each MCP server as a distinct node, labeled with transport: stdio/http)
   - "State Layer" (in-memory run dict, ChromaBackend collections, prewarm cache)
   - "Human Gate Layer" (approval modal, decision review modal)

4. Mark every edge that crosses a process boundary (MCP call) with a 
   dashed line. In-process calls use solid lines.

5. On every node that performs a write operation, prefix the label with 
   [WRITE]. On every node gated by demo_mode, suffix with (demo-mocked).

6. Show the SSE event emission points as nodes labeled with the event 
   name (e.g., "emit: tool_completed"). Place them on the edges where 
   they actually fire.

7. Show graceful-degradation paths: where an MCP failure returns a 
   structured error dict instead of raising, draw a dotted edge labeled 
   "degraded" to the next node.

8. Annotate the strategist node with its two phases: 
   gather_intelligence() (parallel MCP fan-out) and generate_runbook() 
   (LLM planning). Show the fan-out explicitly.

9. Keep node labels short (3-5 words). Put detail in edge labels.

10. At the end, output a separate "Legend" subgraph explaining: solid 
    vs dashed vs dotted edges, [WRITE] prefix, (demo-mocked) suffix, 
    and the five layer colors.

Do not include: Pydantic schema definitions, helper functions under 20 
lines, frontend Alpine internals, or test code paths.

Output the Mermaid source only, in a single code block. Do not 
summarize the code or explain the diagram in prose.