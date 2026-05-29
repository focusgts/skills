---
name: aa-segment-performance-comparator
description: >
  Compares the performance of two or more audience segments across key metrics
  side by side. Use this skill when someone wants to compare audiences or
  visitor groups — for example, "how do mobile visitors compare to desktop on
  conversion," "compare new vs. returning visitors," "show me the difference
  between these two segments," "compare these audiences on our KPIs," or
  "which segment performs better." Also trigger for "segment comparison" or
  "audience comparison."
license: Apache-2.0
metadata:
  author: Adobe
  version: "1.0"
---

# Segment Performance Comparator (Adobe Analytics)

Compare the performance of two or more audience segments across key metrics
side by side to understand how different visitor groups behave. Uses direct
segment-vs-segment comparison to determine a winner, loser, and spread for
each metric, with a separate context panel showing segment sizing.

> **AA Call Budget:** AA's `runReport` accepts a single `segmentId` per call.
> For N segments × M metrics the comparison requires N×M calls, plus 1
> baseline call for the segment-size context panel. For 3 segments × 5
> metrics = 16 calls. Limit to 4 segments and 6 metrics for practical
> performance. Always confirm the segment/metric list with the user before
> starting.

---

## AA MCP Tools Used

- `findReportSuites` — select report suite
- `setSessionDefaults` — set session context (reportSuiteId + globalCompanyId)
- `findSegments` — discover and select comparison segments
- `findMetrics` — resolve metric IDs
- `runReport` — one call per segment per metric, plus one unsegmented call for sizing context

---

## Phase 0 — Setup

1. Confirm report suite with `findReportSuites` / `setSessionDefaults`.

```
findReportSuites(globalCompanyId: "<gcid>", page: 0, limit: 10)
setSessionDefaults(globalCompanyId: "<gcid>", reportSuiteId: "<rsid>")
```

---

## Phase 1 — Select Segments

Ask the user which segments to compare. If not specified, prompt:
> "Which visitor audiences would you like to compare? For example:
> Mobile vs. Desktop, New vs. Returning, Paid Search vs. Organic, or
> specific named segments from your library."

Search for and confirm each segment:

```
findSegments(page: 0, limit: 50)
# Filter locally by name. Built-in IDs: "Paid_Search", "Purchasers", "Return_Visits"
```

> **Note:** `findSegments` does not accept a `searchTerm` parameter. Retrieve all segments
> and filter by name locally. Built-in template segments have short IDs like "Paid_Search"
> that can be passed directly as `segmentIds` in `runReport`.

If the user requests a segment that doesn't exist by name, offer to build
it first using the aa-segment-builder skill, or suggest the closest existing
segment from search results.

Limit: 4 segments maximum per comparison. Advise this limit upfront.

---

## Phase 2 — Select Metrics

Ask the user which metrics to compare. Suggest a balanced mix:

- **Volume:** `metrics/visits`
- **Engagement:** `metrics/pageviews`, `metrics/bouncerate`,
  `metrics/pagespervisit`
- **Conversion:** `metrics/orders`, conversion rate calculated metric
- **Revenue:** `metrics/revenue`

Call `findMetrics` to resolve each metric ID:

```
findMetrics(expansions: "componentType,categories", page: 0, limit: 200)
# Filter locally by name. Key IDs: metrics/visits, metrics/revenue, metrics/orders, metrics/bouncerate
```

Limit: 6 metrics maximum. Confirm the final list with the user:
> "I'll compare these 3 segments across 5 metrics. This requires 16 report
> calls (3 segments × 5 metrics + 1 sizing call). OK to proceed?"

---

## Phase 3 — Select Date Range

Ask for or confirm the analysis period:
- Last 7 days (good for quick comparison)
- Last 30 days (recommended default)
- Last 90 days (for seasonal smoothing)
- Custom range

---

## Phase 4 — Run Comparison Reports

### 4.1 Segment sizing (context only)

Run a single unsegmented call for `metrics/visits` to get the total
population size, then one call per segment for `metrics/visits` to
compute each segment's share of total. These sizing values populate the
context panel — they are **not** used in the comparison matrix.

