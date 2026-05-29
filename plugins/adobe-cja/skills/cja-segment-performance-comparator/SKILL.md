---
name: cja-segment-performance-comparator
description: >
  Compares the performance of two or more audience segments across key metrics
  side by side. Use this skill when someone wants to compare audiences, cohorts,
  or groups — for example, "how do mobile users compare to desktop users on
  conversion," "compare new vs. returning visitors," "show me the difference
  between these two segments," "compare these audiences on our KPIs," or
  "which segment performs better." Also trigger for "segment comparison,"
  "audience comparison," or "cohort comparison."
license: Apache-2.0
metadata:
  author: Adobe
  version: "1.0"
---

# Segment Performance Comparator (Customer Journey Analytics)

Compare 2–5 audience segments across a set of key metrics in a side-by-side
matrix. The output tells the user not just what each segment looks like in
isolation, but which segment wins or loses on each metric — and which
differences are large enough to act on.

This skill answers the question "which audience should we focus on?" with data.
Segment comparisons drive product decisions, personalization strategy, and
budget allocation — so clarity and actionability matter more than exhaustive data.

---

## CJA MCP Tools Used

- `findSegments` — search for segments by name or keyword
- `describeSegment` — understand the logic of candidate segments before using them
- `findMetrics` — resolve base metric IDs
- `findCalculatedMetrics` — include custom KPIs in the comparison
- `listComponentUsage` — identify the most-used metrics as default comparison set
- `runReport` (with `segmentIds` or `adhocSegments`) — pull metric values per segment

---

## Phase 0 — Setup

1. Call `findDataViews` to list available data views.
2. If the user hasn't specified a data view, present the list and ask which to use.
3. Call `setDefaultSessionDataViewId` with the chosen ID.
4. Ask the user which segments to compare if not already specified. Confirm the metrics to compare them on.

---

## Phase 1 — Identify Segments to Compare

### 1.1 From user description

If the user named specific segments, resolve them:
```
findSegments(search: "<segment name>")
```

For each match, call `describeSegment` to verify it is the correct one:
```
describeSegment(segmentId: "<id>")
```

Show the segment definition summary to the user if there is ambiguity:
> "I found two segments matching 'mobile users': **Mobile Visitors (All Devices)**
> and **Mobile App Users**. Which do you want to compare?"

### 1.2 From plain-English descriptions

If the user says "compare mobile vs desktop users" but there are no matching
segments, offer to create ad hoc segments inline for the comparison:
> "I don't see pre-built segments for mobile and desktop. I can create
> temporary ad hoc segments for this comparison using device type. Should I
> proceed with ad hoc segments, or would you like to create permanent segments
> first?"

Ad hoc segments are constructed using `adhocSegments` in `runReport` — no
save required for the comparison itself.

### 1.3 Segment count limit

Maximum 5 segments for a single comparison. More than 5 creates a matrix
that is too wide to read meaningfully. If the user requests more, say:
> "I'll limit to the 5 most relevant segments for readability. Would you like
> me to prioritize by usage count or stick with your list order?"

---

## Phase 2 — Identify Metrics to Compare

### 2.1 From user specification

Resolve named metrics via `findMetrics` and `findCalculatedMetrics`.

### 2.2 Default metric discovery

If the user did not specify metrics, pull the top metrics by usage. The
`listComponentUsage` tool does not support a `limit` parameter — it returns all
components ranked by usage count; take the top 6–8 from the result:
```
listComponentUsage(componentType: "metric")
listComponentUsage(componentType: "calculatedMetric")
```

Prefer calculated metrics over raw base metrics when they measure the same
thing — calculated metrics reflect intentional KPI definitions.

### 2.3 Metric selection for a comparison

Good comparison metrics should be meaningful across all segments. For example,
"Revenue" is meaningful for both mobile and desktop users; "App Installs" is
only meaningful for mobile. Remove metrics that would be trivially zero for
one segment.

