---
name: cja-dimension-analysis
description: >
  Comprehensive dimension analysis and reporting for CJA. Use this skill whenever the user
  wants to analyze one or more dimensions — including cardinality, distribution/skew, trends,
  anomalies, data quality errors, comparisons, and forecasting. Also trigger when someone
  asks "what are the top values for...", "dimension health", "explore this dimension",
  "dimension dashboard", "dimension statistics", "data quality check on a dimension",
  "dimension cardinality", "dimension trends", "dimension skew", "dimension anomalies",
  "compare dimensions", or any similar request to understand what's inside a CJA dimension.
  Produces an interactive HTML dashboard or a markdown report. Works with the CJA MCP server.
license: Apache-2.0
metadata:
  author: Adobe
  version: "1.0"
---
# CJA Dimension Analysis

Analyze one or more CJA dimensions to understand their cardinality, distribution, trends,
anomalies, data quality issues, and forecasts. Produces an actionable report that helps
teams understand what's inside their dimensions and where to focus attention.

## Why This Matters

Dimensions are the backbone of every CJA report — but most teams have never looked closely
at what's actually inside them. High-cardinality dimensions hurt query performance. Skewed
distributions mean a few values dominate reporting. Stale or error-laden dimensions
silently corrupt analysis. This skill surfaces those issues before they become problems.

## Workflow

Execute phases in order. Each phase is **selectable** — the user can ask for a subset
(e.g., "just cardinality and errors") or the full analysis. Default is all phases.

### Phase 0 — Setup

1. Call `findDataViews` to list available data views. If the user hasn't specified one,
   ask which data view to analyze. Set it with `setDefaultSessionDataViewId`.
2. Ask which dimensions to analyze. Options:
   - **Named dimensions**: "Analyze Page Name and Browser Type"
   - **By ID**: user provides dimension IDs directly
   - **All dimensions**: warn that this may be slow; ask for a limit (default: top 50 by name)
3. Ask which analyses to run (or confirm "all" as the default):
   - Cardinality, Distribution/Skew, Trends, Anomalies, Data Quality, Comparisons, Forecasting
4. Ask for the date range. If the user hasn't specified one, test a few ranges to find data:
   - Try last 30 days, last 90 days, last 6 months, last year — use the first that returns rows.
5. Ask for the primary metric to use for distribution/skew (default: occurrences or visits).
6. Confirm the plan with the user before proceeding.

### Phase 1 — Cardinality

For each dimension:

1. Call `searchDimensionItems(dimensionId, limit: 50000)` to estimate unique value count,
   or `runReport` with the dimension as rows and a count metric to get row count.
2. Classify cardinality:
   | Level | Threshold |
   |-------|-----------|
   | LOW | < 100 unique values |
   | MEDIUM | 100 – 1,000 |
   | HIGH | 1,000 – 10,000 |
   | VERY HIGH | > 10,000 |
3. Track cardinality over time (optional): `runReport` with dimension + date breakdown;
   count unique dimension values per day/week to see cardinality growth trend.
4. Flag HIGH and VERY HIGH dimensions with performance recommendations.

Store: `{dimensionId, name, uniqueValueCount, cardinalityLevel, cardinalityTrend}`

### Phase 2 — Distribution & Skew

For each dimension:

1. `runReport` with dimension as rows + primary metric (e.g., occurrences/visits).
   Request at least 50 rows to capture the distribution shape.
2. Compute:
   - **Top-N % share**: what % of total metric the top 1, 5, 10 values hold
   - **Gini coefficient**: measure of concentration (0 = perfectly flat, 1 = one value dominates)
   - **Cumulative distribution**: what % of metric is covered by top N values
3. Classify skew:
   | Label | Condition |
   |-------|-----------|
   | **Extreme skew** | Top 1 value > 50% of total |
   | **High skew** | Top 1 value > 30% of total |
   | **Moderate** | Top 5 values < 70% of total |
   | **Long tail** | Top 10 values < 50% of total |
