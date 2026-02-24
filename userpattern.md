You are a senior SRE + data engineer. Write a production-quality Python 3.11 script to analyze IBM Content Manager style query strings stored in a CSV.

INPUT
- Path to a CSV file.
- The CSV has a column named: cm_query
- Each row is a CM query string similar to:
  /SNG_KYC[(...)] AND ( @SYSROOTATTRS.CREATETS BETWEEN CM-TIMESTAMP("2013-01-01") AND CM-TIMESTAMP("2026-02-24 05:09:11") ) AND ( @Doc_Status IS NULL OR ( @Doc_Status NOT LIKE "Invisible" AND ... ) )

OUTPUT (write into an output folder)
1) summary.json: overall stats
   - total_queries, parse_success_rate
   - top_classes (prefix like /SNG_KYC, /SNG_Credit, etc.)
   - predicate_type_counts distribution (EQUAL, IN, BETWEEN, LIKE, NOT LIKE, IS NULL, OR)
   - top_fields overall and per class
   - IN-list size distribution (min/mean/p50/p90/p99/max) for each field
   - timestamp window distribution (hours) where CREATETS BETWEEN exists
   - template_count and top_templates (frequency)
2) templates.csv:
   columns: template_id, normalized_template, frequency, example_query
3) query_features.csv: one row per original query with extracted features
   columns include:
   - row_id, class_name, has_sortby, sort_field, sort_dir(if any)
   - num_predicates, num_or, num_not_like, num_like, num_in, num_between, num_is_null, num_equals
   - fields_used (pipe-separated)
   - in_fields (pipe-separated)
   - in_list_sizes (field:size pairs)
   - has_createts_between, createts_start, createts_end, createts_window_hours
   - complexity_score (define as: +3 per OR, +2 per NOT LIKE, +1 per 10 items in any IN list, +2 if createts_window_hours>168, +1 per BETWEEN, +1 per LIKE)
4) field_cooccurrence.csv:
   - pairs of fields and count of co-occurrence in same query (top 200)
5) (Optional but preferred) charts as PNG using matplotlib (no seaborn):
   - class frequency bar
   - IN-list size histogram (overall)
   - createts window histogram
   - complexity score histogram

PARSING REQUIREMENTS (robust heuristics)
- Do NOT try to implement a full grammar. Use careful regex + tokenization.
- Extract class_name as the prefix starting with "/" up to first "[" or "(".
- Count predicate operators by detecting patterns:
  - " IN (" ... ")" -> IN; count items split by commas at top level.
  - " BETWEEN " ... " AND " ... -> BETWEEN; for timestamps detect CM-TIMESTAMP("...").
  - " NOT LIKE " and " LIKE " (ensure NOT LIKE counted separately).
  - " IS NULL "
  - " OR " (uppercase OR surrounded by spaces) and " AND "
  - Equality: occurrences of " = " (but avoid matching inside timestamps).
- Extract field names as tokens beginning with "@" followed by letters/numbers/underscore, including dotted forms like @SYSROOTATTRS.CREATETS.
- Handle escaped quotes like \" inside the string.
- Handle extra whitespace/newlines.
- If parsing fails for a row, still emit a row in query_features.csv with parse_error=1 and best-effort class_name.

TEMPLATE NORMALIZATION
- normalized_template should:
  - replace all quoted string literals with "{STR}"
  - replace numeric literals with "{NUM}"
  - replace IN list bodies with "{LIST:n}" where n is item count
  - replace CM-TIMESTAMP("...") with CM-TIMESTAMP("{TS}")
  - collapse whitespace to single spaces
- template_id can be a stable hash (e.g., sha1 of normalized_template truncated).

CLI
- Use argparse: python analyze_cm_queries.py --input queries.csv --out outdir
- Add logging, type hints, docstrings.
- Include a small --sample N option to run on first N rows.
- Make sure it runs fast for 100k+ rows (avoid O(n^2) except limited top-k cooccurrence).

Finally, print an ops-friendly console summary:
- top 10 templates
- top 10 fields
- top 10 worst complexity queries with row_id and a short snippet
- parse success rate

Return only the final Python script content.
