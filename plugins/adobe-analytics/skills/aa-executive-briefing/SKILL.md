---
name: aa-executive-briefing
description: >
  Generates a concise, executive-ready performance summary covering key metrics,
  trends, and what's driving movement. Use this skill when someone needs to
  produce a briefing, executive summary, performance narrative, or stakeholder
  readout — for example, "write an exec summary of last week's performance,"
  "create a performance briefing for our leadership team," "produce a monthly
  business review summary," "what should I tell executives about our metrics,"
  or "generate a performance narrative." Also trigger for "QBR summary,"
  "weekly business review," or "stakeholder briefing."
license: Apache-2.0
metadata:
  author: Adobe
  version: "1.0"
---

# Executive Briefing (Adobe Analytics)

Generate a concise, executive-ready performance summary with narrative
context, metric highlights, and key drivers. Designed for leadership
audiences — no raw data dumps, just clear signals and business implications.

> **Key parameter facts validated during implementation:**
> - `runReport` → `metricIds` (plural), not `metricId`; dates as `YYYY-MM-DDTHH:mm:ss`
> - `findMetrics` → `expansions` is a **required** parameter (use `"componentType"`)
> - `describeAa` → parameter is `guideType`, not `guide`; `REPORT_SUITE_CONTEXT_GUIDE`
>   may return empty for some report suites — skip gracefully and proceed
> - `metrics/uniquevisitors` is often unauthorized — use `metrics/visits` instead
> - `setSessionDefaults` is the correct tool name (not `setDefaultReportSuite`)

---

## AA MCP Tools Used

- `findReportSuites` — select report suite
- `setSessionDefaults` — set session context (reportSuiteId + globalCompanyId)
- `describeAa(guideType: "REPORT_SUITE_CONTEXT_GUIDE")` — load org context;
  note: may return no output for some report suites — proceed without it
- `findMetrics(expansions: "componentType")` — resolve metric IDs (expansions
  parameter is **required**)
- `listComponentUsage(componentType: "metric")` — identify most-used metrics
  if not specified by the user
- `runReport` — current and comparison period with all metrics batched in one
  call (`metricIds` accepts comma-separated IDs); dimension breakdowns for top movers
- `searchDimensionItems` — validate dimension values for driver callouts

---

## Phase 0 — Setup

1. Confirm report suite.
2. Load organizational context:

```
findReportSuites()
setSessionDefaults(reportSuiteId: "<rsid>", globalCompanyId: "<companyId>")
describeAa(guideType: "REPORT_SUITE_CONTEXT_GUIDE")   # may return empty — proceed if so
```

Use the context guide to understand:
- The organization's name and industry
- Which metrics are most used (top KPIs)
- **Calendar conventions and timezone.** Record these values — they are
  inputs to every date computation in Phase 1:
  - `WEEK_START_DOW` — day of week each week starts on. Default: **Monday**
    (ISO 8601).
  - `FISCAL_YEAR_START_MONTH` — month the fiscal year begins. Default:
    **January** (calendar year).
  - `TIMEZONE` — for example, `America/Los_Angeles`.
  - `CALENDAR_SOURCE` — one of `"context guide"`, `"default fallback"`, or
    `"user override"`.
- Any active report suite segments

---

## Phase 1 — Clarify the Briefing

Ask the user:

1. **Period** — "What time period should this briefing cover?"
   - Last week
   - Last month
   - Month-to-date (MTD)
   - This quarter / QTD
   - Last quarter
   - Custom date range

#### The principle

Two runs of this skill on the same report suite, period type, and prompt MUST
produce identical `current` and `comparison` date ranges. Determinism comes
from (a) reading calendar conventions from Phase 0 instead of improvising, and
(b) applying the alignment rule for the period type without taste calls.

#### Period type → alignment rule

Pick exactly one period type from the user's request:

