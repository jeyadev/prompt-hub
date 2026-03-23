# CDS Incident Data Analysis — GitHub Copilot Prompt

## How to Use This

### Recommended approach
1. Open your ServiceNow CSV export in VS Code
2. Open Copilot Chat (Ctrl+Shift+I or Cmd+Shift+I)
3. Reference the file with `#file:your-incident-export.csv`
4. Paste the **Master Prompt** below
5. Copilot will generate a Python script — save it and run it
6. If Copilot truncates output, use the **Modular Prompts** section to feed one phase at a time

### Before you start
- Confirm your CSV column names match what the prompt expects (see Field Mapping section)
- Have Python 3.9+ with pandas, scikit-learn, matplotlib, seaborn installed
- Expected data volume: 1 year of CDS incidents (hundreds to low thousands of rows)

---

## Field Mapping

The prompt assumes standard ServiceNow field names. **Edit the FIELD_MAP dictionary in the generated script** if your export uses different column headers.

```
Expected fields (ServiceNow defaults):
- number              → Incident ID (e.g., INC0012345)
- short_description   → Title / summary
- description         → Full description
- work_notes          → Chronological work log entries
- close_notes         → Resolution summary
- priority            → P1/P2/P3/P4
- severity            → Severity level
- state               → Current state (Resolved, Closed, etc.)
- assignment_group    → Team assigned
- assigned_to         → Individual assigned
- opened_at           → Created timestamp
- resolved_at         → Resolved timestamp
- closed_at           → Closed timestamp
- sys_updated_on      → Last updated timestamp
- category            → ServiceNow category
- subcategory         → ServiceNow subcategory
- u_root_cause        → Root cause (if populated)
- cmdb_ci             → Configuration item / service name
- reassignment_count  → Number of reassignments
- reopen_count        → Number of reopens
```

> Adjust the FIELD_MAP in the script if your export differs. The script will warn you about missing columns and degrade gracefully.

---

## Master Prompt

Copy everything between the `---START---` and `---END---` markers.

---START---