4. Note: per-value breakdown with percentage and cumulative %.

Store: `{dimensionId, distribution: [{value, metric, pct, cumulative}], gini, skewLabel, top1Pct, top5Pct, top10Pct}`

### Phase 3 — Trends

For each dimension:

1. `runReport` with dimension + date granularity (day or week depending on range).
   Compare two periods: first half vs second half of the selected date range.
2. Identify:
   - **New values**: appeared in period 2 but not period 1
   - **Disappeared values**: present in period 1, absent in period 2
   - **Growth**: metric in period 2 > metric in period 1 by > 10%
   - **Decline**: metric in period 2 < metric in period 1 by > 10%
   - **Stable**: < 10% change between periods
3. Assign trend badges per value: 🟢 Growing | 🔴 Declining | 🟡 Stable | 🆕 New | ⬜ Disappeared

Store: `{dimensionId, periodComparison: {period1, period2, changes: [{value, p1Metric, p2Metric, pctChange, badge}]}, newValues: [], disappearedValues: []}`

### Phase 4 — Anomalies

For each dimension:

1. Use the time-series data from Phase 3. For each dimension value with enough data points,
   compute a rolling mean and standard deviation.
2. **Z-score detection**: flag any (value, date) pair where
   `z_score = |metric - rolling_mean| / rolling_std > threshold` (default: 2.0).
   Configurable sensitivity: 1.5 (sensitive), 2.0 (default), 3.0 (conservative).
3. **Threshold alerts**:
   - Any single value holding > 50% of total metric on a given day
   - Value count that is > 2× the rolling average for that value
4. **New/disappeared alerts**: flag values that appear or disappear mid-period (from Phase 3).
5. Collect: anomaly type (spike, drop, new, disappeared, threshold), dimension value, date, magnitude.

Store: `{dimensionId, anomalies: [{value, date, type, magnitude, zScore}]}`

### Phase 5 — Data Quality / Errors

For each dimension:

1. Search for known bad values using `searchDimensionItems`:
   - `"Unspecified"`, `"None"`, `"(empty)"`, `""`, `"null"`, `"undefined"`, `"N/A"`, `"unknown"`
2. Count occurrences with `runReport` filtering to each known bad value.
3. Compute: missing data % = (sum of bad value occurrences) / total occurrences.
4. Flag: dimensions where missing data > 5% (warning), > 20% (critical).
5. If the dimension has an expected format (URL, email, date), note it — but don't auto-validate
   patterns unless the user asks.

Store: `{dimensionId, errorPatterns: [{pattern, count, pct}], missingDataPct, missingDataSeverity}`

### Phase 6 — Comparisons (multi-dimension or time-period)

This phase runs when the user is analyzing 2+ dimensions OR requests period comparison.

**Side-by-side (2–3 dimensions):**
1. For each dimension pair, compare cardinality level, skew, top-5 values, error rate.
2. Produce a comparison table: dimension A vs B vs C on each metric.

**Time-period comparison (single dimension):**
1. Compare two custom date ranges provided by the user (or auto-detect: first half vs second half).
2. For each value: metric in period 1, metric in period 2, delta, % change.
3. Surface the biggest movers (top 5 growing, top 5 declining).

Store: `{comparisons: [{type, dimensions or periods, table}]}`

### Phase 7 — Forecasting

For each dimension with sufficient time-series data (>= 7 data points):

1. For the top 5–10 values by metric, fit a linear regression to the time series.
2. Project 7 periods forward.
3. Report:
   - Trend direction: Upward / Downward / Flat (based on slope)
   - Confidence: High (R² > 0.7), Medium (0.4–0.7), Low (< 0.4)
   - Projected value at end of forecast window
4. Flag values with strong upward trend (might become dominant) or strong downward trend
   (might disappear soon).

Store: `{dimensionId, forecasts: [{value, slope, r2, direction, confidence, projectedValues: []}]}`

