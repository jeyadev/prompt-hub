# Copilot Prompt: Generate Grafana 10 Dashboard JSON from Dashboard Specification

---

## CONTEXT

You have a dashboard specification document (`.md` file) that describes a Grafana dashboard with exact panel layouts, PromQL queries, thresholds, overrides, and configuration details. Your job is to produce a **complete, importable Grafana 10 JSON file** that can be pasted directly into Grafana's "Import dashboard" (JSON Model) without manual edits.

You also have access to the `/actuator/prometheus` endpoint output for metric name validation.

---

## INPUT FILES

### Input 1: Dashboard Specification
The `.md` file describing the full dashboard layout, panels, queries, thresholds, and overrides:

```
<PASTE THE DASHBOARD SPEC .md CONTENT HERE>
```

### Input 2: Prometheus Metrics (for metric name validation)
```
<PASTE THE /actuator/prometheus OUTPUT HERE>
```

### Input 3: Datasource UID
The Prometheus datasource UID in the target Grafana instance:

```
<PASTE DATASOURCE UID HERE — find it in Grafana → Connections → Data sources → click your Prometheus source → the UID is in the URL or settings>
```

If unknown, use `"${DS_PROMETHEUS}"` as a variable placeholder so the user is prompted on import.

---

## GRAFANA 10 JSON SCHEMA REQUIREMENTS

Follow the Grafana 10 dashboard JSON model exactly. Do NOT produce Grafana 7/8/9 legacy formats.

### Top-level dashboard structure

```json
{
  "__inputs": [
    {
      "name": "DS_PROMETHEUS",
      "label": "Prometheus",
      "description": "",
      "type": "datasource",
      "pluginId": "prometheus",
      "pluginName": "Prometheus"
    }
  ],
  "__requires": [
    {
      "type": "grafana",
      "id": "grafana",
      "name": "Grafana",
      "version": "10.0.0"
    },
    {
      "type": "datasource",
      "id": "prometheus",
      "name": "Prometheus",
      "version": "1.0.0"
    }
  ],
  "annotations": { "list": [] },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 1,
  "id": null,
  "links": [],
  "panels": [],
  "schemaVersion": 39,
  "tags": [],
  "templating": { "list": [] },
  "time": { "from": "now-6h", "to": "now" },
  "timepicker": {
    "refresh_intervals": ["5s", "10s", "30s", "1m", "5m"],
    "time_options": ["5m", "15m", "1h", "6h", "12h", "24h", "2d", "7d"]
  },
  "timezone": "browser",
  "title": "",
  "uid": "",
  "version": 0,
  "refresh": "30s"
}
```

### graphTooltip values
- `0` = Default (individual)
- `1` = Shared crosshair
- `2` = Shared tooltip

Always use `1` (shared crosshair) unless spec says otherwise.

---

## PANEL TYPE REFERENCE

### Stat Panel
```json
{
  "type": "stat",
  "title": "Panel Title",
  "gridPos": { "h": 4, "w": 6, "x": 0, "y": 0 },
  "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
  "targets": [
    {
      "expr": "promql_here",
      "legendFormat": "",
      "refId": "A",
      "instant": true,
      "range": false
    }
  ],
  "fieldConfig": {
    "defaults": {
      "color": { "mode": "thresholds" },
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 5 },
          { "color": "red", "value": 20 }
        ]
      },
      "unit": "short",
      "mappings": [],
      "decimals": 1
    },
    "overrides": []
  },
  "options": {
    "reduceOptions": {
      "values": false,
      "calcs": ["lastNotNull"],
      "fields": ""
    },
    "orientation": "auto",
    "textMode": "auto",
    "wideLayout": true,
    "colorMode": "background",
    "graphMode": "area",
    "justifyMode": "auto"
  }
}
```

### CRITICAL — Threshold direction:

**Higher is better** (throughput, success rate):
```json
"steps": [
  { "color": "red", "value": null },
  { "color": "yellow", "value": 5 },
  { "color": "green", "value": 20 }
]
```
Base = red (worst case), thresholds escalate to green.

**Lower is better** (error rate, latency):
```json
"steps": [
  { "color": "green", "value": null },
  { "color": "yellow", "value": 1 },
  { "color": "red", "value": 5 }
]
```
Base = green (best case), thresholds escalate to red.

**NEVER put green as the highest threshold for a "higher is better" metric. NEVER put red as the base for a "lower is better" metric.** Get this wrong and the panel colors will be inverted.

