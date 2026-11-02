Goal: Produce a single self-contained HTML file (inline CSS/JS, no external dependencies, works offline) that gives me the complete picture of this workshop: every approach we explored, every approach still available, the trade-offs of each, a scored comparison, and the most logical way forward. Save it to docs/decision-map.html. Don't modify any other files.

STEP 0: GATHER HONESTLY
Go through this session, all child threads, and all side chats. Before building, list which threads/chats you could read and which you couldn't. Tag every claim in the artifact with its provenance:
- Decided (we explicitly agreed)
- Considered (discussed, not chosen)
- Rejected (with the reason)
- Inferred (your inference, not something we said)
Never invent history. If the reason for a decision wasn't recorded, write "reason not recorded".

STEP 1: CONTENT
a. The real problem: restate in 2-3 sentences what we're actually solving, with constraints and success criteria as they emerged. If the problem definition drifted across threads, show how.
b. Journey: a timeline of approaches in the order we explored them. For each: what it was, why we tried it, what we learned, and what happened to it (adopted / rejected / parked / merged).
c. Option landscape: every approach, including ones we took and ones still available but unexplored. Label unexplored ones clearly as "your suggestion, not yet discussed" so they're distinguishable from our history. Include hybrids where approaches combine.
d. Per option: plain-language description; core assumptions; FAILURE MODES FIRST (how it breaks, what we'd notice, how bad it gets), then strengths; cost/effort; reversibility; dependencies on other decisions; evidence from our threads.
e. Scoring: derive 5-7 criteria from this actual problem (not generic ones), each with a weight and a one-line justification. Score every option 1-5 per criterion with a one-line rationale per cell. Show weighted totals.
f. Stress test: does the ranking change if weights shift? Which assumptions, if wrong, flip the recommendation? Name the most antifragile option (holds up or improves under surprises) and say why.
g. Recommendation: the most logical path forward, phased, with the first concrete step, decision checkpoints, kill criteria, and what would make you change your mind. Also make the strongest argument against your own recommendation.
h. Open questions and assumptions we never validated.

STEP 2: STRUCTURE FOR HOW I LEARN
Overview first, then deep-dive. The top of the page must show the whole picture on one screen: the problem, the option landscape at a glance, the ranking, the recommendation. Everything else sits beneath it as progressive disclosure. Aim for evaluation and synthesis, not description: judge the options and show how decisions connect to each other (a decision/dependency map). Use plain language, define jargon on first use, and use analogies only where they genuinely clarify. Don't force an infrastructure or SRE metaphor.

STEP 3: INTERACTIVITY
- Sticky navigation: Overview | Journey | Options | Compare | Scores | Stress test | Recommendation | Open questions
- Expandable option cards; filter by status (taken / rejected / available / hybrid)
- Compare mode: pick 2-3 options side by side
- Scoring: sliders to reweight criteria, with ranking and chart updating live; a reset button to your default weights; a heat-map table of scores with the rationale on hover
- Inline SVG for charts (radar or bar), the timeline, and the decision map; no external libraries
- Light/dark mode, responsive, keyboard accessible, print-friendly
- Muted, professional design. Use color only to carry meaning (status, score) and always pair it with a text label.

BEFORE YOU FINISH
Check that every score matches its rationale text, every recommendation traces to evidence, and nothing is labelled "Decided" that we didn't explicitly decide. Then give me a short summary of what's in the file and any gaps you couldn't fill.