### Phase 8 — Report Generation

After all analysis phases complete:

1. Save all collected data to a JSON file:
   `dimension_analysis_results_YYYY-MM-DD_HH-MM.json`
   (in a temp output directory, e.g. `/tmp/cja-dimension-analysis/`, or a path the user specifies)

2. Run the Python report generator:
   ```bash
   python3 scripts/cja_dimension_analysis.py \
     <analysis_json> \
     "<data_view_name>" \
     "<data_view_id>" \
     [output_directory] \
     [--format=html|markdown] \
     [--keep-analyses=N]
   ```

   **Options:**
   - `--format=html` (default): Interactive HTML dashboard with Chart.js visualizations
   - `--format=markdown`: Comprehensive text-based report with tables
   - `--keep-analyses=N` (default: 0 = keep all): Auto-cleanup of old analysis files

3. The script generates a second output file: the report (HTML or markdown).

4. Open with `open <output_directory>/dimension_analysis_report_*.html`
5. Present the report path to the user and summarize key findings:
   - Dimensions with HIGH/VERY HIGH cardinality
   - Dimensions with extreme or high skew
   - Any anomalies found
   - Data quality issues above warning threshold
   - Forecast trends worth watching

## CJA MCP Tools Used

| Tool | Phase | Purpose |
|------|-------|---------|
| `findDataViews` | 0 | List available data views |
| `setDefaultSessionDataViewId` | 0 | Set active data view for session |
| `findDimensions` | 0 | Discover dimensions by name/search |
| `describeDimension` | 0 | Get dimension metadata and ID |
| `searchDimensionItems` | 1, 5 | Count unique values; search for specific items (error patterns) |
| `runReport` | 1–7 | Primary data engine: dimension rows + metric, with optional date breakdown |

## Output Format

### HTML Dashboard (default)

Interactive report with:
- Executive summary cards (total dimensions, flagged dimensions, critical issues)
- Per-dimension sections: cardinality badge, distribution chart (Chart.js bar), skew metrics,
  trend table, anomaly list, data quality indicators
- Comparison section (if multiple dimensions or period comparison requested)
- Forecast section (if forecasting was run)
- Recommendations panel: grouped by priority (critical → warning → info)
- Design: dark navy-to-blue gradient header, full-width, card-based layout, collapsible sections

### Report HTML Style — Required

The generated HTML **must** use the editorial design system shared across all skills:
warm off-white surface, serif display title, red-on-black gradient header, and
underline-on-hover text-link nav. Do **not** introduce corporate-blue chrome,
centered headers, or alternative gradients.

Add Google Fonts to the document head:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Playfair+Display:wght@700;900&family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
```

```css
* { box-sizing: border-box; margin: 0; padding: 0; }
:root {
  --bg: #f5f4f1;
  --surface: #ffffff;
  --ink: #1a1a1a;
  --ink-muted: #6b6b6b;
  --border: #e5e2dc;
  --header-bg: #0e0e10;
  --header-warm: #3a1010;
  --accent-red: #c8312f;
  --accent-red-bright: #ff6b68;
  --accent-red-soft: #fdecea;
  --accent-green: #1f7a4d;
  --accent-yellow: #d4a017;
}
body { font-family: "Inter", -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
       background: var(--bg); color: var(--ink); line-height: 1.5;
       -webkit-font-smoothing: antialiased; }