| User request | `PERIOD_TYPE` | Current period | Comparison period |
|---|---|---|---|
| "last week" | `weekly` | Most recent full week ending before today, aligned to `WEEK_START_DOW` (exactly 7 days) | The week immediately before, same alignment |
| "this month" / "MTD" | `month-to-date` | 1st of current month → today | 1st of prior month → same day-of-month as today |
| "last month" | `monthly` | Prior full calendar month | The month before that |
| "this quarter" / "QTD" | `quarter-to-date` | Start of current fiscal quarter → today; fiscal quarters derived from `FISCAL_YEAR_START_MONTH` | Same days into the prior fiscal quarter |
| "last quarter" / "Q[N]" | `quarterly` | Prior full fiscal quarter | The fiscal quarter before that |
| Custom date range | `custom` | Use as specified | Equal-length window ending the day before `current.startDate` |

#### Universal invariants (must hold for every period type)

Before calling `runReport`, verify all six:

1. `current.startDate < current.endDate`
2. `comparison.startDate < comparison.endDate`
3. `comparison.endDate < current.startDate` (no overlap)
4. The day after `comparison.endDate` equals `current.startDate` (contiguous)
5. `current` and `comparison` have the **same length in days**
6. The alignment rule for `PERIOD_TYPE` is satisfied:
   - `weekly`: both `startDate`s fall on `WEEK_START_DOW`
   - `monthly`: both `startDate`s fall on the 1st of a month
   - `month-to-date`: both `startDate`s fall on the 1st; both `endDate`s have the same day-of-month
   - `quarterly`: both `startDate`s fall on the first day of a fiscal quarter
   - `quarter-to-date`: both `startDate`s fall on a fiscal quarter start; both `endDate`s are the same number of days into the quarter
   - `custom`: lengths match; contiguity holds

If ANY invariant fails, recompute the dates. **Never** paper over a mismatch by editing the footer.

#### Worked examples — today is Tuesday, May 26, 2026

Assumes `WEEK_START_DOW = Sunday` and `FISCAL_YEAR_START_MONTH = January`. Numbers change for other calendars — that's the point.

| User request | `PERIOD_TYPE` | Current | Comparison |
|---|---|---|---|
| "last week" | `weekly` | May 17 (Sun) – May 23 (Sat) | May 10 (Sun) – May 16 (Sat) |
| "this month" / "MTD" | `month-to-date` | May 1 – May 26 | Apr 1 – Apr 26 |
| "last month" | `monthly` | Apr 1 – Apr 30 | Mar 1 – Mar 31 |
| "this quarter" / "QTD" | `quarter-to-date` | Apr 1 – May 26 | Jan 1 – Feb 24 |
| "last quarter" | `quarterly` | Jan 1 – Mar 31 | Oct 1 – Dec 31 (2025) |
| Custom: "May 15–22" | `custom` | May 15 – May 22 (8 days) | May 7 – May 14 (8 days) |

If `WEEK_START_DOW = Monday` instead, the weekly row becomes `May 18 (Mon) – May 24 (Sun)` vs `May 11 (Mon) – May 17 (Sun)`. The other rows are unchanged.

#### A common AI failure mode

The AI may "know" from training data that weeks are Mon–Sun (ISO 8601) or that
quarters are Q1=Jan–Mar (calendar). Silently overriding the context guide with
those defaults is exactly the determinism bug this section exists to prevent.
The footer's methodology line MUST accurately describe the dates you computed
— if footer says "weeks start Sunday" but `current.startDate` is a Monday,
that's a bug to fix in the dates, not in the footer.

2. **Audience** — "Who is this briefing for?"
   - C-suite / board (highest level, fewest metrics, most context)
   - VP/Director level (more detail, some dimension context)
   - Marketing leadership (channel and campaign focus)
   Default: C-suite tone (concise, business-focused).

3. **North-star metric** — "What is the single most important metric for
   this briefing?" (e.g., revenue, conversions, retention rate)
   If not specified, use the most-used metric from `listComponentUsage`.

4. **Supporting metrics** — "What other metrics should be included? (up to 5)"

5. **Specific topic focus** — "Is there anything you want to highlight or
   investigate? (e.g., mobile performance, campaign results)"

---

## Phase 2 — Resolve Metrics

```
findMetrics(expansions: "componentType")  # expansions required
listComponentUsage(componentType: "metric")  # if metrics not specified
```