```
#file:your-incident-export.csv

Generate a complete, self-contained Python script that performs comprehensive SRE incident analysis on the attached CSV file (1 year of ServiceNow incident data for a shared-services platform called CDS — 12 critical services, 100+ APIs, serving downstream consumers across an Asset & Wealth Management division at a large bank).

The script must be production-quality, handle messy real-world ServiceNow data, and output a single comprehensive Markdown report file called `cds_incident_analysis_report.md`.

## FIELD MAPPING

Use this dictionary at the top of the script so column names can be adjusted in one place:

FIELD_MAP = {
    'id': 'number',
    'title': 'short_description',
    'description': 'description',
    'work_notes': 'work_notes',
    'close_notes': 'close_notes',
    'priority': 'priority',
    'severity': 'severity',
    'state': 'state',
    'assignment_group': 'assignment_group',
    'assigned_to': 'assigned_to',
    'opened_at': 'opened_at',
    'resolved_at': 'resolved_at',
    'closed_at': 'closed_at',
    'updated_at': 'sys_updated_on',
    'category': 'category',
    'subcategory': 'subcategory',
    'root_cause': 'u_root_cause',
    'config_item': 'cmdb_ci',
    'reassignment_count': 'reassignment_count',
    'reopen_count': 'reopen_count'
}

At startup, check which mapped fields actually exist in the CSV. Print warnings for missing fields. Proceed with available fields — every analysis section should degrade gracefully if its required fields are missing.

## REQUIREMENTS

Use only: pandas, scikit-learn, numpy, matplotlib, seaborn, collections, re, datetime. No LLM APIs. No network calls.

## DATA PREPROCESSING

1. Parse all timestamp fields. Handle mixed formats (ServiceNow exports are inconsistent). Coerce failures to NaT.
2. Strip HTML tags and ServiceNow formatting artifacts from work_notes, close_notes, description.
3. Compute derived fields:
   - `time_to_resolve_hours`: (resolved_at - opened_at) in hours
   - `time_to_close_hours`: (closed_at - opened_at) in hours
   - `aging_days`: for unresolved tickets, (today - opened_at) in days
   - `hour_of_day`: hour extracted from opened_at
   - `day_of_week`: day name extracted from opened_at
   - `month`: month extracted from opened_at
   - `is_business_hours`: True if opened between 09:00-18:00 IST (UTC+5:30) Mon-Fri
4. Normalize text fields: lowercase, remove special chars, collapse whitespace for NLP processing. Keep originals for display.

## ANALYSIS SECTIONS — implement ALL of these as functions that each return a Markdown string

### SECTION 1: Executive Summary
- Total incidents, incidents by state, by priority
- Mean/median/p90/p99 time-to-resolve by priority
- Top 5 assignment groups by volume
- Top 5 configuration items (services) by volume
- Month-over-month trend (volume, TTR)
- 3-5 bullet "Key Findings" synthesized from the analyses below

### SECTION 2: Incident Categorization via Clustering
- Combine title + description into a single text corpus
- TF-IDF vectorization (max_features=5000, stop_words='english', ngram_range=(1,2))
- K-Means clustering. Use silhouette score to pick optimal k between 5 and 20.
- For each cluster:
  - Auto-generate a descriptive label from top 10 TF-IDF terms
  - Count of incidents
  - Mean TTR
  - Priority distribution
  - Top 3 example titles
- Output: cluster summary table sorted by incident count descending
- Output: list of "Automation Candidates" — clusters with high volume + low TTR variance (repetitive, predictable work)

### SECTION 3: Repeat Offender / Recurrence Analysis
- Within each cluster from Section 2, find incidents with cosine similarity > 0.7 to each other
- Group these into "recurrence chains" — incidents that are essentially the same problem happening repeatedly
- For each recurrence chain:
  - Pattern description (from common terms)
  - Count of occurrences
  - Time span (first occurrence to last)
  - Mean TTR
  - Whether TTR is improving or worsening over time (linear trend)
- Output: Top 15 recurrence patterns ranked by total cumulative hours spent
- Flag: patterns where TTR is getting worse (team is losing context, not gaining it)

### SECTION 4: Rerouting / Reassignment Analysis
- If reassignment_count field exists: distribution stats, correlation with TTR
- Mine work_notes for reassignment signals: regex for patterns like "reassigned to", "transferred to", "routed to", "escalated to", "moved to"
- Build a team-to-team transition matrix: which teams pass tickets to which other teams?
- Identify:
  - Top 10 most-rerouted incident patterns (cluster + reassignment count)
  - "Ping-pong" tickets: incidents that bounce between the same 2-3 teams
  - Average TTR impact of each additional reassignment
  - Teams that are frequent "wrong first assignment" targets (receive then immediately reroute)
- Output: reassignment heatmap data (from_team → to_team matrix with counts)
- Output: "Triage Accuracy" score per category/cluster: % of incidents resolved by first-assigned team

### SECTION 5: Resolution Pattern Extraction
- From close_notes and late-stage work_notes (last 2 entries), extract resolution actions
- TF-IDF on resolution text, cluster into resolution archetypes
- For each resolution archetype:
  - Descriptive label from top terms
  - Count
  - Which incident clusters (Section 2) map to which resolution archetypes
  - Mean TTR
- Identify: "Runbook Candidates" — resolution archetypes that are (a) high frequency, (b) low TTR variance, (c) consistently described in close_notes (low text entropy)
- Output: incident cluster → resolution archetype mapping table

### SECTION 6: Team Collaboration & Load Analysis
- Top assignment groups by: total volume, mean TTR, P1/P2 volume, reassignment rate
- For each top team:
  - Incident volume over time (monthly trend)
  - Priority mix
  - Mean TTR vs platform average
  - Reassignment rate (in and out)
- Team interaction graph: which teams co-occur on the same incidents (from work_notes mentions and reassignment chains)?
- Identify:
  - Teams that are overloaded (volume trending up, TTR trending up)
  - Teams that are bottlenecks (high incoming reassignment, high TTR)
  - Teams with high bus-factor risk (single assigned_to handles disproportionate volume)
- Output: team load summary table

### SECTION 7: Time-to-Resolve Deep Dive
- TTR distribution overall and by priority (histogram stats)
- TTR by cluster — which incident types take longest?
- TTR outlier analysis: incidents > 2 standard deviations above mean TTR for their cluster
  - For each outlier: ID, title, cluster, TTR, brief note on what made it slow (from work_notes keywords: "waiting", "blocked", "dependency", "approval", "vendor", "change window")
- Aging ticket analysis: currently unresolved tickets, aging days, assigned team
- TTR trend over time: is the platform getting faster or slower at resolving incidents?

### SECTION 8: Priority & Severity Analysis
- Volume by priority and severity
- Cross-tabulation: priority vs severity (identify mismatches)
- TTR by priority — are P1s actually resolved faster?
- "Priority Inflation" check: is the share of P1/P2 increasing over time without a corresponding increase in actual impact?
- "Silent P3" check: P3/P4 incidents with TTR > median P1 TTR (low priority but high effort — potential misclassification)

### SECTION 9: Temporal Patterns
- Incident volume by: hour of day, day of week, month
- Heatmap data: day_of_week x hour_of_day
- Business hours vs off-hours volume and TTR comparison
- Identify: peak incident creation windows, off-hours incident characteristics
- Correlation: incident volume spikes with specific months or periods (potential deployment/migration correlation — flag but don't assume)

### SECTION 10: Escalation Path Analysis
- Mine work_notes for escalation signals: "escalated", "L1", "L2", "L3", "manager", "leadership", "bridge call"
- Estimate tier progression: % that stay L1, % that reach L2, % that reach L3+
- Keywords in early work_notes that predict escalation (compare TF-IDF vectors of escalated vs non-escalated incidents)
- Identify: mean TTR added by each escalation tier
- Output: escalation funnel metrics

### SECTION 11: Migration-Correlated Incidents
- Filter incidents where title, description, or work_notes contain migration-related terms: "migration", "pithos", "IBM CM", "content management", "CM query", "cutover", "data migration", "legacy"
- Trend: are migration-related incidents increasing or decreasing over time?
- Cluster these separately: what types of migration incidents occur?
- TTR comparison: migration-related vs non-migration incidents
- Output: migration incident trend and typology

### SECTION 12: Friction & Delay Signal Detection
- Mine work_notes for delay/friction language using keyword lists:
  - Waiting signals: "waiting on", "pending", "no response", "awaiting", "blocked by"
  - Confusion signals: "unclear", "unable to determine", "not sure", "need clarification", "which team"
  - Dependency signals: "dependency", "depends on", "need access", "firewall", "approval needed", "change window"
  - Handoff signals: "please check", "can you look", "FYI", "over to you", "handing off"
- For each signal category:
  - Count of incidents containing these signals
  - Mean TTR impact (incidents with signal vs without)
  - Top teams associated with each friction type
- Output: friction signal summary with TTR impact

### SECTION 13: Service-Level Incident Mapping
- If config_item field exists: incidents per service/CI
- If not: attempt to extract service names from title + description (look for consistent capitalized terms, API names, service identifiers)
- For each identified service:
  - Incident volume, priority mix, mean TTR
  - Top incident clusters
  - Trend over time
- Identify: services with worsening incident trends, services with highest operational burden

## REPORT OUTPUT

Write all sections to a single Markdown file: `cds_incident_analysis_report.md`

Report structure:
1. Report header with generation timestamp and data date range
2. Executive Summary (Section 1) — always first
3. All analysis sections with clear Markdown headers
4. Final section: "Recommended Actions" — synthesize top 10 actionable recommendations ranked by estimated operational impact, each with:
   - What to do
   - Why (which analysis supports it)
   - Expected impact
   - Effort estimate (Low/Medium/High)

Also generate individual CSV files for key outputs that an SRE team would want to slice further:
- `incident_clusters.csv` — each incident with its cluster ID and cluster label
- `recurrence_patterns.csv` — recurrence chains with counts and TTR stats
- `reassignment_matrix.csv` — from_team, to_team, count
- `team_load_summary.csv` — team-level metrics
- `migration_incidents.csv` — filtered migration-related incidents
- `aging_tickets.csv` — currently unresolved tickets sorted by age

## CODE QUALITY

- Wrap each section in a try/except so one failure doesn't kill the entire report. On failure, write "Analysis failed: {error}" into that section and continue.
- Use logging, not print, for progress updates
- Handle encoding issues in CSV read (try utf-8, then latin-1, then cp1252)
- Handle ServiceNow's inconsistent timestamp formats
- Handle NaN/empty text fields gracefully in all NLP operations
- Total script should be runnable with: python analyze_incidents.py <path_to_csv>
- Accept CSV path as command line argument with a sensible default
```