/* === Header === */
header { background: linear-gradient(120deg, var(--header-bg) 0%, #1a0d0d 55%, var(--header-warm) 100%);
         color: #fff; padding: 56px 56px 44px; position: relative; overflow: hidden; }
header::after { content: ""; position: absolute; right: -140px; top: -140px;
                width: 460px; height: 460px;
                background: radial-gradient(circle, rgba(200,49,47,.35) 0%, transparent 70%);
                pointer-events: none; }
.header-inner { max-width: 1080px; margin: 0 auto; position: relative; z-index: 1; }
.eyebrow { display: inline-flex; align-items: center; gap: 8px;
           padding: 6px 14px; border: 1px solid rgba(255,107,104,.55);
           border-radius: 999px; color: var(--accent-red-bright);
           font-size: 11px; font-weight: 600; letter-spacing: 1.2px;
           text-transform: uppercase; margin-bottom: 24px;
           background: rgba(200,49,47,.10); }
.eyebrow::before { content: ""; width: 6px; height: 6px;
                   background: var(--accent-red-bright); border-radius: 50%; }
header h1 { font-family: "Playfair Display", Georgia, serif;
            font-size: 56px; font-weight: 700; letter-spacing: -1.5px;
            line-height: 1.05; margin-bottom: 14px; color: #fff; }
header .lede { font-size: 16px; max-width: 560px;
               color: rgba(255,255,255,.80); margin-bottom: 24px;
               line-height: 1.55; }
header .meta { display: flex; flex-wrap: wrap; gap: 22px;
               font-size: 13px; color: rgba(255,255,255,.60); }
header .meta span { display: inline-flex; align-items: center; gap: 6px; }
header .meta .icon { opacity: .8; }

/* === Tabs === */
nav { background: var(--surface); border-bottom: 1px solid var(--border);
      padding: 0 56px; display: flex; gap: 28px;
      position: sticky; top: 0; z-index: 50; }
nav a { display: block; padding: 16px 0; font-size: 14px;
        color: var(--ink); text-decoration: none;
        border-bottom: 2px solid transparent;
        transition: border-color .15s ease; }
nav a:hover { border-bottom-color: var(--accent-red); }

/* === Container === */
.container { max-width: 1080px; margin: 0 auto; padding: 36px 56px 60px; }

/* === Section (collapsible drill-down cards) === */
.section { background: var(--surface); border-radius: 8px;
           box-shadow: 0 1px 3px rgba(0,0,0,.04);
           margin-bottom: 22px; overflow: hidden; }
.section-header { padding: 18px 28px; border-bottom: 1px solid var(--border);
                  display: flex; justify-content: space-between;
                  align-items: center; cursor: pointer; }
.section-header h2 { font-family: "Playfair Display", Georgia, serif;
                     font-size: 18px; font-weight: 700; }

/* === Tables === */
table { width: 100%; border-collapse: collapse; font-size: 13px; }
thead th { background: #faf8f4; padding: 12px 22px;
           text-align: left; font-weight: 600;
           text-transform: uppercase; letter-spacing: .6px;
           font-size: 11px; color: var(--ink-muted);
           border-bottom: 1px solid var(--border); }
tbody td { padding: 12px 22px; border-bottom: 1px solid #f4f1eb; vertical-align: top; }
tbody tr:last-child td { border-bottom: none; }

/* === Badges (cardinality, skew, anomaly severity) === */
.badge { display: inline-block; padding: 3px 9px; border-radius: 4px;
         font-size: 11px; font-weight: 600; }
.badge.green  { background: #ebf5ef; color: var(--accent-green); }
.badge.red    { background: var(--accent-red-soft); color: var(--accent-red); }
.badge.yellow { background: #fef6e3; color: #b67a08; }
.badge.grey   { background: #f1efea; color: var(--ink-muted); }

footer { text-align: center; padding: 32px 24px;
         font-size: 12px; color: var(--ink-muted); }
```

**Header structure — eyebrow + serif h1 + lede + meta row:**
```html
<header>
  <div class="header-inner">
    <div class="eyebrow">Dimension Analysis</div>
    <h1>{ORG_NAME} Dimension Analysis</h1>
    <p class="lede">Cardinality, distribution, trends, anomalies, and data quality across {DIMENSION_COUNT} dimensions.</p>
    <div class="meta">
      <span><span class="icon">&#128197;</span> {DATE_RANGE}</span>
      <span><span class="icon">&#128202;</span> {DATA_VIEW_NAME}</span>
      <span><span class="icon">&#128340;</span> Prepared {DATE}</span>
    </div>
  </div>
</header>
```

Where `{ORG_NAME}` is the customer's brand name (with technical suffixes like
` — Prod`, ` - Demo`, ` MCP`, ` Stage` stripped). Never substitute a vendor or
product name into the title. The title is all white — do not color any word red.
For single-dimension reports, replace the h1 with `{ORG_NAME} {DIMENSION_NAME} Report`.

**Section titles — no phase prefix**: Section headings in the HTML report must **not** include
the phase number. Use the plain section name only:
- ✅ "Cardinality" — not "Phase 1 — Cardinality"
- ✅ "Distribution & Skew" — not "Phase 2 — Distribution & Skew"
- ✅ "Trends" — not "Phase 3 — Trends"
- ✅ "Data Quality" — not "Phase 5 — Data Quality / Errors"

### Markdown Report

Text-based report with:
- Summary table across all dimensions
- Per-dimension deep-dive sections with inline tables
- Anomaly log
- Recommendations with rationale

## Data Structure (analysis JSON)

The JSON file saved in Phase 8 has this structure, which the Python script reads:

```json
{
  "analysis_metadata": {
    "data_view_name": "...",
    "data_view_id": "...",
    "date_range": "...",
    "analyses_run": ["cardinality", "distribution", "trends", "anomalies", "errors", ...],
    "generated_at": "ISO8601"
  },
  "dimensions": [
    {
      "id": "variables/page",
      "name": "Page",
      "cardinality": { "uniqueValueCount": 1500, "level": "HIGH", "trend": null },
      "distribution": { "gini": 0.72, "skewLabel": "High skew", "top1Pct": 35.2, "topValues": [...] },
      "trends": { "period1": "...", "period2": "...", "changes": [...], "newValues": [], "disappearedValues": [] },
      "anomalies": [...],
      "errors": { "errorPatterns": [...], "missingDataPct": 3.2, "missingDataSeverity": "warning" },
      "forecasts": [...]
    }
  ],
  "comparisons": [...],
  "summary": {
    "totalDimensions": 5,
    "highCardinalityCount": 2,
    "criticalErrorCount": 1,
    "anomalyCount": 7
  }
}
```

## Example Interaction

> "Can you analyze how our 'Marketing Channel' dimension is performing and break it down by device type?"

1. **Setup:** Confirm the data view with `findDataViews`. Call `setDefaultSessionDataViewId`.
2. **Dimension discovery:** Call `findDimensions` to locate the 'Marketing Channel' dimension and its ID. Confirm it exists and has data with `searchDimensionItems`.
3. **Analysis:** Run `runReport` for Marketing Channel performance over the last 30 days (visits, conversions, revenue). Identify top and bottom performers.
4. **Breakdown:** Run a second report cross-tabbing Marketing Channel by Device Type dimension to surface mobile vs. desktop patterns.
5. **Report:** Run the Python analysis script to generate an interactive HTML report. Open it. Summarize top findings: "Email drives 38% of conversions despite only 12% of traffic. Paid Search converts 2× better on mobile than desktop."

## Important Guardrails

- Never modify dimension definitions or project data. This is read-only analysis.
- If a dimension returns no data for the selected date range, try a broader range before giving up.
- For VERY HIGH cardinality dimensions (> 50k values), note that full distribution analysis
  may be truncated — use sampled top-N values.
- If `runReport` times out on a dimension, reduce the row limit and note the limitation.
- Always tell the user which analyses are being run and which were skipped.
- For large dimension sets (> 20 dimensions), run phases 1–2 first and ask if the user wants
  to proceed with deeper analysis on a subset.
- Let the user know progress as you move through phases: "Phase 1 complete (cardinality for 5
  dimensions). Running Phase 2 (distribution)..."
