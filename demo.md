Failure modes first: an IDE assistant narrating a demo will happily invent features, tell instead of show, and stay generic. So the prompts below force it to ground every claim in your repo and build the story around one incident.

**1. Master prompt (set the frame once)**

```
You are my demo co-writer for AEGIS Kit v2 (agentic incident automation: Strategist → Dynamic Runbook → Sub-Agent on MCP servers).

Rules:
- Ground every capability claim in actual files, code, or config in this repo. Cite the file. If you can't find evidence, say "unverified" and don't claim it.
- Use ONE incident as the spine of the whole story. No generic examples.
- Structure the narrative using Don Draper's Kodak Carousel pitch: sell the feeling, not the feature. Name the emotional ache first, introduce the tool as the way back to something the audience misses (calm, sleep, trust in the system), and end by reframing what the tool is.
- Show, don't tell: every claim gets a live action or artifact I can run.
- Audience: [engineering leadership / SRE peers / execs]. Tone: calm, specific, no buzzwords.
```

**2. Story arc prompt (Pixar spine + Draper)**

```
Draft the demo narrative in this spine, one or two sentences per beat:
Once upon a time, on-call meant ___.
Every night/week, ___ (the recurring pain).
Until one day ___ (the incident that breaks the pattern).
Because of that ___, because of that ___ (consequence chain, quantify one cost).
Until finally ___ (AEGIS enters).
Ever since then ___.

Then give me the Carousel moment: one line that reframes AEGIS from "a tool that does X" into "a way to get back Y."
```

**3. Problem-agitation prompt (Problem → Agitate → Solve)**

```
From the repo's docs and incident handling logic, extract the 3 real problems AEGIS v2 solves. For each: the problem in one sentence, who feels it, what it costs (time, trust, money), and what the old workflow looked like. Rank by emotional weight, not technical interest. Flag any I can't prove from the repo.
```

**4. Reveal prompt (Rule of three, characters not components)**

```
Present Strategist, Dynamic Runbook, and Sub-Agent as three characters with one job each, in one line each. Then write a 90-second live walkthrough script: what I type or trigger, what the audience sees, and what I say at each step. Mark the exact "wow" moment.
```

**5. Stress-test prompt (run this last)**

```
Act as a skeptical VP in the audience. List the 7 hardest questions about AEGIS v2 (safety, blast radius, hallucination, approvals, audit trail, cost, "why not existing tooling"). Answer each in 2 sentences using only what's in the repo. Mark gaps I must close or avoid claiming before the demo.
```

Before the demo, two things to protect yourself:
- **Fallback:** pre-record the live moment, because one flaky MCP call kills the Carousel effect.
- **Data:** since AEGIS is JPMC IP, use synthetic incident data only.

Share the audience and what's new in v2, and I'll tighten the master prompt and write the one-line Carousel reframe for you.