---END---

---

## Modular Prompts (Use if Copilot truncates the master prompt)

If Copilot can't generate the full script in one pass, feed these one at a time. Ask Copilot to generate each as a standalone function, then combine them.

### Module 1: Setup & Preprocessing

```
#file:your-incident-export.csv

Generate the imports, FIELD_MAP config, CSV loading, and preprocessing code for a ServiceNow incident analysis script. Include:
- FIELD_MAP dictionary for column name abstraction
- CSV loading with encoding fallback (utf-8 → latin-1 → cp1252)
- Timestamp parsing with mixed format handling
- HTML/formatting cleanup for text fields
- Derived field computation: TTR, aging_days, hour_of_day, day_of_week, month, is_business_hours (IST timezone)
- Text normalization for NLP (lowercase, clean, collapse whitespace) stored in separate columns
- Missing field detection with warnings
- Output: cleaned DataFrame ready for analysis
```

### Module 2: Clustering & Categorization

```
Generate a function `analyze_clusters(df)` that:
- Combines normalized title + description
- TF-IDF with max_features=5000, bigrams, english stop words
- K-Means with silhouette-score-based k selection (range 5-20)
- Returns cluster labels, top terms per cluster, summary stats (count, mean TTR, priority distribution, example titles)
- Identifies automation candidates: high volume + low TTR variance clusters
- Adds cluster_id and cluster_label columns to the DataFrame
- Returns Markdown string for report section
```

