Before continuing, add these to scope:

0. CHECKPOINT: Show your plan first: stage, specialist loaded, files you'll read in order, artifacts you'll write, and what you will NOT cover. Then stop and wait for my go.

1. SOURCE HIERARCHY: pathfinder-architecture.html is a derived summary, not ground truth. Verify every claim you use from it against repos/cc-pathfinder code. Flag any mismatch.

2. QUERY TAXONOMY: Classify CDS queries into: (a) KB-answerable, (b) single live lookup (GUDI, count), (c) multi-step troubleshooting (trace → logs), (d) write/workflow (onboard, cancel). Map each class to: current Pathfinder path, CDS MCP POC tool if any, AEGIS kit component if any. Include the queries tested in the POC and their results.

3. TOOL OVERLAP: Matrix of Pathfinder CDS tools vs POC MCP tools: duplicate, gap, or conflicting behaviour.

4. Q4 CRITERIA, scored per approach: determinism of routing, auditability, write-action safety, identity propagation (ADFS user → MCP → Splunk/DB), per-app onboarding cost, contract versioning, testability, blast radius of one bad server, and who owns each server.

5. Q3 SPECIFICS: multi-server fan-out: tool-name collisions, partial failure (one server down), cross-app questions, latency, transport and where servers run in Pathfinder's hosting environment. Say "I don't know" for hosting/auth details not in the repos.

6. FEASIBILITY BAR: Define what evidence would make each option "feasible" (e.g. a golden set of N CDS queries passing via MCP), and state whether that evidence exists today.

7. REUSE BOUNDARY: For each AEGIS component, state: reuse as dependency, reuse as pattern only, or not applicable. No code copying.