> **Important:** `findMetrics` requires the `expansions` parameter (use `"componentType"`
> as a safe default). Without it the call will fail.
>
> **Avoid `metrics/uniquevisitors`** — this metric is frequently restricted and returns
> an "unauthorized_metric" error. Prefer `metrics/visits` for audience size.
>
> **Reliable standard metrics for AA briefings:** `metrics/pageviews`, `metrics/visits`,
> `metrics/orders`, `metrics/revenue`, `metrics/bouncerate`, `metrics/occurrences`

### Deterministic selection — do not improvise

The KPI set MUST be reproducible across runs. Two runs of this skill on the same
report suite + period must produce the same metric values, which requires
selecting the **same metric IDs** every time. Follow this algorithm exactly:

1. If the user named metrics explicitly, resolve each to ONE specific metric ID
   via `findMetrics`. On ambiguity, pick the metric with the highest `usageCount`
   from `listComponentUsage` and document the choice.
2. Otherwise, take `listComponentUsage(componentType: "metric")` results, sort by
   `usageCount` descending with metric ID alphabetical as the tiebreak (stable
   secondary sort), and take the top **6 metric IDs**.
3. Use `describeMetric` to resolve to display names.
4. Do NOT cherry-pick by metric "type" (volume vs conversion vs revenue) and
   do NOT swap in an alternative metric because its name reads better in the
   narrative. Different runs picking `metrics/orders` vs a calculated metric
   also called "Orders" produce wildly different numbers and break trust.

### Always disclose the metric IDs used

Include a "Metrics included" line in the briefing artifact's footer listing
the resolved metric IDs. This makes the report auditable and lets the user
confirm a re-run is using the same metrics.

Record metric IDs and display names. Limit to 6 metrics total.

---

## Phase 3 — Fetch Performance Data

Batch all metrics into a single `runReport` call per period:

```
runReport(
  dimensionId: "variables/daterangeday",
  metricIds: "<metricId1>,<metricId2>,<metricId3>,...",
  startDate: "<YYYY-MM-DDTHH:mm:ss>",
  endDate:   "<YYYY-MM-DDTHH:mm:ss>"
)
```

> **Critical:** The `runReport` parameter is `metricIds` (plural), not `metricId`.
> It accepts comma-separated IDs — pass all metrics in one call.
> Dates must be ISO 8601 with time component: `2026-03-31T00:00:00` / `2026-04-06T23:59:59`.
> Use `variables/daterangeday` as `dimensionId` to get a day-by-day breakdown;
> totals for each metric are in `summaryData.totals[0]`, `totals[1]`, etc. in order.
> If a metric is unauthorized, it surfaces in `columnErrors` — the overall call still succeeds.

For 6 metrics: 2 calls total (current period + comparison period).

From each pair compute:
- Current total
- Prior total
- Absolute and percent delta
- Trend direction (up/flat/down)

---

## Phase 4 — Driver Context

For the north-star metric and any metric with a change > ±10%, run a
marketing channel breakdown to find what drove the movement:

```
runReport(
  metricIds: "<northStarMetricId>",
  dimensionId: "variables/marketingchannel",
  startDate: "<current period start>T00:00:00",
  endDate: "<current period end>T23:59:59",
  limit: 8
)
```

Run the same for the comparison period if needed to identify share shift.

Also check device type if the user mentioned mobile:

```
runReport(
  metricIds: "<northStarMetricId>",
  dimensionId: "variables/mobiledevicetype",
  startDate: "<current period start>T00:00:00",
  endDate: "<current period end>T23:59:59",
  limit: 5
)
```

---

## Phase 5 — Write the Narrative

Compose the executive briefing as a structured narrative, not a data table.
Tone: confident, clear, forward-looking. Avoid jargon. Use business language.

### Narrative structure

1. **Opening headline** — 1 sentence capturing the period's overall result.
   "Q2 performance was strong, with revenue growing 14% and conversion rates
   reaching a 12-month high."

2. **North-star metric** — 2–3 sentences on the most important KPI:
   value, change, context, what drove it.

3. **Supporting highlights** — 1 sentence per additional metric (positive
   first, then concerns).