```
runReport(
  dimensionId: "variables/page",
  metricIds: "metrics/visits",
  startDate: "<start>",
  endDate: "<end>",
  limit: 1
)
# allVisitorVisits = summaryData.totals[0]
```

```
runReport(
  dimensionId: "variables/page",
  metricIds: "metrics/visits",
  segmentIds: "<segmentId>",
  startDate: "<start>",
  endDate: "<end>",
  limit: 1
)
# segmentVisits = summaryData.totals[0]; shareOfTotal = segmentVisits / allVisitorVisits × 100
```

> Reuse these results if `metrics/visits` is already a comparison metric.

### 4.2 Per segment per metric

For each segment × metric combination:

```
runReport(
  dimensionId: "variables/page",
  metricIds: "<metricId>",          # note: "metricIds" not "metricId"
  segmentIds: "<segmentId>",        # note: "segmentIds" not "segmentId"
  startDate: "<start>",
  endDate: "<end>",
  limit: 1
)
# Total = summaryData.totals[0]
```

> Read totals from `summaryData.totals[0]` (not `rows[]`). `dimensionId` is required — use any dimension with `limit: 1` for aggregate totals. Segment IDs are the raw `id` field from `findSegments`.

Track progress: "Fetching Segment 2 of 3, metric 3 of 5..."

---

## Phase 5 — Build the Comparison Matrix

The matrix compares segments directly to each other — no baseline column.

For each metric row, compute:

| Computed Value | Formula |
|---|---|
| Segment value | Raw from `runReport` |
| Winner | Segment with the best value for this metric |
| Loser | Segment with the worst value for this metric |
| Spread | (max − min) / max × 100 |
| Significant? | `true` if spread > 10% |

For metrics where lower is better (bounce rate, cost per acquisition),
invert the winner/loser logic — the segment with the **lowest** value
wins. Mark these metrics clearly in the report.

### 5.1 Segment profile summary

For each segment, compute an overall performance profile:
- **Wins:** count of metrics where this segment ranks #1
- **Losses:** count of metrics where this segment ranks last
- **Biggest edge:** metric where this segment outperforms others by the widest spread
- **Visits share:** percentage of total visits from the context panel

---

## Phase 6 — Generate HTML Comparison Report

Build the comparison report inline and write to
`/tmp/aa_segment_comparator_report_<YYYY-MM-DD_HHMMSS>.html`.


