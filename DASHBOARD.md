# Capybara Care — Revenue Dashboard

A single-page HTML dashboard that pulls session and payment data from two Google Sheets tabs and presents it across four views: Home, Monthly Trend, Therapist, and Customers.

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

| Filter | Field |
|---|---|
| Year | `Year` column |
| Month | `Month` column |
| Therapist | `Therapist` column |
| Customer | `Customer Name` column |

On initial load the dashboard auto-selects the current calendar month and year. Changing any dropdown immediately re-renders all tabs and charts.

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

## Charts Summary

| Chart | Type | Tab |
|---|---|---|
| Revenue, Profit & Sessions (daily/weekly) | Stacked bar + line | Monthly Trend |
| Revenue by Therapist | Grouped stacked bar | Therapist |
| Monthly Sessions by Therapist | Multi-line | Therapist |
| Top Customers — Revenue & Sessions | Bar + line | Customers |

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