If unsure, ask: "Should I use your standard KPI set, or focus on specific
metrics like conversion rate, revenue, and engagement?"

---

## Phase 3 — Run the Comparison

For each segment, run a `runReport` with that segment applied and all
comparison metrics included. Note that `runReport` takes `metricIds` as a
comma-separated string, `startDate`/`endDate` (not `dateRange`), and a
`dimensionIds` (required even for summary-only reports — use a low-cardinality
dimension like `variables/daterangeday` or `variables/web.webPageDetails.name`).
The summary totals for all metrics are in `summaryData.filteredTotals`:

```
runReport(
  dimensionIds: "variables/web.webPageDetails.name",
  metricIds: "metrics/visits,metrics/revenue_1,metrics/orders_1_1",
  startDate: "<period start>T00:00:00",
  endDate: "<period end>T23:59:59",
  page: 0,
  limit: 1,
  segmentIds: "<segment id>"
)
```

For ad hoc segments, use the full CJA segment definition object:
```
runReport(
  dimensionIds: "variables/web.webPageDetails.name",
  metricIds: "metrics/visits,metrics/orders_1_1",
  startDate: "<period start>T00:00:00",
  endDate: "<period end>T23:59:59",
  page: 0,
  limit: 1,
  adhocSegments: [{
    "func": "segment",
    "version": [1, 0, 0],
    "container": {
      "func": "container",
      "context": "visitors",
      "pred": {
        "func": "streq",
        "val": { "func": "attr", "name": "variables/device_type" },
        "str": "Mobile Phone"
      }
    }
  }]
)
```

Read metric totals from `summaryData.filteredTotals[i]` where `i` is the
0-based index of the metric in the `metricIds` string.

Run one report per segment. Collect all results into a matrix:
- Rows = metrics
- Columns = segments

---

## Phase 4 — Build the Comparison Matrix

For each cell (metric × segment):
- `value[metric][segment]` = raw metric value from `runReport`

For each metric row:
- `winner` = segment with the highest value (or lowest, for "lower is better" metrics)
- `loser`  = segment with the lowest value (or highest, for inverse metrics)
- `range`  = (max − min) / max × 100 — the spread across segments as a percentage
- `significant` = true if range > 10% (a meaningful difference worth acting on)

---

## Phase 5 — Generate HTML Comparison Report

Generate the report inline and write to
`/tmp/cja_segment_performance_comparator_report_<YYYY-MM-DD_HHMMSS>.html`.