4. **Key driver** — 2 sentences identifying what primarily caused movement.
   "Paid Search drove 60% of revenue growth, up from 48% in the prior period.
   Email performance also improved materially with a 22% lift in conversions."

5. **Risk or watch item** — 1–2 sentences on anything concerning:
   "Bounce rate on mobile increased 4pp, suggesting friction in the mobile
   experience worth investigating."

6. **Forward look / ask** — optional: what does this imply for next period?

---

## Phase 6 — Generate HTML Briefing Document

Build the executive briefing HTML and write to
`/tmp/aa_executive_briefing_<YYYY-MM-DD_HHMMSS>.html`.


### Rendering rules — apply consistently across runs

Two runs of this skill on the same report suite + period must render identically
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

A KPI tile must reflect what the report suite actually returned. The AI must
**not** silently substitute a different metric or hide a tile to make the
briefing look cleaner.

- **Both periods return 0 or NULL** for a metric in the resolved set: render
  the tile with `kpi-value` = `Data unavailable`, pill class `flat`, pill text
  `⚠ N/A`, and `prior` text = `Both periods returned no data — validate
  instrumentation`. The tile stays in the grid; do not omit it.
- **One period returns valid data, the other 0 / NULL**: render the tile with
  the valid value as `kpi-value`, pill class `flat`, pill text `⚠ N/A`, and
  `prior` text = `Prior {period_noun}: no data`.
- **Never** substitute a derived metric (e.g., adding "Conversion Rate"
  because Revenue came back $0). The visible KPI set MUST match the resolved
  metric IDs disclosed in the footer.
- If a Revenue / monetary metric is unavailable, surface a `.callout.note`
  above the KPI grid explaining the gap. Do not invent a value.


### HTML template

### Template variables

Populate every `{PLACEHOLDER}` in the HTML template below using these rules.
The briefing belongs to the customer — never substitute Adobe, AA, or any
vendor language into customer-visible fields.

- **`{ORG_NAME}`** — The customer's business or brand name, derived from the
  report suite context loaded in Phase 0. Strip technical/environment suffixes
  like ` — Prod`, ` - Demo`, ` Stage`, ` Test`, ` MCP`. If the report suite
  name has no clean brand label, fall back to the report suite display name
  with suffixes removed. Do **not** invent a name and do **not** substitute a
  vendor name.
- **`{PERIOD_TYPE}`** — One of `Weekly`, `Monthly`, `Quarterly`, or
  `Performance` (default), chosen from the period inferred in Phase 1.
- **`{LEDE_SENTENCE}`** — Use exactly this pattern, with no vendor names:
  `Leadership readout for the {period type lowercase} of {PERIOD_LABEL} compared to {COMPARISON_LABEL}.`
- **`{REPORT_SUITE}`** — Report suite display name. Suffixes are acceptable
  here — this row is the technical identifier line, not the title.
- **`{PERIOD_LABEL}`** / **`{COMPARISON_LABEL}`** — Human-readable date ranges,
  e.g., `May 12–18, 2026`.
- **`{GENERATED_DATE}`** — Today's date in the same human format.
- **`{METRIC_LABEL}` / `{FORMATTED_VALUE}` / `{PCT_CHANGE}` / `{PRIOR_VALUE}`** —
  Per-KPI values from Phase 3. Use the metric's customer-facing display name,
  not its internal ID.

