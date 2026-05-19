# Referral Management + Multi-select Filters — Design Spec
Date: 2026-05-19

## Overview

Three features:
1. **Referral Tab** — track what is owed to each referral source
2. **Multi-select Filters** — Month, Therapist, Customer filters become multi-select
3. **SRP Shortcut** — one-click select of SRP clients inside Customer dropdown

---

## 1. Data Sources

### Clients Sheet (already exists)
Sheet name: `Clients`
Columns read by dashboard:
- `Customer Name` — join key with session rows
- `Referral Source` — source name (blank = no referral)
- `Pay Type` — `one-time` | `per-session` | `na` | blank (blank/na = informational only, no payment)
- `Amount Type` — `fixed` | `percent`
- `Amount` — number (₹ if fixed, % if percent)
- `Cap Sessions` — max sessions to pay for (blank = no cap); only relevant for `per-session`
- `Paid` — cumulative amount already paid TO this source FOR THIS CLIENT (number, blank = 0)
- `Paid Date` — last payment date (display only)
- `Paid Notes` — freeform notes (display only)

### Master Sheet (new fetch)
Sheet name: `Master`
Column J: list of SRP client names (non-empty cells = SRP clients)
Fetched at load time via GViz, stored as `srpClients[]`.

---

## 2. Referral Amount Computation

```
function computeReferralAccrued(client, sessions, rows):
  if payType is blank or 'na': return null  // informational only

  sessionCount = min(sessions, capSessions ?? Infinity)
  revenue = sum of You Gave for this client (across all session rows)
  revenueForCap = sum of You Gave for first capSessions sessions (sorted by date)

  if payType == 'one-time':
    if firstSessionExists:
      accrued = fixed ? Amount : (Amount/100 × firstSessionRevenue)
    else:
      accrued = 0

  if payType == 'per-session':
    effectiveRevenue = capSessions ? revenueForCap : revenue
    accrued = fixed ? (Amount × sessionCount) : (Amount/100 × effectiveRevenue)

  owed = accrued - (Paid ?? 0)
  return { accrued, paid: Paid, owed, paidDate, paidNotes }
```

Clients with no `Referral Source` are excluded from all referral calculations.
Clients with `Referral Source` but blank/na `Pay Type` appear in summary as "Info only" (no amounts).

---

## 3. Referral Tab UI

### Summary Table (default view)
One row per unique referral source. Columns:
- Source name
- # Clients referred
- Total Accrued (sum across all their clients with payment terms)
- Total Paid (sum of Paid across their clients)
- Total Owed (Accrued − Paid)
- Status badge: `Settled` (owed=0), `Due ₹X`, `Info only` (no pay terms)

Click a row → expands or navigates to Source Detail view.

### Source Detail View
Triggered by clicking source in summary or selecting from source dropdown.

**KPI cards (3):**
- Total Accrued
- Total Paid
- Currently Owed

**Client table:**
| Client | Sessions | Revenue | Accrued | Paid | Owed | Paid Date | Notes | Status |
|--------|----------|---------|---------|------|------|-----------|-------|--------|

Back button returns to summary.

### Filter
Source selector dropdown at top of tab (single-select, since user wants 1 source at a time in detail view). Summary always shows all sources.

---

## 4. Multi-select Filters

Filters converted from `<select>` to custom pill-based multi-select:
- **Month** — multi-select
- **Therapist** — multi-select
- **Customer** — multi-select
- **Year** — stays single-select (multi-year comparison not useful)

### UI Behavior
- Closed state: shows "All" or pill list of selected items (truncated at 2 with "+N more")
- Open state: dropdown panel with:
  - Search box (for Customer, which has many items)
  - "Select All" / "Clear" buttons
  - **SRP button** (Customer dropdown only — see §5)
  - Checkboxes for each option
- Click outside or press Escape closes dropdown
- Filter logic: `OR` within a field (show rows matching ANY selected value), `AND` across fields (same as current behavior)

### Backward compatibility
All existing filter-dependent code (`applyFilters`, charts, KPIs) uses `filteredRows` — no changes needed there. Only the filter read logic changes: instead of `select.value`, read an array of selected values and filter with `.includes()`.

---

## 5. SRP Shortcut

Inside the Customer multi-select dropdown, above the options list:

```
[ SRP ▾ ]   ← button, styled distinctly (e.g. amber chip)
```

Behavior on click:
1. Deselects all current Customer selections
2. Selects all names in `srpClients[]` that exist in current customer list
3. Closes dropdown
4. Triggers filter re-apply

`srpClients` loaded from Master!J at dashboard load. If Master sheet fetch fails, SRP button is hidden.

---

## 6. Technical Notes

### Fetching Clients Sheet
Use same GViz JSONP pattern as other sheets. Query: `SELECT *` from `Clients` tab.
Store as `clientsMap: Map<string, ClientRecord>` keyed by `Customer Name`.
Load in parallel with existing sheet fetches.

### Fetching Master Sheet
Query column J only: `SELECT J FROM Master WHERE J IS NOT NULL`.
Store as `srpClients: string[]`.
Load in parallel with other fetches.

### State
New global:
```js
let clientsMap = new Map();   // Customer Name → referral/rate record
let srpClients = [];          // SRP client names from Master!J
```

Existing globals unchanged.

### Tab Registration
Add `referrals` to tab list. Tab renders after `allRowsRaw` and `clientsMap` are loaded.

---

## 7. Out of Scope
- Editing referral data from dashboard (sheet is source of truth)
- Email/payment reminders
- Historical tracking of individual payments (single cumulative Paid value per client)
- Referral tab filters (inherits global Year filter only; referrals are lifetime)