### Module 3: Recurrence Analysis

```
Generate a function `analyze_recurrence(df, tfidf_matrix)` that:
- Within each cluster, computes pairwise cosine similarity
- Groups incidents with similarity > 0.7 into recurrence chains
- For each chain: pattern description, count, time span, mean TTR, TTR trend direction
- Returns top 15 chains ranked by cumulative hours spent
- Flags chains where TTR is worsening
- Returns Markdown string for report section
```

### Module 4: Rerouting & Reassignment

```
Generate a function `analyze_rerouting(df)` that:
- Uses reassignment_count field if available
- Mines work_notes for reassignment/transfer patterns via regex
- Builds team-to-team transition matrix
- Identifies ping-pong tickets, wrong-first-assignment teams
- Computes triage accuracy per cluster
- Returns Markdown string and CSV data for reassignment matrix
```

### Module 5: Resolution Patterns

```
Generate a function `analyze_resolutions(df)` that:
- Extracts resolution text from close_notes + last work_notes entries
- TF-IDF clusters resolution text into archetypes
- Maps incident clusters to resolution archetypes
- Identifies runbook candidates: high frequency + low variance + consistent description
- Returns Markdown string for report section
```

### Module 6: Team Analysis

```
Generate a function `analyze_teams(df)` that:
- Top teams by volume, TTR, priority mix, reassignment rate
- Monthly trend per team
- Team interaction graph from work_notes co-occurrence
- Flags: overloaded teams (volume up + TTR up), bottleneck teams (high incoming reassignment), bus-factor risk (single assignee dominance)
- Returns Markdown string and CSV for team summary
```

### Module 7: TTR & Aging Deep Dive

```
Generate a function `analyze_ttr(df)` that:
- TTR distribution overall and by priority and cluster
- Outlier detection (>2 std dev) with work_notes keyword analysis for delay causes
- Aging ticket list (unresolved, sorted by age)
- TTR trend over time (monthly, with linear regression slope)
- Returns Markdown string and CSV for aging tickets
```

### Module 8: Priority, Temporal & Escalation Analysis

```
Generate a function `analyze_priority_temporal_escalation(df)` that:
- Priority × severity cross-tabulation and mismatch detection
- Priority inflation trend
- Silent P3 identification (P3/P4 with TTR > median P1 TTR)
- Volume by hour/day/month, business-hours comparison
- Escalation signal mining from work_notes
- Escalation funnel: L1 → L2 → L3 percentages
- Predictive keywords for escalation
- Returns Markdown string
```

### Module 9: Migration, Friction & Service Mapping

```
Generate a function `analyze_migration_friction_services(df)` that:
- Filters migration-related incidents by keyword matching
- Trends migration incidents over time
- Mines work_notes for friction signals (waiting, confusion, dependency, handoff)
- Computes TTR impact per friction type
- Maps incidents to services/CIs from config_item or text extraction
- Per-service stats: volume, priority, TTR, trends
- Returns Markdown string and migration incidents CSV
```

### Module 10: Synthesis & Report Assembly

```
Generate the main() function that:
- Loads and preprocesses data (Module 1)
- Runs all analysis functions (Modules 2-9)
- Assembles report: header → executive summary → all sections → recommended actions
- Writes cds_incident_analysis_report.md
- Writes all supplementary CSVs
- Wraps each section in try/except for resilience
- Accepts CSV path as CLI argument
- Uses logging for progress
```

---

## Post-Analysis: Follow-Up Prompts for Copilot

After the script runs and you have the report, use these follow-up prompts to go deeper:

### Drill into a specific cluster
```
Looking at cluster [X] from the incident analysis — the [label] cluster with [N] incidents.
Analyze the work_notes for all incidents in this cluster. What are the exact steps engineers
take to resolve these? Generate a draft runbook outline.
```

### Validate automation candidates
```
For the top 5 automation candidate clusters identified in the report, analyze the close_notes
to determine: (1) Can resolution be fully automated? (2) What data/access would automation
need? (3) What's the failure mode if automation gets it wrong? Generate an automation
feasibility matrix.
```

### Investigate aging tickets
```
For the top 20 aging tickets from aging_tickets.csv, analyze the work_notes chronologically.
For each ticket, identify: (1) Where it got stuck, (2) What's blocking resolution,
(3) Recommended next action. Output as a triage action list.
```

### Deeper team dependency analysis
```
Using the reassignment_matrix.csv, generate a network graph visualization showing team
dependencies. Highlight: (1) Bidirectional heavy flows (ownership confusion), (2) Teams
that are pure sinks (everything flows in, nothing out — potential bottlenecks),
(3) Teams that are pure sources (everything flows out — potential triage problems).
```
