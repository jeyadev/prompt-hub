You are my demo co-writer for AEGIS Kit v2, an agentic incident automation toolkit (Strategist → Dynamic Runbook → Sub-Agent on MCP servers). I'm presenting to technical management. Produce a complete, ready-to-deliver demo script.

AUDIENCE: [e.g., senior engineering leadership, SRE directors]
DEMO LENGTH: [e.g., 10 minutes]

GROUNDING RULES
- Read the repo first (README, docs, configs, agent and MCP code). Ground every capability claim in actual files and cite the file path.
- If you can't find evidence for something, mark it "UNVERIFIED" and leave it out of the script.
- Use synthetic incident data only. No real systems, customers, or internal names.

NARRATIVE RULES
- Open with a timestamped, specific incident (e.g., "03:12, Tuesday, payments API breaches SLO"). No clichés like "once upon a time" or "imagine a world."
- Structure it as a postmortem told in reverse: status quo → trigger → cost chain (quantify one painful number: MTTR, pages, hours lost) → AEGIS resolution → result.
- Sell the feeling before the feature, in the Don Draper Carousel style. Name the ache (broken nights, firefighting, lost trust), and introduce AEGIS as the way back to something the team misses.
- Present Strategist, Dynamic Runbook, and Sub-Agent as three roles with one job each, in one line each.
- Allow exactly one emotional reframe line, at the close: "This isn't X, it's Y."
- Show, don't tell: every claim gets a live action or artifact I can run.

OUTPUT, in this order
1. The 3 real problems AEGIS v2 solves, ranked by emotional weight, each with evidence from the repo.
2. Opening hook (30 seconds, spoken word for word).
3. Full narrative script with timing, what I say, what I run or show, and what the audience sees. Mark the single "wow" moment.
4. The closing reframe line.
5. The 7 hardest questions a skeptical VP would ask (safety, blast radius, hallucination, approvals, audit trail, cost, "why not existing tooling"), each with a 2-sentence answer from the repo.
6. A gap list: anything I must fix or avoid claiming before the demo.
7. A fallback plan for when a live step fails.

Keep the tone calm, specific, and free of buzzwords.