---

### Time Series Panel
```json
{
  "type": "timeseries",
  "title": "Panel Title",
  "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
  "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
  "targets": [
    {
      "expr": "promql_here",
      "legendFormat": "Legend Text",
      "refId": "A",
      "instant": false,
      "range": true
    }
  ],
  "fieldConfig": {
    "defaults": {
      "color": { "mode": "palette-classic" },
      "custom": {
        "axisBorderShow": false,
        "axisCenteredZero": false,
        "axisColorMode": "text",
        "axisLabel": "",
        "axisPlacement": "auto",
        "barAlignment": 0,
        "drawStyle": "line",
        "fillOpacity": 10,
        "gradientMode": "none",
        "hideFrom": { "legend": false, "tooltip": false, "viz": false },
        "insertNulls": false,
        "lineInterpolation": "linear",
        "lineWidth": 2,
        "pointSize": 1,
        "scaleDistribution": { "type": "linear" },
        "showPoints": "auto",
        "spanNulls": true,
        "stacking": { "group": "A", "mode": "none" },
        "thresholdsStyle": { "mode": "off" }
      },
      "unit": "short",
      "min": 0
    },
    "overrides": []
  },
  "options": {
    "legend": {
      "calcs": ["lastNotNull", "mean", "max"],
      "displayMode": "table",
      "placement": "bottom",
      "showLegend": true
    },
    "tooltip": {
      "mode": "multi",
      "sort": "desc"
    }
  }
}
```

### CRITICAL — Time series display fixes:

1. **Always set `"spanNulls": true`** — this connects data points across scrape gaps. Without this, sparse metrics appear as disconnected dots.

2. **Always set `"lineWidth": 2`** and `"pointSize": 1`** — visible lines, minimal dot clutter.

3. **Legend as table with calcs** — always include `lastNotNull`, `mean`, `max` in legend calcs so engineers can read values without hovering.

4. **Tooltip mode "multi" with sort "desc"** — shows all series in tooltip, sorted by value. Essential for multi-query panels.

---

### Time Series — Dual Y-Axis (via Overrides)

When a panel has queries on different scales (e.g., CPU % on left, thread count on right), use overrides to assign series to the right axis:

```json
"overrides": [
  {
    "matcher": { "id": "byName", "options": "Live Threads" },
    "properties": [
      {
        "id": "custom.axisPlacement",
        "value": "right"
      },
      {
        "id": "custom.axisLabel",
        "value": "count"
      },
      {
        "id": "unit",
        "value": "short"
      }
    ]
  }
]
```

---

### Time Series — Per-Series Color Overrides

When the spec assigns specific colors to series (e.g., errors=red, warnings=orange):

```json
"overrides": [
  {
    "matcher": { "id": "byName", "options": "Errors/min" },
    "properties": [
      {
        "id": "color",
        "value": { "fixedColor": "red", "mode": "fixed" }
      }
    ]
  },
  {
    "matcher": { "id": "byName", "options": "Warnings/min" },
    "properties": [
      {
        "id": "color",
        "value": { "fixedColor": "orange", "mode": "fixed" }
      }
    ]
  }
]
```

**The matcher `options` value must match the `legendFormat` string exactly.** If the legend uses `{{label}}` template syntax, the matcher must match the resolved label value, not the template.

---

### Time Series — Horizontal Threshold Line

To draw a dashed threshold line on a time series panel (e.g., red line at 5 errors/min):

```json
"fieldConfig": {
  "defaults": {
    "custom": {
      "thresholdsStyle": { "mode": "line" }
    },
    "thresholds": {
      "mode": "absolute",
      "steps": [
        { "color": "green", "value": null },
        { "color": "red", "value": 5 }
      ]
    }
  }
}
```

Use `"mode": "line"` for a single line, `"mode": "line+area"` for line with shaded region above, or `"mode": "dashed"` for dashed line. Use `"mode": "off"` to disable (default).

---

