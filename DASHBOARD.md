# Capybara Care — Revenue Dashboard

A single-page HTML dashboard that pulls session and payment data from two Google Sheets tabs and presents it across five views: Home, Monthly Trend, Therapist, Customers, and Insights.

---

## Building / Publishing

Edit `dashboard_src.html` (unencrypted source — gitignored). Then encrypt and publish:

```bash
npx staticrypt dashboard_src.html -p YOUR_PASSWORD --short && cp encrypted/dashboard_src.html dashboard.html
```

- `dashboard_src.html` → source file to edit (unencrypted, gitignored)
- `encrypted/dashboard_src.html` → staticrypt output (intermediate, don't use directly)
- `dashboard.html` → the file that is actually served / committed

**Important:** Never use `-d .` — that outputs to the current directory and **overwrites `dashboard_src.html`** with the encrypted version, destroying the source.

The `-o` flag does **not** exist in this version of staticrypt — always use the default `-d` (outputs to `encrypted/`) and copy manually.

**To recover the source if it gets overwritten:**
```bash
npx staticrypt --decrypt dashboard_src.html -p YOUR_PASSWORD -d decrypted
cp decrypted/dashboard_src.html dashboard_src.html
```

---

## Data Source

Data is fetched via the Google Visualization JSONP API from a single Google Spreadsheet split across two sheets:

| Sheet | Date range |
|---|---|
| `DataCopy` | Up to March 2026 |
| `DataCopy 2026` | April 2026 onwards |

Both sheets must be publicly shared (Anyone with the link → Viewer). On load the dashboard merges both sheets, deduplicating by date boundary, so there is no double-counting at the cutover point.

**Session rows** are defined as rows where `You Gave > 0`. Payment-only rows (where `You Got` has a value but `You Gave` is 0) are kept in a separate raw dataset so that outstanding balances are computed correctly.

---

## Global Controls

### Filters bar
Four dropdowns that apply across every tab simultaneously:

| Filter | Field | Affects Insights tab? |
|---|---|---|
| Year | `Year` column | Yes |
| Month | `Month` column | No — Insights always uses full history |
| Therapist | `Therapist` column | No |
| Customer | `Customer Name` column | No |

On initial load the dashboard auto-selects the current calendar month and year. Changing any dropdown immediately re-renders all tabs and charts. The Insights tab respects only the Year filter so that historical context is always meaningful.

### Refresh button
Re-fetches both sheets from scratch and resets the last-updated timestamp shown in the header.

### Last Updated
Displays the time of the most recent successful data load (HH:MM format, Indian locale).

---

## Tabs

### 1. Home

#### KPI Cards
Six summary metrics derived from the currently filtered session rows:

| Card | Calculation |
|---|---|
| **Total Revenue** | Sum of `You Gave` |
| **Net Revenue** | Total Revenue − Therapist Payout |
| **Total Sessions** | Count of rows where `You Gave > 0` |
| **Therapist Payout** | Sum of `Share` |
| **Avg per Session** | Total Revenue ÷ Total Sessions |
| **Outstanding** | Total Revenue − Sum of `You Got` (all raw rows, not just session rows) |

KPI values display in compact format (e.g. ₹1.25L, ₹4.50K). Hovering over a value shows the exact amount as a tooltip.

#### All Transactions Table
Full list of session rows matching the active filters.

**Columns:** Date · Customer · Details · Therapist · You Gave (₹) · Payment Received (₹) · Therapist Share (₹) · Month

**Features:**
- **Search** — full-text search across all fields; resets to page 1 on each keystroke.
- **Column sort** — click any column header to sort ascending; click again to toggle descending. Active sort column shows a ▲/▼ indicator.
- **Pagination** — Previous / Next buttons with a page-of-total counter. Page size selector: 20 / 50 / 100 / 200 rows per page.
- **Share View** — opens a print-ready modal (see below).

#### Share View Modal
Opens a compact two-column session list suitable for printing or saving as PDF.

- Header shows the active filter context (therapist · month/year · customer) and total session count plus total therapist share due.
- Sessions are arranged in two columns reading top-to-bottom (left column rows 1–⌈N/2⌉, right column the remainder).
- Alternating row shading for readability.
- **Print / PDF** button triggers `window.print()`. A print stylesheet hides everything except the modal for a clean printed output.
- Close by clicking × or clicking outside the modal box.

---

### 2. Monthly Trend

Breaks down revenue, profit, and session count within the filtered period.

#### Toolbar
- **Day View / Week View** toggle — switches the x-axis granularity between individual days of the month and four-week buckets (W1 1–7, W2 8–14, W3 15–21, W4 22–28, W5 29+).
- A hint line shows the current period and any active therapist/customer filters.

#### Stat Pills
Two summary cards update automatically when the view is toggled:

| Pill | Calculation |
|---|---|
| Avg Revenue / Day (or Week) | Total Revenue ÷ number of active buckets |
| Avg Profit / Day (or Week) | Total Profit ÷ number of active buckets |

#### Revenue, Profit & Sessions Chart
Stacked bar + line combo chart (Chart.js):
- **Profit** (dark green bars, bottom stack) = Revenue − Therapist Share per bucket.
- **Therapist Share** (light green bars, top stack) = Share per bucket. Together both bars equal total revenue.
- **Sessions** (amber line, right y-axis) = count of sessions per bucket.
- Tooltip shows Profit, Therapist Share, and a footer summing them back to total Revenue.

---

### 3. Therapist

Per-therapist performance overview for the filtered data.

#### KPI Cards (dynamic)
One card per therapist, sorted by total revenue descending. Each card shows:
- Therapist name
- Total revenue (compact format)
- Session count · therapist share · net profit

#### Revenue by Therapist Chart
Grouped stacked bar chart. Each therapist occupies its own stack group, split into:
- **Net Profit** (solid color)
- **Therapist Share** (same color, semi-transparent)

X-axis = month/year periods sorted chronologically. Tooltip shows the exact ₹ value for each segment.

#### Monthly Sessions by Therapist Chart
Multi-line chart with one line per therapist. X-axis = month/year periods. Y-axis = session count. Each therapist uses a distinct color from a 10-color green-to-teal palette.

---

### 4. Customers

Customer-level revenue and balance tracking.

#### Top Customers Chart
Horizontal bar + line combo showing the top 15 customers by revenue:
- **Revenue (₹)** — green bars, left y-axis.
- **Sessions** — dark line with points, right y-axis.

#### Customer Balances Table
All customers (not just top 15) in a sortable, searchable table.

**Columns:** Customer · Sessions · Revenue (₹) · Payments Received (₹) · Balance (₹) · Status

**Balance logic:**
- `Balance = Revenue − Payments Received`
- Positive balance → customer owes money → red **Due ₹X** badge.
- Zero balance → green **Settled** badge.
- Negative balance → customer has credit → green **Advance ₹X** badge.

**Features:**
- **Search** — filters by customer name; resets to page 1.
- **Column sort** — click any header to sort; defaults to revenue descending.
- **Pagination** — same prev/next + page-size controls as the main table.

---

### 5. Insights

An automated analysis engine that scans all historical data and surfaces actionable findings ranked by urgency. Insights always use the full raw dataset (year-filtered only) so results are never skewed by month or therapist filters.

#### Header
Shows total insight count ("N insights as of today") and a "Last checked: HH:MM" timestamp. Re-runs every time data is refreshed or the year filter changes.

#### Insight Cards
Each card has a coloured left border and badge based on priority:

| Priority | Colour | Meaning |
|---|---|---|
| 1 — URGENT | Red | Needs action today |
| 2 — WATCH | Amber | Review this week |
| 3 — POSITIVE | Green | Good news / momentum |
| 4 — INFO | Grey | Context and trends |

Card anatomy:
- **Category badge** (URGENT / WATCH / POSITIVE / INFO)
- **`(i) rule_id` tooltip** — shows which rule generated the card (for debugging)
- **Title** — short headline with specific names and amounts
- **Detail** — one or two sentences answering "so what should I do?"
- **Metric** — optional large number (e.g. ₹8.1K, 14 days)
- **Action button** — optional CTA (e.g. "Message Sarvag →")

Cards are sorted priority 1 → 4, capped at `maxInsightsToShow` (default 12).

If no rules fire: shows "All clear — no issues detected today 🎉".

#### Insights Engine

Implemented as a flat `INSIGHT_RULES` array. Each rule is an independent object — enable/disable by flipping `enabled: true/false` without touching any other code.

**`INSIGHT_CONFIG`** — a single config object at the top of the engine holds every tunable threshold:

| Key | Default | Purpose |
|---|---|---|
| `outstandingUrgentThreshold` | 5000 | ₹ owed before high-balance alert |
| `advanceWarningThreshold` | 2000 | ₹ advance remaining before warning |
| `dropoutGapDays` | 10 | Days of no sessions → dropout watch |
| `dropoutUrgentDays` | 14 | Days of no sessions → urgent |
| `newClientDays` | 45 | Days since first session = "new client" |
| `newClientMinSessions` | 4 | Sessions below this = low engagement |
| `revenueDropPercent` | 30 | Week-over-week drop threshold (%) |
| `revenueRisePercent` | 20 | Week-over-week rise threshold (%) |
| `concentrationRiskPercent` | 40 | Top-3 clients revenue share before risk flag |
| `therapistDropPercent` | 30 | Projected sessions drop vs monthly avg |
| `highLifetimeValue` | 50000 | ₹ lifetime billing milestone |
| `unusualSessionMultiple` | 3 | Nx avg rate = unusual session value |
| `dataFreshnessDays` | 2 | Days since last row before stale warning |
| `returningClientGapDays` | 21 | Gap (days) that counts as a "return" |
| `maxInsightsToShow` | 12 | Cap on total rendered cards |
| `weeksOfHistoryForAverage` | 8 | Weeks of data used for rolling averages |
| `largePaymentThreshold` | 8000 | ₹ You Got entry = "large payment" |
| `loyalClientMonths` | 4 | Months span required for loyal-client badge |
| `loyalClientMaxGapDays` | 21 | Max gap allowed in loyal-client check |

#### Rules Reference

**Group 1 — Outstanding & Payments**

| Rule ID | Priority | Condition |
|---|---|---|
| `high_outstanding_balance` | URGENT | Any client owes > ₹5,000 |
| `advance_expiring_soon` | URGENT | Client advance < ₹2,000 remaining |
| `multiple_outstanding_clients` | WATCH | More than 3 clients owe money this month |
| `large_single_payment_received` | POSITIVE | Single `You Got` entry > ₹8,000 in last 7 days |

**Group 2 — Session Frequency & Dropout Risk**

| Rule ID | Priority | Condition |
|---|---|---|
| `client_missing_expected_sessions` | WATCH | Active client's last-2-week sessions < 50% of their weekly average |
| `client_no_show_streak` | URGENT / WATCH | 0 sessions in last 10 days but had sessions in the 10 days before |
| `new_client_low_engagement` | WATCH | First session in last 45 days and fewer than 4 total sessions |
| `multiple_dropout_risks` | WATCH | More than 2 clients simultaneously flagged as dropout risk |

**Group 3 — Revenue Pace & Trends**

| Rule ID | Priority | Condition |
|---|---|---|
| `monthly_revenue_pace` | INFO / POSITIVE / WATCH | Always fires; priority adjusts based on % vs last month |
| `best_revenue_day_pattern` | INFO | A day of week averages 30%+ more than overall daily average |
| `revenue_concentration_risk` | WATCH | Top 3 clients > 40% of last-30-day revenue |
| `week_over_week_drop` | WATCH | This week's revenue < 70% of last week |
| `strong_week` | POSITIVE | This week's revenue > 120% of last week |

**Group 4 — Therapist Insights**

| Rule ID | Priority | Condition |
|---|---|---|
| `therapist_utilization_drop` | WATCH | Therapist's projected monthly sessions < 70% of their 3-month average |
| `top_performing_therapist` | POSITIVE | Always fires (when ≥ 2 therapists); highlights highest net-profit contributor |
| `therapist_payout_due` | INFO | Always fires; summarises each therapist's share owed this month |

**Group 5 — Client Milestones & Positives**

| Rule ID | Priority | Condition |
|---|---|---|
| `long_term_loyal_client` | POSITIVE | Sessions spanning > 4 months with no gap > 21 days |
| `high_lifetime_value_client` | POSITIVE | Client total billing > ₹50,000 |
| `client_returning_after_gap` | POSITIVE | Client returns within last 7 days after a gap > 21 days |

**Group 6 — Operational Flags**

| Rule ID | Priority | Condition |
|---|---|---|
| `sessions_without_payment_logged` | WATCH | Session rows older than 7 days with no payment on the same date |
| `data_freshness_check` | URGENT | Most recent row in the sheet is older than 2 days |
| `unusually_high_single_session` | WATCH | Session value > 3× that client's average in last 14 days |

#### All Rules Debug Section
A collapsed section at the bottom of the tab lists every rule ID, its category badge, and enabled/disabled status. Expand it to audit the engine without opening source code.

---

## Charts Summary

| Chart | Type | Tab |
|---|---|---|
| Revenue, Profit & Sessions (daily/weekly) | Stacked bar + line | Monthly Trend |
| Revenue by Therapist | Grouped stacked bar | Therapist |
| Monthly Sessions by Therapist | Multi-line | Therapist |
| Top Customers — Revenue & Sessions | Bar + line | Customers |

The Insights tab contains no charts — it is text-card based only.

All charts use [Chart.js 4.4.3](https://www.chartjs.org/), are fully responsive, and are destroyed and rebuilt on every filter change to avoid stale data.

---

## Number Formatting

| Range | Format |
|---|---|
| ≥ 1,00,000 | ₹X.XXL |
| ≥ 1,000 | ₹X.XXK |
| < 1,000 | ₹X.XX |

Exact values (full Indian locale formatting) are shown in tooltips on KPI cards.

---

## Responsive Behaviour

| Breakpoint | Change |
|---|---|
| ≤ 900 px | Charts grid collapses to single column; header/filter padding reduced |
| ≤ 600 px | KPI grid collapses to two columns |

---

## Error Handling

- A loading spinner overlay is shown while both sheets are being fetched.
- If either sheet fails to load (network error, not publicly shared, or 15-second timeout), an error banner appears above the content area with a plain-English message and the underlying error detail.