### HTML Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Segment Comparison &mdash; {ORG_NAME} &mdash; {DATE_RANGE}</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Playfair+Display:wght@700;900&family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
<style>
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

  /* === Section label === */
  .section-label { font-size: 11px; font-weight: 700;
                   text-transform: uppercase; letter-spacing: 1.4px;
                   color: var(--ink-muted); margin-bottom: 14px;
                   padding-bottom: 10px; border-bottom: 1px solid var(--border); }

  /* === KPI grid === */
  .kpi-row { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
             gap: 14px; margin-bottom: 36px; }
  .kpi-tile { background: var(--surface); border-radius: 8px;
              padding: 22px 22px 20px;
              border-top: 3px solid #b9b6ae;
              box-shadow: 0 1px 3px rgba(0,0,0,.05); }
  .kpi-tile.down { border-top-color: var(--accent-red); }
  .kpi-tile.up   { border-top-color: var(--accent-green); }
  .kpi-tile.flat { border-top-color: #b9b6ae; }
  .kpi-head { display: flex; justify-content: space-between;
              align-items: center; margin-bottom: 10px; }
  .kpi-label { font-size: 11px; font-weight: 700;
               text-transform: uppercase;
               color: var(--ink-muted); letter-spacing: 1px; }
  .kpi-value { font-family: "Playfair Display", Georgia, serif;
               font-weight: 700; font-size: 38px;
               line-height: 1; color: var(--ink);
               margin-bottom: 12px; }
  .pill { display: inline-flex; align-items: center; gap: 4px;
          padding: 3px 9px; border-radius: 4px;
          font-size: 12px; font-weight: 600; line-height: 1.4; }
  .pill.down { background: var(--accent-red-soft); color: var(--accent-red); }
  .pill.up   { background: #ebf5ef; color: var(--accent-green); }
  .pill.flat { background: #f1efea; color: var(--ink-muted); }
  .prior { display: block; margin-top: 10px;
           font-size: 12px; color: var(--ink-muted); }

  /* === Sections (collapsible tables) === */
  .section { background: var(--surface); border-radius: 8px;
             box-shadow: 0 1px 3px rgba(0,0,0,.04);
             margin-bottom: 22px; overflow: hidden; }
  .section-header { padding: 18px 28px; border-bottom: 1px solid var(--border);
                    display: flex; justify-content: space-between;
                    align-items: center; cursor: pointer; }
  .section-header h2 { font-family: "Playfair Display", Georgia, serif;
                       font-size: 18px; font-weight: 700; }
  table { width: 100%; border-collapse: collapse; font-size: 13px; }
  thead th { background: #faf8f4; padding: 12px 22px;
             text-align: left; font-weight: 600;
             text-transform: uppercase; letter-spacing: .6px;
             font-size: 11px; color: var(--ink-muted);
             border-bottom: 1px solid var(--border); }
  tbody td { padding: 12px 22px; border-bottom: 1px solid #f4f1eb; }
  tbody tr:last-child td { border-bottom: none; }

  /* === Segment matrix table (centered columns, winner/loser cells) === */
  .matrix-table thead th { text-align: center; }
  .matrix-table thead th:first-child { text-align: left; }
  .matrix-table tbody td { text-align: center; }
  .matrix-table tbody td:first-child { text-align: left; font-weight: 600; }
  .cell-winner { background: #ebf5ef !important;
                 color: var(--accent-green); font-weight: 700; }
  .cell-loser  { background: var(--accent-red-soft) !important;
                 color: var(--accent-red); }

  .badge { display: inline-block; padding: 3px 9px; border-radius: 4px;
           font-size: 11px; font-weight: 600; }
  .badge.green  { background: #ebf5ef; color: var(--accent-green); }
  .badge.red    { background: var(--accent-red-soft); color: var(--accent-red); }
  .badge.yellow { background: #fef6e3; color: #b67a08; }
  .badge.grey   { background: #f1efea; color: var(--ink-muted); }

  .insight-box { background: var(--accent-red-soft);
                 border-left: 4px solid var(--accent-red);
                 border-radius: 6px; padding: 16px 20px;
                 margin: 14px 28px; }
  .insight-box p { font-size: 14px; line-height: 1.6; color: #4a2222; }

  .seg-legend { display: flex; gap: 18px; flex-wrap: wrap;
                padding: 18px 28px; border-bottom: 1px solid var(--border); }
  .seg-chip { display: flex; align-items: center; gap: 8px;
              font-size: 13px; font-weight: 600; color: var(--ink); }
  .seg-dot  { width: 12px; height: 12px; border-radius: 50%; }

  .back-top { position: fixed; bottom: 24px; right: 24px;
              background: var(--accent-red); color: #fff;
              width: 44px; height: 44px; border-radius: 50%;
              border: none; font-size: 20px; cursor: pointer;
              box-shadow: 0 4px 12px rgba(200,49,47,0.30); }
  footer { text-align: center; padding: 32px 24px;
           font-size: 12px; color: var(--ink-muted); }

  /* === Print === */
  @media print {
    nav { display: none; position: static; }
    header { padding: 36px 32px 28px; }
    header h1 { font-size: 42px; }
    .section-header { cursor: default; }
    .kpi-row { page-break-inside: avoid; }
    .kpi-tile, .section {
      box-shadow: none; border: 1px solid var(--border);
    }
    .back-top { display: none; }
  }
</style>
</head>
<body>

<header>
  <div class="header-inner">
    <div class="eyebrow">Segment Comparison Report</div>
    <h1>{ORG_NAME} Segment Comparison</h1>
    <p class="lede">Side-by-side performance for {SEGMENT_NAMES_SUMMARY} across {DATE_RANGE}, with winners and significant gaps surfaced.</p>
    <div class="meta">
      <span><span class="icon">&#128197;</span> {DATE_RANGE}</span>
      <span><span class="icon">&#128202;</span> {DATA_VIEW}</span>
      <span><span class="icon">&#128340;</span> Prepared {GENERATED_DATE}</span>
    </div>
  </div>
</header>

<nav>
  <a href="#overview">Overview</a>
  <a href="#matrix">Comparison Matrix</a>
  <a href="#insights">Insights</a>
</nav>

<div class="container">

  <!-- Summary KPI Tiles -->
  <div class="section-label">Comparison Summary</div>
  <div id="overview" class="kpi-row">
    <div class="kpi-tile flat">
      <div class="kpi-head"><div class="kpi-label">Segments Compared</div></div>
      <div class="kpi-value">{NUM_SEGMENTS}</div>
      <span class="prior">Audiences in scope</span>
    </div>
    <div class="kpi-tile flat">
      <div class="kpi-head"><div class="kpi-label">Metrics Evaluated</div></div>
      <div class="kpi-value">{NUM_METRICS}</div>
      <span class="prior">KPIs in the matrix</span>
    </div>
    <div class="kpi-tile down">
      <div class="kpi-head"><div class="kpi-label">Significant Differences</div></div>
      <div class="kpi-value">{NUM_SIGNIFICANT}</div>
      <span class="prior">Rows with &gt;10% spread</span>
    </div>
    <div class="kpi-tile up">
      <div class="kpi-head"><div class="kpi-label">Most Wins</div></div>
      <div class="kpi-value" style="font-size:22px;">{OVERALL_WINNER}</div>
      <span class="prior">Leading segment overall</span>
    </div>
  </div>

  <!-- Segment Legend -->
  <div class="section">
    <div class="seg-legend">
      <!-- For each segment:
      <div class="seg-chip">
        <div class="seg-dot" style="background: {COLOR}"></div>
        {SEGMENT_NAME} &mdash; {VISITOR_COUNT} visitors
      </div>
      -->
    </div>
  </div>

  <!-- Comparison Matrix -->
  <div id="matrix" class="section">
    <div class="section-header" onclick="toggle('matrix-body')">
      <h2>Metric Comparison Matrix</h2>
      <span id="matrix-body-icon">&#9662;</span>
    </div>
    <div id="matrix-body">
      <table class="matrix-table">
        <thead>
          <tr>
            <th>Metric</th>
            <!-- For each segment: <th>{SEGMENT_NAME}</th> -->
            <th>Winner</th>
            <th>Spread</th>
            <th>Significant?</th>
          </tr>
        </thead>
        <tbody>
          <!-- For each metric row:
          <tr>
            <td>{METRIC_NAME}</td>
            <!-- For each segment value:
            <td class="{cell-winner if winner, cell-loser if loser}">{VALUE}</td>
            -->
            <td><span class="badge green">{WINNER_SEGMENT}</span></td>
            <td>{SPREAD}%</td>
            <td>{YES/NO badge}</td>
          </tr>
          -->
        </tbody>
      </table>
    </div>
  </div>

  <!-- Narrative Insights -->
  <div id="insights" class="section">
    <div class="section-header" onclick="toggle('ins-body')">
      <h2>Key Insights</h2>
      <span id="ins-body-icon">&#9662;</span>
    </div>
    <div id="ins-body" style="padding: 0 0 14px;">
      <!-- 3-5 insight boxes, each covering one notable finding -->
      <!--
      <div class="insight-box">
        <p>{INSIGHT_TEXT}</p>
      </div>
      -->
    </div>
  </div>

</div>

<button class="back-top" onclick="window.scrollTo({top:0,behavior:'smooth'})">&uarr;</button>
<footer>Segment Comparison &mdash; {ORG_NAME} &mdash; Generated {GENERATED_DATE}</footer>

<script>
function toggle(id) {
  var el = document.getElementById(id);
  var ic = document.getElementById(id + '-icon');
  if (el.style.display === 'none') { el.style.display=''; ic.textContent='\u25be'; }
  else { el.style.display='none'; ic.textContent='\u25b8'; }
}
</script>
</body></html>
```

---

## Phase 6 — Narrative Insights

After building the matrix, generate 3–5 insight bullets for the Insights section:

1. **Overall Winner**: "Returning Visitors outperform New Visitors on 5 of 7
   metrics, with the largest gap in Revenue per Session (+82%)."
2. **Most Significant Difference**: "The biggest gap is Conversion Rate: Mobile
   converts at 1.2% vs Desktop at 3.8% — a 68% gap worth prioritizing."
3. **Surprising Parity**: "New vs Returning Visitors show nearly identical
   Bounce Rates (42% vs 44%), suggesting landing page quality is consistent."
4. **Actionable Signal**: "Paid Search visitors have 2.3× higher Revenue per
   Session than Direct visitors — consider shifting budget toward Paid Search."
5. **Anomaly**: "One segment shows near-zero values across all metrics — verify
   that the segment definition is correct and matches the current data view."

Insights should be plain English, not metric IDs. Name the specific segments
and metric values.

---

## Workflow Summary

1. Resolve 2–5 segments (by name or ad hoc definition).
2. Identify 5–8 comparison metrics (from user or top usage).
3. Run one `runReport` per segment with all metrics; collect results.
4. Build comparison matrix: rows = metrics, columns = segments.
5. Mark winner/loser per row; compute spread; flag significant differences.
6. Generate HTML report with matrix and insight bullets.
7. Write to `/tmp/cja_segment_performance_comparator_report_<YYYY-MM-DD_HHMMSS>.html`.
8. Open with `open /tmp/cja_segment_performance_comparator_report_<YYYY-MM-DD_HHMMSS>.html`.
9. Deliver inline summary: which segment wins overall, biggest gap metric,
   one actionable recommendation.

---

## Important Guardrails

- **Read-only analysis.** Never delete or modify segments or calculated metrics.
- **Always confirm segments before running.** Ambiguous segment names (e.g., "Mobile" could be several) should be resolved by showing the user the matched segment IDs and definitions.
- **Use the same date range for all segments.** Comparisons across different time windows are misleading.
- **Note overlap between segments.** If two segments share substantial audience overlap, note it — the "difference" may be exaggerated.
- **Cap the number of segments compared.** Comparing more than 5–6 segments in a single report makes the output unreadable; ask the user to prioritize.
- **Distinguish statistical significance from practical significance.** A 0.1% difference is rarely actionable — focus on differences of 5%+ unless the user specifies otherwise.

---

## Example Interaction

> "Compare our mobile vs. desktop segment performance for last quarter."

1. **Setup:** Confirm data view. Call `findDataViews`, user selects. Call `setDefaultSessionDataViewId`.
2. **Segment resolution:** Call `findSegments` to locate the "Mobile Users" and "Desktop Users" segments. Show matched names and IDs to confirm. User approves.
3. **Metrics:** Ask "Which metrics should I compare?" User: "Sessions, Conversion Rate, Revenue, and Average Order Value."
4. **Analysis:** Run `runReport` for Q1 2026 with both segments applied. Tabulate results side-by-side.
5. **Findings:** Mobile: 45% of sessions, 2.1% CVR, $0.84 RPV. Desktop: 55% of sessions, 4.8% CVR, $2.10 RPV. Desktop converts 2.3× better. Present a comparison table and 3 recommended next steps.