### Table Panel (Instant Query)
```json
{
  "type": "table",
  "title": "Panel Title",
  "gridPos": { "h": 8, "w": 24, "x": 0, "y": 0 },
  "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
  "targets": [
    {
      "expr": "promql_here",
      "legendFormat": "",
      "refId": "A",
      "instant": true,
      "range": false,
      "format": "table"
    }
  ],
  "fieldConfig": {
    "defaults": {},
    "overrides": []
  },
  "options": {
    "showHeader": true,
    "sortBy": [{ "displayName": "Value", "desc": true }]
  },
  "transformations": [
    {
      "id": "organize",
      "options": {
        "excludeByName": { "Time": true, "__name__": true },
        "indexByName": {},
        "renameByName": {
          "jobId": "Job ID",
          "hasException": "Has Errors",
          "skipProcessing": "Skipped",
          "Value": "Document Count"
        }
      }
    }
  ]
}
```

### CRITICAL — Table panels:
1. **Always set `"instant": true` and `"range": false`** — range queries produce time series data that breaks the "Organize fields" transformation.
2. **Always set `"format": "table"`** — this splits label sets into individual columns.
3. **Exclude `Time` and `__name__` columns** in the organize transformation.
4. **If multiple queries feed one table**, add a `"merge"` transformation BEFORE `"organize"`:
```json
"transformations": [
  { "id": "merge", "options": {} },
  { "id": "organize", "options": { ... } }
]
```

---

### Heatmap Panel (for histogram metrics)
```json
{
  "type": "heatmap",
  "title": "Panel Title",
  "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
  "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
  "targets": [
    {
      "expr": "sum(increase(metric_bucket{labels}[$__rate_interval])) by (le)",
      "legendFormat": "{{le}}",
      "refId": "A",
      "format": "heatmap",
      "instant": false,
      "range": true
    }
  ],
  "options": {
    "calculate": false,
    "cellGap": 1,
    "color": {
      "exponent": 0.5,
      "fill": "dark-orange",
      "mode": "scheme",
      "reverse": false,
      "scale": "exponential",
      "scheme": "Oranges",
      "steps": 64
    },
    "yAxis": {
      "axisPlacement": "left",
      "unit": "s"
    },
    "tooltip": {
      "show": true,
      "yHistogram": true
    },
    "legend": { "show": true }
  }
}
```

### CRITICAL — Heatmap:
1. **Set `"format": "heatmap"`** on the target — not "time_series".
2. **Set `"calculate": false`** — the Prometheus histogram buckets are pre-calculated.
3. **Use `increase()` not `rate()`** for heatmap bucket queries — `rate()` produces per-second values that lose bucket semantics.

---

### Row Panel (collapsible section header)
```json
{
  "type": "row",
  "title": "Row Title",
  "gridPos": { "h": 1, "w": 24, "x": 0, "y": 0 },
  "collapsed": false,
  "panels": []
}
```

Place a row panel before each logical section of the dashboard. Set `"collapsed": true` for rows that should be collapsed by default (e.g., resource utilization rows).

---

## TEMPLATE VARIABLE REFERENCE

### Standard variable structure
```json
{
  "templating": {
    "list": [
      {
        "name": "pod",
        "label": "Pod Name",
        "type": "query",
        "datasource": { "type": "prometheus", "uid": "${DS_PROMETHEUS}" },
        "query": {
          "query": "label_values(amcds_counter_migratePipeline_total, podName)",
          "refId": "StandardVariableQuery"
        },
        "refresh": 2,
        "regex": "",
        "sort": 1,
        "multi": true,
        "includeAll": true,
        "allValue": ".*",
        "current": { "selected": true, "text": "All", "value": "$__all" }
      }
    ]
  }
}
```

### CRITICAL — Multi-value variable config:
1. **`"multi": true`** — enables selecting multiple values.
2. **`"includeAll": true`** — adds "All" option.
3. **`"allValue": ".*"`** — when "All" is selected, the regex `.*` matches everything. WITHOUT THIS, selecting "All" produces a pipe-separated list of every value which can break with high cardinality.
4. **`"refresh": 2`** — refresh on time range change (1 = on dashboard load, 2 = on time range change).
5. **`"sort": 1`** — alphabetical ascending (0 = disabled, 1 = asc, 2 = desc, 3 = numerical asc, 4 = numerical desc).

### Variable reference in queries
- In PromQL use regex match: `podName=~"$pod"` — NOT `podName="$pod"`.
- The `=~` operator handles both single selection (literal match) and multi-selection (pipe-separated regex).

---

## GRID POSITIONING RULES

Grafana uses a 24-column grid. Panel positions are defined by `gridPos`:

```json
"gridPos": { "h": height, "w": width, "x": column, "y": row }
```

