# OpTel Interpreter — Reference Context

## What Is OpTel Explorer?

Adobe rebranded RUM (Real User Monitoring) Explorer to Operational Telemetry Explorer in early 2025. The tool is available at `aem.live/tools/rum/explorer.html` and requires a domain key, which is provisioned when a site is onboarded to EDS. OpTel captures data from real user sessions — not synthetic tests — by injecting a lightweight sampling script into every EDS page. The data includes Core Web Vitals (LCP, CLS, INP), page view counts, traffic referrers, device types (mobile, desktop, tablet), browser types, geographic regions, and engagement signals.

## How OpTel Sampling Works

OpTel does not capture 100% of traffic. It uses a sampling rate that varies by site tier: typically 1-in-100 for high-traffic sites, with lower sampling ratios for smaller sites. This means the absolute numbers in OpTel are extrapolated — a page showing 5,000 views may have actually had 50 sampled sessions multiplied by the sampling rate. The relative proportions (e.g., "60% mobile, 40% desktop") are statistically valid, but small absolute numbers should be treated with caution. The sampling script adds negligible overhead (under 1KB, loaded in the delayed phase).

## CWV Thresholds in OpTel

OpTel follows Google's Core Web Vitals thresholds. For each metric, values are bucketed into three categories:
- **Good** (green): LCP < 2.5s, CLS < 0.1, INP < 200ms
- **Needs Improvement** (amber): LCP 2.5-4.0s, CLS 0.1-0.25, INP 200-500ms
- **Poor** (red): LCP > 4.0s, CLS > 0.25, INP > 500ms

The 75th percentile (p75) is the standard reporting percentile — this means 75% of user experiences are at or below the reported value. A p75 LCP of 2.8s means 75% of users see LCP at 2.8s or faster, but 25% see it slower.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| OpTel shows zero data for a domain | Domain key not configured or sampling not active | Verify the domain is onboarded to EDS and the OpTel script is present in the page source |
| Traffic numbers seem impossibly low | Sampling rate not accounted for | Multiply the raw count by the sampling ratio (typically 100x) to estimate actual traffic |
| CWV data is missing for Safari users | Safari does not fully support the Performance Observer API | Acknowledge the gap — CWV data is primarily from Chromium browsers; Safari metrics may be underrepresented |
| Metrics fluctuate wildly day to day | Low traffic volume produces noisy samples | Use a wider date range (7-30 days) to smooth out sampling variance |
| LCP shows "good" but pages feel slow | TTFB may be high, which is not captured separately in CWV | Check server response time independently using curl or WebPageTest |
