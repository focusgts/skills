---
name: cja-kpi-pulse
description: >
  Produces a compact KPI digest showing how key metrics changed over a period and
  what's driving the movement. Use this skill when someone asks for a performance
  summary, a weekly recap, a morning briefing, a KPI update, or any variation of
  "how did we do this week/month." Also trigger for requests like "give me a
  performance overview," "what moved in the last 7 days," "pull our KPI report,"
  or "summarize our metrics."
license: Apache-2.0
metadata:
  author: Adobe
  version: "1.0"
---

# KPI Pulse (Customer Journey Analytics)

Produce a compact KPI digest in under 2 minutes. The goal is a crisp answer to
"how did we do?" — not a deep-dive, not a data dump. Each KPI gets a scorecard
showing current value, period-over-period change, trend direction, and the top
dimension breakdown that explains any movement.

---

## CJA MCP Tools Used

- `describeCja(DATAVIEW_CONTEXT_GUIDE)` — understand the data view context
- `listComponentUsage` — find the most-used metrics (the org's real KPIs)
- `findMetrics` — resolve metric IDs from user-specified names
- `findCalculatedMetrics` — include custom KPIs if present
- `runReport` — pull metric values for current and prior periods
- `searchDimensionItems` — top dimension breakdown for movers

---

## Phase 0 — Setup

1. Call `findDataViews` to list available data views.
2. If the user hasn't specified a data view, present the list and ask which to use.
3. Call `setDefaultSessionDataViewId` with the chosen ID.
4. Call `describeCja("DATAVIEW_CONTEXT_GUIDE")` to load data view context.
   Record the data view's first-day-of-week as `WEEK_START_DOW` and timezone
   as `TIMEZONE`. If the context guide does not return a week-start value,
   default to **Monday** (ISO 8601). You will use both in Phase 1.1.
5. Clarify the monitoring scope: which KPIs to track and the comparison period (e.g., WoW, MoM, vs. target).

## Phase 1 — Clarify Scope

### 1.1 Determine the reporting period

If the user did not specify a period, ask one question:
> "What time window would you like? Options: last 7 days, last 30 days, this
> week vs last week, this month vs last month, or a custom range."

Default to **this week vs last week** if no answer is given.

Map the answer to two date ranges:
- **Period A** (current): e.g., "thisWeek", "thisMonth", last 7 days
- **Period B** (comparison): e.g., "lastWeek", "lastMonth", prior 7 days

**Calendar rule (mandatory):**

Use `WEEK_START_DOW` from Phase 0 to define what "week" means. The current
period (Period A) and the comparison period (Period B) MUST use the same
first-day-of-week — i.e., both periods' `startDate` fall on the same
day-of-week, both are exactly equal length, and the comparison period ends
immediately before the current period starts. Never mix conventions
(e.g., a Mon–Sun current with a Sun–Sat prior) within the same pulse run.
Pick the boundary once, then derive both periods from it. For custom date
ranges, compute Period B as the equal-length window ending immediately
before Period A starts.

**Sanity check before calling `runReport`:** confirm `periodA.startDate` and
`periodB.startDate` are the same day-of-week and that
`periodA.startDate - periodB.endDate == 1 day`. If not, recompute.

### 1.2 Determine the metrics

If the user named specific metrics, resolve them with `findMetrics` or
`findCalculatedMetrics`. Otherwise, discover the top 5–8 KPIs automatically:

```
listComponentUsage(componentType: "metric")
listComponentUsage(componentType: "calculatedMetric")
```

Note: `listComponentUsage` may return an empty list for data views with no
usage history. If it returns empty, fall back to:
```
findMetrics(searchQuery: "sessions visits revenue orders")
findMetrics(searchQuery: "page views cart conversion")
```
Pick the most business-relevant metrics from the results (sessions, orders,
revenue, product views, cart views, people — in that priority order).

Deduplicate: if a built-in metric and a calculated metric measure the same
thing, keep only the calculated metric (it's more intentional).

Final list: 5–8 metrics. More than 8 KPIs in a pulse report is noise.

---

## Phase 2 — Pull Current and Prior Period Data

Run a single `runReport` call per period with all KPI metrics included.
Use one call for Period A and one for Period B to minimize round-trips.
Use a summary dimension (e.g., `variables/daterangeday`) and limit: 1 to
get aggregate totals from `summaryData.totals` in the response.

```
runReport(
  dimensionIds: "variables/daterangeday",
  metricIds: "metrics/visits,metrics/visitors,metrics/orders_1_1,metrics/productListItems.priceTotal,metrics/cart_views",
  startDate: "<periodA start>T00:00:00",
  endDate: "<periodA end>T23:59:59",
  page: 0,
  limit: 1
)
```

```
runReport(
  dimensionIds: "variables/daterangeday",
  metricIds: "metrics/visits,metrics/visitors,metrics/orders_1_1,metrics/productListItems.priceTotal,metrics/cart_views",
  startDate: "<periodB start>T00:00:00",
  endDate: "<periodB end>T23:59:59",
  page: 0,
  limit: 1
)
```

Read aggregate totals from `summaryData.totals` (not row data), which
gives you the full-period sum for each metric in the order they were listed.

Capture for each metric:
- `valueA` (current period)
- `valueB` (comparison period)
- `delta` = valueA − valueB
- `pctChange` = (delta / valueB) × 100, rounded to 1 decimal

---

## Phase 3 — Classify Trends

For each KPI, assign a trend indicator:
- **↑ Up** if pctChange > +3%
- **↓ Down** if pctChange < −3%
- **→ Flat** if −3% ≤ pctChange ≤ +3%

Assign a signal color:
- For "higher is better" metrics: ↑ = green, ↓ = red, → = grey
- For "lower is better" metrics (bounce rate, error rate): ↑ = red, ↓ = green

---

## Phase 4 — Top Mover Drill-Down

For the 1–2 metrics with the largest absolute % change, find what's driving
the movement. Run a dimension breakdown for the current period:

```
runReport(
  dimensionIds: "variables/marketing_channel",
  metricIds: "<moving metric id>",
  startDate: "<periodA start>T00:00:00",
  endDate: "<periodA end>T23:59:59",
  page: 0,
  limit: 5
)
```

Note: Use `variables/marketing_channel` (not `variables/marketingchannel`) — 
verify the exact dimension ID with `findDimensions(searchQuery: "marketing channel")`
if unsure.

Compare dimension values between Period A and Period B to identify the top
contributor to the change. This becomes the "What drove it" entry in the report.

---

## Phase 5 — Generate HTML Report

Generate the KPI Pulse HTML report INLINE — do not use a Python script.
Build the HTML string directly from the collected data and output it as a
code block the user can save, or write it to `/tmp/cja_kpi_pulse_report_<YYYY-MM-DD_HHMMSS>.html`
using a one-line bash command.


### Rendering rules — apply consistently across runs

Two runs of this skill on the same data view + period must render identically
(modulo the generation timestamp). The rules below pin the formatting choices
that the AI would otherwise drift on.

#### Number formatting

- **KPI values** (the big number in each tile) — use full digits with
  thousands separators (`8,160`, `77,584`, `1,250,000`). Do **NOT** use SI
  suffixes like `K` or `M`, even for large values. Executives want exact
  numbers, not abbreviations.
- **Percent change** (in pills and narrative bullets) — always one decimal
  place, rounded **half-away-from-zero**. For example, `−23.55%` displays as
  `−23.6%`, never `−23.5%`. Compute on full-precision values; round only at
  display time.
- **Percentage-point change** (for already-percentage metrics like Conversion
  Rate or Bounce Rate) — same rounding, suffix `pp`. Example: `+0.40 pp`.
- **Currency** — `$` prefix with thousands separators and no decimals for
  values ≥ $100 (`$1,240,000`); cents only when value < $100 (`$45.20`).

#### Null / missing data handling

A KPI tile must reflect what the data view actually returned. The AI must
**not** silently substitute a different metric or hide a tile to make the
report look cleaner.

- **Both periods return 0 or NULL** for a KPI being rendered: render the tile
  with `kpi-value` = `Data unavailable`, pill class `flat`, pill text
  `⚠ N/A`, and `prior` text = `Both periods returned no data — validate
  instrumentation`. The tile stays in the grid; do not omit it.
- **One period returns valid data, the other 0 / NULL**: render the tile with
  the valid value as `kpi-value`, pill class `flat`, pill text `⚠ N/A`, and
  `prior` text = `Prior {period_noun}: no data`.
- **Never** substitute a derived metric (e.g., adding "Conversion Rate"
  because Revenue came back $0). The visible KPI set MUST match the metrics
  selected for this run.


### HTML Template

Use the following structure (fill in actual data):

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>KPI Pulse &mdash; {ORG_NAME} &mdash; {PERIOD_LABEL}</title>
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

  .badge { display: inline-block; padding: 3px 9px; border-radius: 4px;
           font-size: 11px; font-weight: 600; }
  .badge.green  { background: #ebf5ef; color: var(--accent-green); }
  .badge.red    { background: var(--accent-red-soft); color: var(--accent-red); }
  .badge.yellow { background: #fef6e3; color: #b67a08; }
  .badge.grey   { background: #f1efea; color: var(--ink-muted); }

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
    <div class="eyebrow">KPI Pulse Report</div>
    <h1>{ORG_NAME} KPI Pulse</h1>
    <p class="lede">Period-over-period snapshot of headline metrics for {PERIOD_LABEL} compared to {COMPARISON_LABEL}.</p>
    <div class="meta">
      <span><span class="icon">&#128197;</span> {PERIOD_LABEL}</span>
      <span><span class="icon">&#128202;</span> {DATA_VIEW}</span>
      <span><span class="icon">&#128340;</span> Prepared {GENERATED_DATE}</span>
    </div>
  </div>
</header>

<nav>
  <a href="#scorecards">Scorecards</a>
  <a href="#detail">Detail Table</a>
  <a href="#movers">Top Movers</a>
</nav>

<div class="container">

  <!-- KPI Tiles -->
  <div class="section-label">Key Performance Indicators</div>
  <div id="scorecards" class="kpi-row">
    <!-- Repeat one tile per KPI; use .up | .down | .flat modifier:
    <div class="kpi-tile up">
      <div class="kpi-head"><div class="kpi-label">{METRIC_NAME}</div></div>
      <div class="kpi-value">{FORMATTED_VALUE_A}</div>
      <span class="pill up">&#9650; {PCT_CHANGE}%</span>
      <span class="prior">Prior period: {FORMATTED_VALUE_B}</span>
    </div>
    -->
  </div>

  <!-- Detail Table -->
  <div id="detail" class="section">
    <div class="section-header" onclick="toggle('det-body')">
      <h2>Full KPI Detail</h2>
      <span id="det-body-icon">&#9662;</span>
    </div>
    <div id="det-body">
      <table>
        <thead><tr>
          <th>Metric</th>
          <th>{PERIOD_LABEL}</th>
          <th>{COMPARISON_LABEL}</th>
          <th>Delta</th>
          <th>% Change</th>
          <th>Trend</th>
        </tr></thead>
        <tbody>
          <!-- One row per metric -->
          <!--
          <tr>
            <td>{METRIC_NAME}</td>
            <td>{VALUE_A}</td>
            <td>{VALUE_B}</td>
            <td>{DELTA}</td>
            <td><span class="badge {green|red|grey}">{PCT_CHANGE}%</span></td>
            <td>{ARROW}</td>
          </tr>
          -->
        </tbody>
      </table>
    </div>
  </div>

  <!-- Top Movers -->
  <div id="movers" class="section">
    <div class="section-header" onclick="toggle('mov-body')">
      <h2>What Drove the Movement</h2>
      <span id="mov-body-icon">&#9662;</span>
    </div>
    <div id="mov-body">
      <table>
        <thead><tr>
          <th>Metric</th>
          <th>Driver Dimension</th>
          <th>Top Value</th>
          <th>Contribution</th>
          <th>Context</th>
        </tr></thead>
        <tbody>
          <!-- Fill with dimension breakdown data -->
        </tbody>
      </table>
    </div>
  </div>

</div>

<button class="back-top" onclick="window.scrollTo({top:0,behavior:'smooth'})">&uarr;</button>
<footer>KPI Pulse &mdash; {ORG_NAME} &mdash; Generated {GENERATED_DATE}</footer>

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

## Phase 6 — Deliver the Report

After generating the HTML:
1. Write it to `/tmp/cja_kpi_pulse_report_<YYYY-MM-DD_HHMMSS>.html`
2. Open with `open /tmp/cja_kpi_pulse_report_<YYYY-MM-DD_HHMMSS>.html`
3. Provide a 3–5 line text summary inline in the chat:

```
KPI Pulse — This Week vs Last Week

↑ Revenue: $1.24M (+8.2%)  — Paid Search drove most of the gain
↓ Conversion Rate: 2.1% (−0.4pp) — Drop in mobile checkout
→ Sessions: 540K (+1.1%)  — Flat week-over-week
↑ Orders: 11,340 (+6.7%)  — Product page improvements appear to be working
↓ Bounce Rate: 43.2% (+2.1pp) — Worth monitoring next week
```

The text summary gives immediate value even without opening the HTML file.

---

## Important Guardrails

- **Read-only monitoring.** Never modify metrics, segments, or projects.
- **Use consistent date ranges.** Week-over-week and month-over-month comparisons must use equal-length periods.
- **Flag anomalies, don't diagnose them.** The pulse report surfaces significant deviations — deep root cause analysis belongs in the anomaly triage skill.
- **Respect business calendar.** Holiday periods, campaigns, and seasonal patterns affect normal variance — note context when flagging anomalies.
- **Cap metric count.** Monitor up to 10–15 KPIs per pulse; more than that dilutes focus. Ask the user to prioritize if they specify too many.
- **Note data freshness.** If the most recent data point is older than expected, warn the user before presenting the pulse.

## Example Interaction

> "Give me a quick pulse on our key metrics for this week."

1. **Setup:** Confirm data view with `findDataViews`. User selects their main data view. Call `setDefaultSessionDataViewId`.
2. **Scope:** Ask "Which KPIs should I include?" User says: "Sessions, Revenue, Conversion Rate, and Average Order Value."
3. **Data pull:** Run `runReport` for current week vs. prior week for all four metrics.
4. **Analysis:** Sessions +8% WoW (within normal range). Revenue +3% WoW. Conversion Rate -12% WoW — flagged as anomalous. AOV +17% WoW — notable positive.
5. **Summary:** Present a KPI scorecard with traffic-light status (green/yellow/red), highlight the Conversion Rate drop as needing investigation, and note that the AOV increase partially offsets it.

## Error Handling

- If `runReport` returns no data for Period B (comparison is too far in the
  past or data view lacks history), show "N/A" for the delta and flag it with
  a grey badge.
- If a metric returns null, display "—" rather than 0 to avoid false
  impressions of zero performance.
- If fewer than 3 metrics are available, warn the user that the pulse may be
  incomplete and suggest they verify the data view is correctly configured.