| Layout | w value | Panels per row |
|--------|---------|---------------|
| Full width | 24 | 1 |
| Half width | 12 | 2 |
| Third width | 8 | 3 |
| Quarter width | 6 | 4 |

Standard heights:
- Row headers: `h: 1`
- Stat panels: `h: 4`
- Time series / heatmap: `h: 8`
- Tables: `h: 8` to `h: 10`

**Calculate `y` positions sequentially.** Each row's `y` = previous row's `y` + previous row's `h`. If you get `y` values wrong, panels will overlap.

Example for two rows:
- Row 1 header: `y: 0, h: 1`
- Row 1 stat panels: `y: 1, h: 4`
- Row 2 header: `y: 5, h: 1`
- Row 2 time series: `y: 6, h: 8`
- Row 3 header: `y: 14, h: 1`

---

## GENERATION RULES

1. **Produce one complete JSON object per dashboard.** The output must be valid JSON — parseable by `JSON.parse()` with zero errors. No trailing commas. No comments.

2. **Every `expr` field must use exact metric names from the Prometheus payload.** Do not invent metric names. If the spec references a metric not in the payload, add a comment-equivalent: a disabled panel with the title prefixed with `[MISSING METRIC]`.

3. **Every target must have explicit `instant` and `range` fields.** Stat panels and table panels: `instant: true, range: false`. Time series and heatmap panels: `instant: false, range: true`.

4. **Every time series panel must have `"spanNulls": true`** to prevent disconnected dot rendering on sparse data.

5. **Validate threshold direction for every panel:**
   - Higher is better → base = red, thresholds escalate to green
   - Lower is better → base = green, thresholds escalate to red
   - State this in a code comment above each panel if producing the JSON iteratively.

6. **Every multi-value template variable must have `"allValue": ".*"`** to prevent broken "All" selections.

7. **Every PromQL query using a multi-value variable must use `=~` regex match**, not `=` equality.

8. **Assign unique `id` values to each panel**, incrementing from 1. Grafana will auto-reassign on import, but having unique IDs prevents import warnings.

9. **Include all overrides inline in the panel's `fieldConfig.overrides` array.** Do not rely on dashboard-level overrides.

10. **For dual Y-axis panels, always include overrides for:**
    - `custom.axisPlacement` → `"right"`
    - `custom.axisLabel` → human-readable label
    - `unit` → appropriate unit for that axis

11. **Set `"graphTooltip": 1`** (shared crosshair) at the dashboard level.

12. **Set `"schemaVersion": 39`** for Grafana 10 compatibility.

13. **Add dashboard tags** relevant to the use case, e.g.: `["migration", "pithos", "cm", "sre"]`.

14. **Add dashboard links** between the two dashboards (Status ↔ Processes) using:
```json
"links": [
  {
    "asDropdown": false,
    "icon": "external link",
    "includeVars": true,
    "keepTime": true,
    "tags": [],
    "targetBlank": true,
    "title": "Migration Processes Dashboard",
    "type": "link",
    "url": "/d/PROCESSES_DASHBOARD_UID"
  }
]
```
Set `"includeVars": true` and `"keepTime": true` so variable selections and time range carry over when switching dashboards.

15. **Output the JSON inside a code fence** with the suggested filename:
    - `migration-status-dashboard.json`
    - `migration-processes-dashboard.json`

---

## VALIDATION CHECKLIST

Before finalizing the JSON, verify:

```
□ JSON is valid (no trailing commas, no comments, proper quoting)
□ All metric names exist in the /actuator/prometheus payload
□ All label names in queries match actual labels in the payload
□ All template variables are defined in templating.list
□ All queries referencing multi-value variables use =~ not =
□ All stat/table panels use instant: true, range: false
□ All time series panels use instant: false, range: true
□ All time series panels have spanNulls: true
□ All table panels have format: "table" on targets
□ Threshold direction is correct for every panel (higher-is-better vs lower-is-better)
□ Grid positions (y values) are sequential with no overlaps
□ Every panel has a unique id
□ Dashboard UID is set and slugified
□ Datasource references use ${DS_PROMETHEUS} or the provided UID
□ Multi-value variables have allValue: ".*"
□ Overrides matcher options values match legendFormat strings exactly
□ Heatmap targets use format: "heatmap" and calculate: false
```

---

## OUTPUT

Produce the complete Grafana 10 JSON for the dashboard described in the spec. The JSON must be importable into Grafana 10 via Dashboard → Import → Paste JSON with zero manual fixes required.