### HTML Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Performance Briefing &mdash; {ORG_NAME} &mdash; {PERIOD_LABEL}</title>
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
              line-height: 1.05; margin-bottom: 14px; }
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

  /* === Callout === */
  .callout { background: var(--accent-red-soft);
             border-left: 4px solid var(--accent-red);
             border-radius: 6px;
             padding: 18px 22px; margin-bottom: 32px;
             display: flex; gap: 14px; align-items: flex-start; }
  .callout .warn-icon { font-size: 22px; line-height: 1; flex-shrink: 0;
                        color: var(--accent-yellow); }
  .callout-title { font-weight: 700; color: var(--accent-red);
                   margin-bottom: 4px; font-size: 15px; }
  .callout-body { font-size: 14px; color: #4a2222; line-height: 1.55; }
  .callout.note { background: #fdf6e7; border-left-color: var(--accent-yellow); }
  .callout.note .callout-title { color: #8a5a08; }
  .callout.note .callout-body { color: #5a4108; }

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

  /* === Narrative === */
  .narrative-card { background: var(--surface); border-radius: 8px;
                    padding: 32px 36px;
                    box-shadow: 0 1px 3px rgba(0,0,0,.04);
                    margin-bottom: 24px; }
  .narrative-card h2 { font-family: "Playfair Display", Georgia, serif;
                       font-size: 24px; font-weight: 700;
                       margin-bottom: 18px;
                       padding-bottom: 12px;
                       border-bottom: 1px solid var(--border); }
  .bullet-list { list-style: none; }
  .bullet-list li { padding: 14px 0; font-size: 15px; line-height: 1.65;
                    border-bottom: 1px solid #f1eeea; }
  .bullet-list li:last-child { border-bottom: none; }
  .bullet-list .signal { margin-right: 6px; }
  .bullet-list .metric-highlight { font-weight: 700; }
  .bullet-list .driver { color: var(--accent-red); font-style: italic; }

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

  footer { text-align: center; padding: 32px 24px;
           font-size: 12px; color: var(--ink-muted); }

  /* === Print === */
  @media print {
    nav { display: none; position: static; }
    header { padding: 36px 32px 28px; }
    header h1 { font-size: 42px; }
    .section-header { cursor: default; }
    .kpi-row { page-break-inside: avoid; }
    .kpi-tile, .narrative-card, .section {
      box-shadow: none; border: 1px solid var(--border);
    }
  }
</style>
</head>
<body>

<header>
  <div class="header-inner">
    <div class="eyebrow">{PERIOD_TYPE} Performance Briefing</div>
    <h1>{ORG_NAME} Performance</h1>
    <p class="lede">{LEDE_SENTENCE}</p>
    <div class="meta">
      <span><span class="icon">&#128197;</span> {PERIOD_LABEL}</span>
      <span><span class="icon">&#128202;</span> {REPORT_SUITE}</span>
      <span><span class="icon">&#128340;</span> Prepared {GENERATED_DATE}</span>
    </div>
  </div>
</header>

<nav>
  <a href="#highlights">Highlights</a>
  <a href="#narrative">Summary</a>
  <a href="#data">KPI Detail</a>
  <a href="#drivers">Drivers</a>
</nav>

<div class="container">

  <!--
    Optional critical callout. Include ONLY when the data has a
    notable finding worth raising above the fold (e.g., a metric
    moved >25% week-over-week, or data is missing/anomalous).
    Use the .note variant (yellow) for non-critical context;
    omit entirely if there is nothing noteworthy.
  -->
  <!--
  <div class="callout">
    <span class="warn-icon">&#9888;</span>
    <div>
      <div class="callout-title">Critical: {ONE_LINE_HEADLINE}</div>
      <div class="callout-body">{ONE_PARAGRAPH_CONTEXT}</div>
    </div>
  </div>
  -->

  <!-- Headline KPI Tiles -->
  <div class="section-label">Key Performance Indicators</div>
  <div id="highlights" class="kpi-row">
    <!-- Repeat for each KPI (4-6 tiles), class = up | down | flat:
    <div class="kpi-tile down">
      <div class="kpi-head">
        <div class="kpi-label">{METRIC_LABEL}</div>
      </div>
      <div class="kpi-value">{FORMATTED_VALUE}</div>
      <span class="pill down">&#9660; {PCT_CHANGE}%</span>
      <span class="prior">Prior week: {PRIOR_VALUE}</span>
    </div>
    -->
  </div>

  <!-- Executive Narrative -->
  <div id="narrative" class="narrative-card">
    <h2>Executive Summary</h2>
    <ul class="bullet-list">
      <!--
      <li>
        <span class="signal">{EMOJI}</span>
        <span class="metric-highlight">{Metric}:</span> {value} ({delta} vs prior period) &mdash;
        <span class="driver">{plain-English driver or context}</span>
      </li>
      -->
    </ul>
  </div>

  <!-- Supporting Data Table -->
  <div id="data" class="section">
    <div class="section-header" onclick="toggle('data-body')">
      <h2>Full KPI Detail</h2>
      <span id="data-body-icon">&#9662;</span>
    </div>
    <div id="data-body">
      <table>
        <thead><tr>
          <th>Metric</th>
          <th>{PERIOD_LABEL}</th>
          <th>{COMPARISON_LABEL}</th>
          <th>Change</th>
          <th>% Change</th>
          <th>Signal</th>
        </tr></thead>
        <tbody>
          <!-- One row per KPI with delta and badge -->
        </tbody>
      </table>
    </div>
  </div>

  <!-- Key Drivers -->
  <div id="drivers" class="section">
    <div class="section-header" onclick="toggle('drv-body')">
      <h2>What Drove the Movement</h2>
      <span id="drv-body-icon">&#9662;</span>
    </div>
    <div id="drv-body">
      <table>
        <thead><tr>
          <th>Metric</th>
          <th>Top Driver</th>
          <th>Value This Period</th>
          <th>Value Prior Period</th>
          <th>Contribution</th>
        </tr></thead>
        <tbody>
          <!-- One row per metric with top dimension driver -->
        </tbody>
      </table>
    </div>
  </div>

</div>

<footer>
  Performance Briefing &mdash; {ORG_NAME} &mdash; Generated {GENERATED_DATE}<br>
  <span style="opacity:.7">
    Methodology: {PERIOD_TYPE}
    &middot; current {CURRENT_START_ISO}&ndash;{CURRENT_END_ISO}
    &middot; comparison {COMPARISON_START_ISO}&ndash;{COMPARISON_END_ISO}
    &middot; week starts {WEEK_START_DOW}
    &middot; fiscal year starts {FISCAL_YEAR_START_MONTH}
    &middot; tz {TIMEZONE}
    &middot; calendar source: {CALENDAR_SOURCE}
    &middot; metrics: {METRIC_IDS_CSV}
  </span>
</footer>

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

**Section titles — no phase prefix**: Section headings in the HTML report must **not** include
the phase number. Use the plain section name only (e.g., "KPI Scorecards" not "Phase 2 — KPI Scorecards",
"Narrative" not "Phase 3 — Narrative", "Watch Items" not "Phase 5 — Watch Items").

Write to `/tmp/aa_executive_briefing_<YYYY-MM-DD_HHMMSS>.html` and open:

```bash
open /tmp/aa_executive_briefing_<YYYY-MM-DD_HHMMSS>.html
```

---

## Tone and Style Guidelines

| Audience | Tone | Metrics | Length |
|---|---|---|---|
| C-suite | High-level, business outcomes, 1-2 sentences per metric | 3–5 max | 1 page |
| VP/Director | Operational detail, channel context, trend narrative | 5–7 | 1.5 pages |
| Marketing leadership | Campaign and channel depth, attribution, segment | 5–8 | 2 pages |

Always:
- Lead with the positive, then address concerns.
- Express deltas as business impact, not just percentages ("revenue grew $42K,
  up 14%" not just "+14%").
- For metrics that went down, include a possible explanation or next step.

---

## Guardrails

- Do not present raw API output — always narrate the numbers.
- If a metric change is within ±3%, describe as "roughly flat" not a
  directional story.
- Always identify the comparison period explicitly in the briefing header.
- If context from `REPORT_SUITE_CONTEXT_GUIDE` identifies the company name
  or industry, use it in the narrative for a more personalized output.

---

## Example Interaction

> "Write an exec summary of last week's performance for our leadership team."

1. Confirm report suite; load context guide.
2. Identify period: last 7 days vs. prior 7 days. Audience: VP level.
3. Confirm metrics: revenue, visits, orders, conversion rate, bounce rate.
4. Run 2 reports (all 5 metrics batched in one call per period).
5. Run channel breakdown for revenue.
6. Compose narrative: "Revenue grew 9% WoW to $186K, led by Paid Search
   which contributed 58% of revenue. Visits grew 18%, but mobile bounce rate
   increased 4pp, signaling a checkout experience issue worth investigating."
7. Generate HTML briefing and open.
8. Deliver the narrative as inline text for immediate use.