### HTML template

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Segment Comparison &mdash; {ORG_NAME}</title>
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
  .header-inner { max-width: 1150px; margin: 0 auto; position: relative; z-index: 1; }
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
  header .lede { font-size: 16px; max-width: 620px;
                 color: rgba(255,255,255,.80); margin-bottom: 24px;
                 line-height: 1.55; }
  header .meta { display: flex; flex-wrap: wrap; gap: 22px;
                 font-size: 13px; color: rgba(255,255,255,.60); }
  header .meta span { display: inline-flex; align-items: center; gap: 6px; }
  header .meta .icon { opacity: .8; }

  /* === Scroll-spy nav (underline-on-active text links) === */
  nav { background: var(--surface); border-bottom: 1px solid var(--border);
        padding: 0 56px; display: flex; gap: 28px;
        position: sticky; top: 0; z-index: 50; }
  nav a { display: block; padding: 16px 0; font-size: 14px;
          color: var(--ink); text-decoration: none;
          border-bottom: 2px solid transparent;
          transition: border-color .15s ease, color .15s ease; }
  nav a:hover { border-bottom-color: var(--accent-red); }
  nav a.active { border-bottom-color: var(--accent-red); color: var(--accent-red); }

  /* === Container === */
  .container { max-width: 1150px; margin: 0 auto; padding: 36px 56px 60px; }

  /* === Section label === */
  .section-label { font-size: 11px; font-weight: 700;
                   text-transform: uppercase; letter-spacing: 1.4px;
                   color: var(--ink-muted); margin-bottom: 14px;
                   padding-bottom: 10px; border-bottom: 1px solid var(--border); }

  /* === KPI summary cards === */
  .kpi-grid { display: grid;
              grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
              gap: 14px; margin-bottom: 28px; }
  .kpi-card { background: var(--surface); border-radius: 8px; padding: 22px;
              box-shadow: 0 1px 3px rgba(0,0,0,.05);
              border-top: 3px solid var(--accent-red); }
  .kpi-card .num { font-family: "Playfair Display", Georgia, serif;
                   font-size: 34px; font-weight: 700;
                   color: var(--ink); line-height: 1; }
  .kpi-card .lbl { font-size: 11px; color: var(--ink-muted); margin-top: 10px;
                   text-transform: uppercase; letter-spacing: 1px; font-weight: 700; }

  /* === Segment context panel === */
  .ctx-grid { display: grid;
              grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
              gap: 14px; margin-bottom: 32px; }
  .ctx-card { background: var(--surface); border-radius: 8px; padding: 22px;
              box-shadow: 0 1px 3px rgba(0,0,0,.05);
              border-top: 3px solid #b9b6ae; }
  .ctx-card h3 { font-size: 13px; font-weight: 700; margin-bottom: 12px;
                 text-transform: uppercase; letter-spacing: .8px;
                 color: var(--ink-muted); }
  .ctx-card .visits { font-family: "Playfair Display", Georgia, serif;
                      font-size: 26px; font-weight: 700; color: var(--ink);
                      line-height: 1; }
  .ctx-card .share  { font-size: 13px; color: var(--ink-muted); margin-top: 6px; }
  .ctx-card .wins   { font-size: 13px; color: var(--accent-green); font-weight: 600;
                      margin-top: 10px; }
  .ctx-card .edge   { font-size: 12px; color: var(--ink-muted); margin-top: 4px; }

  /* === Collapsible sections === */
  .section { background: var(--surface); border-radius: 8px; margin-bottom: 22px;
             box-shadow: 0 1px 3px rgba(0,0,0,.04); overflow: hidden; }
  .section-header { padding: 18px 28px; border-bottom: 1px solid var(--border);
                    display: flex; justify-content: space-between;
                    align-items: center; cursor: pointer; user-select: none; }
  .section-header h2 { font-family: "Playfair Display", Georgia, serif;
                       font-size: 18px; font-weight: 700; }
  .toggle { font-size: 16px; color: var(--accent-red); }

  /* === Comparison matrix table === */
  table { width: 100%; border-collapse: collapse; font-size: 13px; }
  thead th { background: #faf8f4; padding: 12px 18px; text-align: center;
             font-weight: 600; text-transform: uppercase; letter-spacing: .6px;
             font-size: 11px; color: var(--ink-muted);
             border-bottom: 1px solid var(--border); }
  thead th:first-child { text-align: left; }
  tbody td { padding: 12px 18px; border-bottom: 1px solid #f4f1eb;
             text-align: center; }
  tbody td:first-child { text-align: left; font-weight: 600; }
  tbody tr:last-child td { border-bottom: none; }
  .cell-winner { background: #ebf5ef !important; color: var(--accent-green); font-weight: 700; }
  .cell-loser  { background: var(--accent-red-soft) !important; color: var(--accent-red); }
  .badge { display: inline-block; padding: 3px 9px; border-radius: 4px;
           font-size: 11px; font-weight: 600; }
  .badge.green  { background: #ebf5ef; color: var(--accent-green); }
  .badge.red    { background: var(--accent-red-soft); color: var(--accent-red); }
  .badge.yellow { background: #fef6e3; color: #b67a08; }
  .badge.grey   { background: #f1efea; color: var(--ink-muted); }

  /* === Insight boxes === */
  .insight { background: var(--accent-red-soft); border-left: 4px solid var(--accent-red);
             border-radius: 6px; padding: 14px 20px;
             margin: 14px 28px; font-size: 14px; line-height: 1.6;
             color: #4a2222; }

  .back-top { position: fixed; bottom: 24px; right: 24px; background: var(--accent-red);
              color: #fff; width: 44px; height: 44px; border-radius: 50%; border: none;
              font-size: 20px; cursor: pointer;
              box-shadow: 0 4px 12px rgba(200,49,47,.35); }
  footer { text-align: center; padding: 32px 24px;
           font-size: 12px; color: var(--ink-muted); }

  /* === Print === */
  @media print {
    nav { display: none; position: static; }
    header { padding: 36px 32px 28px; }
    header h1 { font-size: 42px; }
    .section-header { cursor: default; }
    .back-top { display: none; }
    .kpi-card, .ctx-card, .section {
      box-shadow: none; border: 1px solid var(--border);
    }
  }
</style>
</head>
<body>

<header>
  <div class="header-inner">
    <div class="eyebrow">Segment Comparison Report</div>
    <h1>{ORG_NAME} Segment Comparison</h1>
    <p class="lede">Side-by-side comparison of audience segments across key metrics for {DATE_RANGE}.</p>
    <div class="meta">
      <span><span class="icon">&#128197;</span> {DATE_RANGE}</span>
      <span><span class="icon">&#128202;</span> {REPORT_SUITE}</span>
      <span><span class="icon">&#128340;</span> Prepared {GENERATED_DATE}</span>
    </div>
  </div>
</header>

<nav id="topnav">
  <a href="#overview" class="active">Overview</a>
  <a href="#matrix">Comparison Matrix</a>
  <a href="#insights">Key Insights</a>
</nav>

<div class="container">

  <!-- Overview KPI Cards -->
  <div id="overview">
    <div class="kpi-grid">
      <div class="kpi-card">
        <div class="num">{NUM_SEGMENTS}</div>
        <div class="lbl">Segments Compared</div>
      </div>
      <div class="kpi-card">
        <div class="num">{NUM_METRICS}</div>
        <div class="lbl">Metrics Evaluated</div>
      </div>
      <div class="kpi-card">
        <div class="num">{NUM_SIGNIFICANT}</div>
        <div class="lbl">Significant Differences</div>
      </div>
      <div class="kpi-card">
        <div class="num">{OVERALL_WINNER}</div>
        <div class="lbl">Most Wins</div>
      </div>
    </div>

    <!-- Segment Context Panel -->
    <div class="ctx-grid">
      <!-- One .ctx-card per segment:
           h3        = segment name
           .visits   = formatted visit count
           .share    = "38.7% of all visits"
           .wins     = "3 wins out of 5 metrics"
           .edge     = "Biggest edge: Conversion Rate (spread 68%)"
      -->
    </div>
  </div>

  <!-- Comparison Matrix -->
  <div id="matrix" class="section">
    <div class="section-header" onclick="toggle('mat')">
      <h2>Metric Comparison Matrix</h2>
      <span class="toggle" id="mat-icon">&#9662;</span>
    </div>
    <div id="mat">
      <table>
        <thead>
          <tr>
            <th>Metric</th>
            <!-- one <th> per segment -->
            <th>Winner</th>
            <th>Spread</th>
            <th>Significant?</th>
          </tr>
        </thead>
        <tbody>
          <!-- One row per metric.
               Winner cell gets class="cell-winner".
               Loser cell gets class="cell-loser".
               Winner column shows a .badge.green with the segment name.
               Spread column shows the % spread.
               Significant column shows a .badge.green "Yes" or .badge.grey "No".
          -->
        </tbody>
      </table>
    </div>
  </div>

  <!-- Key Insights -->
  <div id="insights" class="section">
    <div class="section-header" onclick="toggle('ins')">
      <h2>Key Insights</h2>
      <span class="toggle" id="ins-icon">&#9662;</span>
    </div>
    <div id="ins">
      <!-- 3-5 .insight divs: one per notable finding -->
    </div>
  </div>

</div>

<button class="back-top" onclick="window.scrollTo({top:0,behavior:'smooth'})">&#8679;</button>
<footer>Segment Comparison &mdash; {ORG_NAME} &mdash; Generated {GENERATED_DATE}</footer>

<script>
function toggle(id) {
  var el = document.getElementById(id);
  var ic = document.getElementById(id + '-icon');
  if (el.style.display === 'none') { el.style.display = ''; ic.textContent = '\u25be'; }
  else { el.style.display = 'none'; ic.textContent = '\u25b8'; }
}

(function() {
  var nav = document.getElementById('topnav');
  if (!nav) return;
  var links = nav.querySelectorAll('a[href^="#"]');
  var sections = [];
  links.forEach(function(a) {
    var target = document.querySelector(a.getAttribute('href'));
    if (target) sections.push({ link: a, el: target });
  });
  if (!sections.length) return;

  var docHeight = document.documentElement.scrollHeight;
  var winHeight = window.innerHeight;
  if (docHeight <= winHeight + 50) {
    nav.style.display = 'none';
    return;
  }

  window.addEventListener('scroll', function() {
    var scrollY = window.scrollY + 80;
    var current = sections[0].link;
    for (var i = 0; i < sections.length; i++) {
      if (sections[i].el.offsetTop <= scrollY) current = sections[i].link;
    }
    links.forEach(function(a) { a.classList.remove('active'); });
    current.classList.add('active');
  }, { passive: true });
})();
</script>

</body>
</html>
```

**Section titles — no phase prefix**: Section headings in the HTML report must **not** include
the phase number. Use the plain section name only (e.g., "Segment Comparison" not "Phase 2 — Segment Comparison",
"Metric Details" not "Phase 3 — Metric Details").

Write to `/tmp/aa_segment_comparator_report_<YYYY-MM-DD_HHMMSS>.html` and open:

```bash
open /tmp/aa_segment_comparator_report_<YYYY-MM-DD_HHMMSS>.html
```

---

## Inline Summary (Always Deliver)

Always follow the HTML report with a text summary:

```
Segment Comparison — [Date Range] | Report Suite: [Name]

Segment Context:  Mobile 48,200 visits (38.7%)  Desktop 72,400 (58.2%)

                  Mobile   Desktop   Winner     Spread
────────────────  ───────  ────────  ─────────  ──────
Visits            48,200   72,400    Desktop    33%
Bounce Rate       61.4%    40.1% ✓  Desktop    35%  ✦
Conversion Rate    1.2%     3.1% ✓  Desktop    61%  ✦
Revenue           $9,400   $31,200   Desktop    70%  ✦

✦ = spread > 10%  ✓ = winner

Key findings:
- Desktop converts 2.6× better (3.1% vs 1.2%). Prioritize mobile checkout.
- Paid Search (not shown) has highest CVR at 4.8% — most efficient channel.
```

---

## Guardrails

- Confirm segments and metrics with the user before starting — the call
  count is N×M and can grow quickly.
- For bounce rate and other "lower is better" metrics, invert the winner
  logic — the segment with the **lowest** value wins. Label these metrics
  clearly in the report (e.g., "↓ lower is better").
- If a segment returns very few visits (< 1,000), note that results may
  not be statistically reliable.
- Do **not** show baseline delta percentages (segment vs. All Visitors)
  in the comparison matrix. Segments are subsets of the total population,
  so count-metric deltas are always negative and misleading. Use the
  context panel for segment sizing instead.

---

## Example Interaction

> "Compare our mobile and desktop visitors on conversion metrics."

1. Confirm report suite.
2. Find segments: "Mobile Devices" and "Desktop" (or offer to create them).
3. Confirm metrics: visits, bounce rate, orders, conversion rate, revenue.
4. Date range: last 30 days.
5. Preview: "2 segments × 5 metrics + 1 sizing call = 11 reports. Proceed?"
6. Run all reports; announce progress.
7. Context: Mobile = 48.2k visits (38.7%), Desktop = 72.4k visits (58.2%).
8. Matrix: Desktop wins 3 of 5 metrics. Conversion rate spread 61%.
9. Generate HTML report and open.
10. Insight: "Desktop is your primary conversion engine. Mobile drives
    volume (39% of visits) but converts at 1.2% vs. Desktop's 3.1% — a 61% spread.
    Prioritize mobile checkout optimization for the biggest conversion lift opportunity."
