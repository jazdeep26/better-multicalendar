# Referral Management + Multi-select Filters Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a Referral tab showing amounts owed per referral source, convert Month/Therapist/Customer filters to multi-select with SRP shortcut, all in `dashboard_src.html`.

**Architecture:** Single-file vanilla HTML/JS dashboard reads Google Sheets via GViz JSONP. New `Clients` sheet fetch (already exists, new to dashboard) provides referral master data. `Master` sheet provides SRP client list. Multi-select replaces `<select>` elements with custom pill-based dropdowns tracked in a JS state map. Referral tab is lifetime-only (ignores all filters).

**Tech Stack:** Vanilla HTML/CSS/JS, Google Visualization JSONP API, Chart.js (not needed for referral tab).

---

## File Map

- Modify only: `dashboard_src.html`
  - Constants block (~line 721): add `SHEET_CLIENTS`, `SHEET_MASTER`
  - State block (~line 744): add `clientsMap`, `srpClients`, `msSelections`
  - `sheetUrl()` (~line 737): add optional `tq` param
  - `fetchSheetData()` (~line 828): add optional `tq` param
  - `populateFilters()` (~line 938): replace 3 fillSelect calls with multi-select builders
  - `fillSelect()` (~line 972): keep for Year + Calendar (single-select)
  - `getFilterValues()` (~line 998): return arrays for month/therapist/customer
  - `applyFilters()` (~line 1009): use `.includes()` for array fields
  - `computeKPIs()` (~line 1044): update filter checks for arrays
  - `buildMonthlyTrend()` (~line 1090): update projRows filter for arrays
  - `render()` (~line 3097): add `renderReferralsTab()` call
  - `init()` (~line 3117): add Clients + Master fetches in parallel
  - HTML filters bar (~line 428): replace 3 `<select>` with multi-select wrappers
  - HTML tab nav (~line 465): add Referrals button
  - HTML main (~line 700): add Referrals tab panel
  - CSS (`<style>` block): add multi-select + referral tab styles
  - Event listeners (~line 3240): update filter listeners for multi-select

---

## Task 1: Constants, State, Extend Sheet Fetcher

**Files:**
- Modify: `dashboard_src.html` (CONFIG block ~721, STATE block ~744, sheetUrl ~737, fetchSheetData ~828)

- [ ] **Step 1: Add sheet name constants**

In the CONFIG block after line 725, add:
```js
const SHEET_CLIENTS  = 'Clients';          // Referral master data
const SHEET_MASTER   = 'Master';           // SRP list in column J
```

- [ ] **Step 2: Add state variables**

In the STATE block after line 756, add:
```js
let clientsMap = new Map();   // Customer Name → { 'Referral Source', 'Pay Type', 'Amount Type', 'Amount', 'Cap Sessions', 'Paid', 'Paid Date', 'Paid Notes' }
let srpClients = [];          // Customer names from Master!J
// Multi-select filter state: arrays of selected values (empty = All)
const msSelections = { month: [], therapist: [], customer: [] };
```

- [ ] **Step 3: Extend sheetUrl to accept optional query**

Replace the existing `sheetUrl` function (~line 737):
```js
function sheetUrl(sheetName, cbName, tq = '') {
  let url = `https://docs.google.com/spreadsheets/d/${SHEET_ID}/gviz/tq?tqx=responseHandler:${cbName}&sheet=${encodeURIComponent(sheetName)}&_t=${Date.now()}`;
  if (tq) url += `&tq=${encodeURIComponent(tq)}`;
  return url;
}
```

- [ ] **Step 4: Extend fetchSheetData to accept optional query**

Replace the `fetchSheetData` function signature and the `script.src` line (~line 828):
```js
function fetchSheetData(sheetName, tq = '') {
  return new Promise((resolve, reject) => {
    const cbName  = `__gvizCb_${++_gvizReqId}`;
    const scriptId = `__gviz_script_${_gvizReqId}`;

    document.querySelectorAll('script[id^="__gviz_script_"]').forEach(s => s.remove());

    const timeout = setTimeout(() => {
      delete window[cbName];
      reject(new Error('Request timed out. Check that the sheet is publicly shared (Anyone with the link → Viewer).'));
    }, 15000);

    window[cbName] = function(json) {
      clearTimeout(timeout);
      delete window[cbName];
      document.getElementById(scriptId)?.remove();
      try { resolve(parseGvizTable(json)); } catch(e) { reject(e); }
    };

    const script = document.createElement('script');
    script.id  = scriptId;
    script.src = sheetUrl(sheetName, cbName, tq);
    script.onerror = () => {
      clearTimeout(timeout);
      delete window[cbName];
      script.remove();
      reject(new Error(`Failed to load sheet "${sheetName}". Check that it is publicly shared.`));
    };
    document.head.appendChild(script);
  });
}
```

- [ ] **Step 5: Verify no breakage**

Open `dashboard_src.html` in browser. Dashboard should load exactly as before. No console errors.

- [ ] **Step 6: Commit**

```bash
git add dashboard_src.html
git commit -m "feat: add Clients/Master sheet constants and extend fetchSheetData with optional query"
```

---

## Task 2: Fetch Clients and Master in init()

**Files:**
- Modify: `dashboard_src.html` (init function ~line 3117)

- [ ] **Step 1: Add Clients fetch and parse inside init()**

Inside `init()`, after `const raw = await fetchAllData();` and before `allRowsRaw = ...`, add:
```js
    // Fetch Clients sheet for referral master data (in parallel with session data already fetched)
    try {
      const clientRows = await fetchSheetData(SHEET_CLIENTS);
      clientsMap = new Map();
      for (const r of clientRows) {
        const name = (r['Customer Name'] || '').trim();
        if (!name || name.startsWith('_default_')) continue;
        clientsMap.set(name, r);
      }
    } catch (e) {
      console.warn('Clients sheet unavailable — referral tab will be empty:', e.message);
      clientsMap = new Map();
    }
```

- [ ] **Step 2: Add Master sheet fetch for SRP list**

Directly after the clientsMap block, add:
```js
    // Fetch Master!J for SRP client list
    try {
      const masterRows = await fetchSheetData(SHEET_MASTER, 'SELECT J');
      srpClients = masterRows
        .map(r => Object.values(r)[0])
        .filter(v => v && typeof v === 'string' && v.trim())
        .map(v => v.trim());
    } catch (e) {
      console.warn('Master sheet unavailable — SRP button hidden:', e.message);
      srpClients = [];
    }
```

- [ ] **Step 3: Verify in browser**

Open dashboard. Open DevTools console. After load, run:
```js
console.log([...clientsMap.entries()].slice(0,3));
console.log(srpClients);
```
Should see Clients map entries with referral fields and srpClients array (or empty array if Master not public).

- [ ] **Step 4: Commit**

```bash
git add dashboard_src.html
git commit -m "feat: fetch Clients sheet into clientsMap and Master!J into srpClients at init"
```

---

## Task 3: Referral Computation Functions

**Files:**
- Modify: `dashboard_src.html` (add new section before MASTER RENDER ~line 3094)

- [ ] **Step 1: Add `computeReferralData()` function**

Add the following before the `// MASTER RENDER` comment (~line 3094):
```js
// ─────────────────────────────────────────────────────────────────────────────
// REFERRAL TAB
// ─────────────────────────────────────────────────────────────────────────────
function computeReferralData() {
  // Returns Map<sourceName, { source, clients[], totalAccrued, totalPaid, totalOwed }>
  // Uses allRows (session rows, You Gave > 0) and clientsMap for lifetime calculations
  const referralMap = new Map();

  for (const [clientName, record] of clientsMap) {
    const source   = (record['Referral Source'] || '').trim();
    if (!source) continue;

    const payType  = (record['Pay Type']      || '').trim().toLowerCase();
    const amtType  = (record['Amount Type']   || '').trim().toLowerCase();
    const amount   = parseNum(record['Amount']);
    const rawCap   = record['Cap Sessions'];
    const capSess  = (rawCap !== '' && rawCap !== null && rawCap !== undefined && !isNaN(parseInt(rawCap)))
                     ? parseInt(rawCap) : Infinity;
    const paid     = parseNum(record['Paid']);
    const paidDate = record['Paid Date']  || '';
    const paidNotes= record['Paid Notes'] || '';

    // All session rows for this client, sorted ascending by date
    const sessions = allRows
      .filter(r => r['Customer Name'] === clientName)
      .sort((a, b) => {
        const da = parseDate(a['Date']), db = parseDate(b['Date']);
        return (da && db) ? da - db : 0;
      });

    const sessionCount = sessions.length;
    const totalRevenue = sessions.reduce((s, r) => s + parseNum(r['You Gave']), 0);

    let accrued = null;

    if (payType && payType !== 'na') {
      const effectiveCount = isFinite(capSess) ? Math.min(sessionCount, capSess) : sessionCount;
      const cappedSessions = sessions.slice(0, effectiveCount);
      const cappedRevenue  = cappedSessions.reduce((s, r) => s + parseNum(r['You Gave']), 0);

      if (payType === 'one-time') {
        if (sessionCount > 0) {
          accrued = amtType === 'percent'
            ? (amount / 100) * parseNum(sessions[0]['You Gave'])
            : amount;
        } else {
          accrued = 0;
        }
      } else if (payType === 'per-session') {
        accrued = amtType === 'percent'
          ? (amount / 100) * cappedRevenue
          : amount * effectiveCount;
      }
    }

    const owed = accrued !== null ? accrued - paid : null;

    const clientEntry = {
      clientName, source, payType, amtType, amount, capSess,
      sessionCount, revenue: totalRevenue,
      accrued, paid, owed, paidDate, paidNotes,
    };

    if (!referralMap.has(source)) {
      referralMap.set(source, { source, clients: [], totalAccrued: 0, totalPaid: 0, totalOwed: 0 });
    }
    const sd = referralMap.get(source);
    sd.clients.push(clientEntry);
    if (accrued !== null) {
      sd.totalAccrued += accrued;
      sd.totalPaid    += paid;
      sd.totalOwed    += owed;
    }
  }

  return referralMap;
}
```

- [ ] **Step 2: Verify function in console**

Open browser, open DevTools after load, run:
```js
const rd = computeReferralData();
console.log([...rd.entries()].map(([s,d]) => ({ source: s, owed: d.totalOwed, clients: d.clients.length })));
```
Should see entries for Dr. Garima (with non-zero totalOwed for per-session percent clients), Naina (owed = 0, no pay terms), Tushar (owed = 0, fully paid), etc.

- [ ] **Step 3: Commit**

```bash
git add dashboard_src.html
git commit -m "feat: add computeReferralData() for lifetime referral accrual calculation"
```

---

## Task 4: Referral Tab CSS

**Files:**
- Modify: `dashboard_src.html` (`<style>` block, before `</style>` ~line 397)

- [ ] **Step 1: Add referral tab CSS**

Insert before `</style>`:
```css
/* ── Referral Tab ── */
.referral-kpi-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 14px; margin-bottom: 24px; }
.referral-back-btn {
  display: inline-flex; align-items: center; gap: 6px;
  background: var(--white); border: 1px solid var(--gray-200);
  border-radius: 8px; padding: 7px 14px; font-size: 13px; font-weight: 600;
  color: var(--gray-600); cursor: pointer; font-family: inherit;
  margin-bottom: 18px; transition: all .15s;
}
.referral-back-btn:hover { background: var(--green-pale); border-color: var(--green-mid); color: var(--green-dark); }
.referral-source-header {
  font-size: 18px; font-weight: 700; color: var(--gray-800);
  margin-bottom: 16px; display: flex; align-items: center; gap: 10px;
}
.ref-badge-settled { background: var(--green-pale); color: var(--green-dark); }
.ref-badge-due     { background: #fee2e2; color: var(--danger); }
.ref-badge-info    { background: var(--gray-100); color: var(--gray-600); }
.ref-amt-type      { font-size: 11px; color: var(--gray-400); display: block; margin-top: 2px; }
.referral-empty {
  text-align: center; padding: 48px 24px; color: var(--gray-400);
  font-size: 14px;
}
```

- [ ] **Step 2: Verify CSS loads with no errors**

Open browser. No layout shifts, no console CSS errors.

---

## Task 5: Referral Tab HTML + Render Functions

**Files:**
- Modify: `dashboard_src.html` (tab-nav ~line 465, main ~line 701, before MASTER RENDER)

- [ ] **Step 1: Add Referrals tab button**

In the tab nav (after the Insights button, ~line 470):
```html
  <button class="tab-btn" data-tab="referrals">Referrals</button>
```

- [ ] **Step 2: Add Referrals tab panel HTML**

After the `</div><!-- /insights -->` (~line 699), before `</main>`:
```html
  <!-- ═══════════════════ REFERRALS TAB ═══════════════════ -->
  <div class="tab-panel" id="tab-referrals">

    <!-- Summary view -->
    <div id="referrals-summary-view">
      <div class="table-card">
        <div class="table-header">
          <h3>Referral Sources</h3>
          <span style="font-size:12px;color:var(--gray-400)">Lifetime totals · click a row to see clients</span>
        </div>
        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>Source</th>
                <th>Clients</th>
                <th>Accrued (₹)</th>
                <th>Paid (₹)</th>
                <th>Owed (₹)</th>
                <th>Status</th>
              </tr>
            </thead>
            <tbody id="referral-summary-body"></tbody>
          </table>
        </div>
      </div>
    </div>

    <!-- Detail view (hidden until source clicked) -->
    <div id="referrals-detail-view" style="display:none">
      <button class="referral-back-btn" id="referral-back-btn">← All Sources</button>
      <div class="referral-source-header" id="referral-detail-title">Source Name</div>

      <div class="referral-kpi-grid" id="referral-detail-kpis"></div>

      <div class="table-card">
        <div class="table-header">
          <h3>Referred Clients</h3>
        </div>
        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>Client</th>
                <th>Sessions</th>
                <th>Revenue (₹)</th>
                <th>Terms</th>
                <th>Accrued (₹)</th>
                <th>Paid (₹)</th>
                <th>Owed (₹)</th>
                <th>Paid Date</th>
                <th>Notes</th>
                <th>Status</th>
              </tr>
            </thead>
            <tbody id="referral-detail-body"></tbody>
          </table>
        </div>
      </div>
    </div>

  </div><!-- /referrals -->
```

- [ ] **Step 3: Add `renderReferralsTab()` function**

Add after `computeReferralData()`:
```js
function renderReferralsTab() {
  const referralMap = computeReferralData();
  const summaryBody = document.getElementById('referral-summary-body');

  if (clientsMap.size === 0) {
    summaryBody.innerHTML = `<tr><td colspan="6" class="referral-empty">Clients sheet not loaded. Make sure it is publicly shared.</td></tr>`;
    return;
  }

  const sources = [...referralMap.values()].sort((a, b) => b.totalOwed - a.totalOwed);

  if (sources.length === 0) {
    summaryBody.innerHTML = `<tr><td colspan="6" class="referral-empty">No referral sources found in Clients sheet.</td></tr>`;
    return;
  }

  summaryBody.innerHTML = sources.map(sd => {
    const hasTerms  = sd.clients.some(c => c.accrued !== null);
    const badge     = !hasTerms      ? `<span class="badge ref-badge-info">Info only</span>` :
                      sd.totalOwed <= 0 ? `<span class="badge ref-badge-settled">Settled</span>` :
                                          `<span class="badge ref-badge-due">Due ${fmtFull(sd.totalOwed)}</span>`;
    return `<tr style="cursor:pointer" data-source="${sd.source.replace(/"/g,'&quot;')}" onclick="showReferralDetail('${sd.source.replace(/'/g,"\\'")}')">
      <td style="font-weight:600">${sd.source}</td>
      <td>${sd.clients.length}</td>
      <td>${hasTerms ? fmtFull(sd.totalAccrued) : '—'}</td>
      <td>${hasTerms ? fmtFull(sd.totalPaid)    : '—'}</td>
      <td>${hasTerms ? fmtFull(sd.totalOwed)    : '—'}</td>
      <td>${badge}</td>
    </tr>`;
  }).join('');
}

function showReferralDetail(sourceName) {
  const referralMap = computeReferralData();
  const sd = referralMap.get(sourceName);
  if (!sd) return;

  document.getElementById('referrals-summary-view').style.display = 'none';
  document.getElementById('referrals-detail-view').style.display  = '';
  document.getElementById('referral-detail-title').textContent = sourceName;

  // KPI cards
  const hasTerms = sd.clients.some(c => c.accrued !== null);
  const kpiEl    = document.getElementById('referral-detail-kpis');
  kpiEl.innerHTML = hasTerms ? `
    <div class="kpi-card c1"><span class="kpi-label">Total Accrued</span><span class="kpi-value" title="${fmtExact(sd.totalAccrued)}">${fmtShort(sd.totalAccrued)}</span></div>
    <div class="kpi-card c3"><span class="kpi-label">Total Paid</span><span class="kpi-value" title="${fmtExact(sd.totalPaid)}">${fmtShort(sd.totalPaid)}</span></div>
    <div class="kpi-card ${sd.totalOwed > 0 ? 'c1' : 'c5'}"><span class="kpi-label">Currently Owed</span><span class="kpi-value" title="${fmtExact(sd.totalOwed)}">${fmtShort(sd.totalOwed)}</span></div>
  ` : `<div class="kpi-card c5"><span class="kpi-label">Clients Referred</span><span class="kpi-value">${sd.clients.length}</span><span class="kpi-sub">Informational only — no payment terms set</span></div>`;

  // Client rows
  const body = document.getElementById('referral-detail-body');
  body.innerHTML = sd.clients.map(c => {
    const termsStr = c.accrued !== null
      ? `${c.payType === 'one-time' ? 'One-time' : 'Per-session'} · ${c.amtType === 'percent' ? c.amount + '%' : fmtFull(c.amount)}${isFinite(c.capSess) ? ` · cap ${c.capSess}` : ''}`
      : 'Info only';
    const badge = c.accrued === null ? `<span class="badge ref-badge-info">Info</span>` :
                  c.owed <= 0        ? `<span class="badge ref-badge-settled">Settled</span>` :
                                       `<span class="badge ref-badge-due">Due</span>`;
    return `<tr>
      <td style="font-weight:600">${c.clientName}</td>
      <td>${c.sessionCount}</td>
      <td>${fmtFull(c.revenue)}</td>
      <td style="font-size:12px;color:var(--gray-600)">${termsStr}</td>
      <td>${c.accrued !== null ? fmtFull(c.accrued) : '—'}</td>
      <td>${c.accrued !== null ? fmtFull(c.paid)    : '—'}</td>
      <td>${c.accrued !== null ? fmtFull(c.owed)    : '—'}</td>
      <td style="font-size:12px;color:var(--gray-600)">${c.paidDate || '—'}</td>
      <td style="font-size:12px;color:var(--gray-600)">${c.paidNotes || '—'}</td>
      <td>${badge}</td>
    </tr>`;
  }).join('');
}
```

- [ ] **Step 4: Wire back button and add renderReferralsTab to render()**

Add to event listeners section:
```js
document.getElementById('referral-back-btn').addEventListener('click', () => {
  document.getElementById('referrals-detail-view').style.display  = 'none';
  document.getElementById('referrals-summary-view').style.display = '';
});
```

In `render()`, add as the last line before the closing `}`:
```js
  renderReferralsTab();
```

- [ ] **Step 5: Verify in browser**

Click "Referrals" tab. Summary table should show sources (Dr. Garima with owed amount, Naina as Info only, Tushar as Settled, etc.). Click a row → detail view with KPIs and client table. Back button returns to summary.

- [ ] **Step 6: Commit**

```bash
git add dashboard_src.html
git commit -m "feat: add Referrals tab with summary + detail views"
```

---

## Task 6: Multi-select Component CSS + Builder

**Files:**
- Modify: `dashboard_src.html` (`<style>` block, and new JS functions)

- [ ] **Step 1: Add multi-select CSS**

Insert before `</style>` (after referral CSS from Task 4):
```css
/* ── Multi-select filter ── */
.ms-wrap { position: relative; }
.ms-trigger {
  display: flex; align-items: center; gap: 6px;
  border: 1px solid var(--gray-200); border-radius: 8px; padding: 5px 10px;
  font-size: 13px; font-family: inherit; color: var(--gray-800);
  background: var(--white); cursor: pointer; min-width: 130px;
  transition: border-color .2s; user-select: none; white-space: nowrap;
}
.ms-trigger:hover, .ms-trigger.open { border-color: var(--green-mid); }
.ms-trigger-text { flex: 1; overflow: hidden; text-overflow: ellipsis; }
.ms-caret { font-size: 10px; color: var(--gray-400); flex-shrink: 0; }
.ms-pill {
  display: inline-block; background: var(--green-pale); color: var(--green-dark);
  border-radius: 999px; padding: 1px 7px; font-size: 11px; font-weight: 600;
  max-width: 90px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap;
}
.ms-dropdown {
  display: none; position: absolute; top: calc(100% + 4px); left: 0;
  background: var(--white); border: 1px solid var(--gray-200);
  border-radius: 10px; box-shadow: 0 8px 24px rgba(0,0,0,.12);
  min-width: 220px; z-index: 200; overflow: hidden;
}
.ms-dropdown.open { display: block; }
.ms-search {
  display: block; width: 100%; padding: 8px 12px; border: none;
  border-bottom: 1px solid var(--gray-100); font-size: 13px; font-family: inherit;
  outline: none; color: var(--gray-800);
}
.ms-actions {
  display: flex; gap: 4px; padding: 6px 10px 4px;
  border-bottom: 1px solid var(--gray-100);
}
.ms-action-btn {
  font-size: 11px; font-weight: 600; color: var(--green-dark);
  background: none; border: none; cursor: pointer; padding: 2px 4px;
  font-family: inherit;
}
.ms-action-btn:hover { text-decoration: underline; }
.ms-srp-btn {
  margin-left: auto; background: #fef3c7; color: #92400e;
  border: 1px solid #fcd34d; border-radius: 999px;
  padding: 2px 10px; font-size: 11px; font-weight: 700;
  cursor: pointer; font-family: inherit;
}
.ms-srp-btn:hover { background: #fde68a; }
.ms-options { max-height: 200px; overflow-y: auto; padding: 4px 0; }
.ms-option {
  display: flex; align-items: center; gap: 8px;
  padding: 6px 12px; cursor: pointer; font-size: 13px;
  transition: background .1s;
}
.ms-option:hover { background: var(--green-bg); }
.ms-option input[type="checkbox"] {
  accent-color: var(--green-mid); width: 14px; height: 14px; flex-shrink: 0; cursor: pointer;
}
.ms-option label { cursor: pointer; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.ms-empty { padding: 10px 12px; font-size: 13px; color: var(--gray-400); }
```

- [ ] **Step 2: Add multi-select builder and helpers**

Add new JS section after the `fillSelect` function (~line 982):
```js
// ─────────────────────────────────────────────────────────────────────────────
// MULTI-SELECT COMPONENT
// ─────────────────────────────────────────────────────────────────────────────
function buildMultiSelect(wrapperId, key, values, placeholder) {
  // wrapperId = DOM id of .ms-wrap div
  // key = one of 'month' | 'therapist' | 'customer'
  const wrap = document.getElementById(wrapperId);
  if (!wrap) return;

  const selected = msSelections[key]; // array ref — mutate in place

  // Re-render trigger label
  function updateTrigger() {
    const trigger = wrap.querySelector('.ms-trigger');
    const textEl  = wrap.querySelector('.ms-trigger-text');
    if (!trigger || !textEl) return;
    if (selected.length === 0) {
      textEl.textContent = placeholder;
      trigger.classList.remove('has-selection');
    } else if (selected.length === 1) {
      textEl.innerHTML = `<span class="ms-pill">${selected[0]}</span>`;
      trigger.classList.add('has-selection');
    } else if (selected.length === 2) {
      textEl.innerHTML = selected.map(s => `<span class="ms-pill">${s}</span>`).join(' ');
      trigger.classList.add('has-selection');
    } else {
      textEl.innerHTML = `<span class="ms-pill">${selected[0]}</span> <span style="font-size:11px;color:var(--gray-600)">+${selected.length - 1} more</span>`;
      trigger.classList.add('has-selection');
    }
  }

  function renderOptions(filter) {
    const optionsEl = wrap.querySelector('.ms-options');
    if (!optionsEl) return;
    const filtered  = filter ? values.filter(v => v.toLowerCase().includes(filter.toLowerCase())) : values;
    if (filtered.length === 0) {
      optionsEl.innerHTML = `<div class="ms-empty">No matches</div>`;
      return;
    }
    optionsEl.innerHTML = filtered.map(v => {
      const chk = selected.includes(v) ? 'checked' : '';
      const eid = `ms-opt-${key}-${v.replace(/\s+/g,'_')}`;
      return `<div class="ms-option">
        <input type="checkbox" id="${eid}" value="${v}" ${chk} onchange="msToggle('${key}','${v.replace(/'/g,"\\'")}',this.checked)">
        <label for="${eid}">${v}</label>
      </div>`;
    }).join('');
  }

  // Build HTML (only once — subsequent calls re-render options)
  const existingTrigger = wrap.querySelector('.ms-trigger');
  if (!existingTrigger) {
    const hasSrp = key === 'customer';
    wrap.innerHTML = `
      <div class="ms-trigger" onclick="msToggleDropdown('${key}')">
        <span class="ms-trigger-text">${placeholder}</span>
        <span class="ms-caret">▾</span>
      </div>
      <div class="ms-dropdown" id="ms-dropdown-${key}">
        ${values.length > 10 ? `<input class="ms-search" type="text" placeholder="Search…" oninput="msSearch('${key}',this.value)">` : ''}
        <div class="ms-actions">
          <button class="ms-action-btn" onclick="msSelectAll('${key}')">All</button>
          <button class="ms-action-btn" onclick="msClear('${key}')">Clear</button>
          ${hasSrp ? `<button class="ms-srp-btn" id="ms-srp-btn" onclick="msSRPSelect(event)" style="${srpClients.length === 0 ? 'display:none' : ''}">SRP</button>` : ''}
        </div>
        <div class="ms-options"></div>
      </div>
    `;
  }

  renderOptions('');
  updateTrigger();

  // Store values list on wrapper for All/Search operations
  wrap._msValues = values;
  wrap._msKey    = key;
}

function msToggle(key, value, checked) {
  if (checked) {
    if (!msSelections[key].includes(value)) msSelections[key].push(value);
  } else {
    msSelections[key] = msSelections[key].filter(v => v !== value);
  }
  // Update trigger label
  const wrap = document.querySelector(`[data-ms-key="${key}"]`);
  if (wrap) {
    const textEl  = wrap.querySelector('.ms-trigger-text');
    const trigger = wrap.querySelector('.ms-trigger');
    const selected = msSelections[key];
    if (selected.length === 0) { textEl.textContent = wrap.dataset.msPlaceholder; trigger.classList.remove('has-selection'); }
    else if (selected.length === 1) { textEl.innerHTML = `<span class="ms-pill">${selected[0]}</span>`; trigger.classList.add('has-selection'); }
    else if (selected.length === 2) { textEl.innerHTML = selected.map(s=>`<span class="ms-pill">${s}</span>`).join(' '); trigger.classList.add('has-selection'); }
    else { textEl.innerHTML = `<span class="ms-pill">${selected[0]}</span> <span style="font-size:11px;color:var(--gray-600)">+${selected.length-1} more</span>`; trigger.classList.add('has-selection'); }
  }
  applyFilters();
}

function msToggleDropdown(key) {
  const drop = document.getElementById(`ms-dropdown-${key}`);
  if (!drop) return;
  const isOpen = drop.classList.contains('open');
  // Close all other dropdowns first
  document.querySelectorAll('.ms-dropdown.open').forEach(d => d.classList.remove('open'));
  document.querySelectorAll('.ms-trigger.open').forEach(t => t.classList.remove('open'));
  if (!isOpen) {
    drop.classList.add('open');
    const trigger = drop.previousElementSibling;
    if (trigger) trigger.classList.add('open');
    const search = drop.querySelector('.ms-search');
    if (search) { search.value = ''; msSearch(key, ''); search.focus(); }
  }
}

function msSearch(key, query) {
  const wrap   = document.querySelector(`[data-ms-key="${key}"]`);
  if (!wrap) return;
  const values   = wrap._msValues || [];
  const selected = msSelections[key];
  const filtered = query ? values.filter(v => v.toLowerCase().includes(query.toLowerCase())) : values;
  const optEl    = document.getElementById(`ms-dropdown-${key}`)?.querySelector('.ms-options');
  if (!optEl) return;
  if (filtered.length === 0) { optEl.innerHTML = `<div class="ms-empty">No matches</div>`; return; }
  optEl.innerHTML = filtered.map(v => {
    const chk = selected.includes(v) ? 'checked' : '';
    const eid = `ms-opt-${key}-${v.replace(/\s+/g,'_')}`;
    return `<div class="ms-option">
      <input type="checkbox" id="${eid}" value="${v}" ${chk} onchange="msToggle('${key}','${v.replace(/'/g,"\\'")}',this.checked)">
      <label for="${eid}">${v}</label>
    </div>`;
  }).join('');
}

function msSelectAll(key) {
  const wrap = document.querySelector(`[data-ms-key="${key}"]`);
  if (!wrap) return;
  msSelections[key] = [...(wrap._msValues || [])];
  buildMultiSelect(wrap.id, key, wrap._msValues, wrap.dataset.msPlaceholder);
  applyFilters();
}

function msClear(key) {
  msSelections[key] = [];
  const wrap = document.querySelector(`[data-ms-key="${key}"]`);
  if (wrap) buildMultiSelect(wrap.id, key, wrap._msValues, wrap.dataset.msPlaceholder);
  applyFilters();
}

function msSRPSelect(event) {
  event.stopPropagation();
  const wrap = document.querySelector('[data-ms-key="customer"]');
  if (!wrap || srpClients.length === 0) return;
  const validSRP = srpClients.filter(c => (wrap._msValues || []).includes(c));
  msSelections.customer = validSRP;
  buildMultiSelect(wrap.id, 'customer', wrap._msValues, wrap.dataset.msPlaceholder);
  // Close dropdown
  const drop = document.getElementById('ms-dropdown-customer');
  if (drop) drop.classList.remove('open');
  applyFilters();
}

// Close dropdowns on outside click
document.addEventListener('click', e => {
  if (!e.target.closest('.ms-wrap')) {
    document.querySelectorAll('.ms-dropdown.open').forEach(d => d.classList.remove('open'));
    document.querySelectorAll('.ms-trigger.open').forEach(t => t.classList.remove('open'));
  }
});
```

- [ ] **Step 3: Commit**

```bash
git add dashboard_src.html
git commit -m "feat: add multi-select component (CSS + builder + helpers)"
```

---

## Task 7: Wire Multi-select to Filters

**Files:**
- Modify: `dashboard_src.html` (HTML filters bar ~428, populateFilters ~938, getFilterValues ~998, applyFilters ~1009, computeKPIs ~1044, buildMonthlyTrend ~1090, event listeners ~3240)

- [ ] **Step 1: Replace 3 filter `<select>` elements with multi-select wrappers in HTML**

Replace the Month filter group (~line 433):
```html
  <div class="filter-group">
    <label>Month</label>
    <div class="ms-wrap" id="ms-month-wrap" data-ms-key="month" data-ms-placeholder="All Months"></div>
  </div>
```

Replace the Therapist filter group (~line 437):
```html
  <div class="filter-group">
    <label>Therapist</label>
    <div class="ms-wrap" id="ms-therapist-wrap" data-ms-key="therapist" data-ms-placeholder="All Therapists"></div>
  </div>
```

Replace the Customer filter group (~line 441):
```html
  <div class="filter-group">
    <label>Customer</label>
    <div class="ms-wrap" id="ms-customer-wrap" data-ms-key="customer" data-ms-placeholder="All Customers"></div>
  </div>
```

- [ ] **Step 2: Update `populateFilters()` to use buildMultiSelect**

Replace the `fillSelect` calls for month/therapist/customer inside `populateFilters()`:
```js
function populateFilters(rows) {
  const src = allRowsRaw;
  const years     = [...new Set(src.map(r => r['Year']).filter(Boolean))].sort((a,b) => b - a);
  const months    = [...new Set(src.map(r => r['Month']).filter(Boolean))].sort((a,b) => mIdx(a) - mIdx(b));
  const therapists = [...new Set(src.map(r => r['Therapist']).filter(Boolean))].sort();
  const customers  = [...new Set(src.map(r => r['Customer Name']).filter(Boolean))].sort();

  fillSelect('filter-year', years, 'All Years');
  buildMultiSelect('ms-month-wrap',     'month',     months,    'All Months');
  buildMultiSelect('ms-therapist-wrap', 'therapist', therapists,'All Therapists');
  buildMultiSelect('ms-customer-wrap',  'customer',  customers, 'All Customers');

  // SRP button visibility
  const srpBtn = document.getElementById('ms-srp-btn');
  if (srpBtn) srpBtn.style.display = srpClients.length === 0 ? 'none' : '';

  // Smridhi profile: lock therapist to Smridhi
  const tWrap = document.getElementById('ms-therapist-wrap');
  if (activeProfile === 'smridhi') {
    msSelections.therapist = [SMRIDHI_NAME];
    buildMultiSelect('ms-therapist-wrap', 'therapist', therapists, 'All Therapists');
    if (tWrap) tWrap.style.pointerEvents = 'none'; tWrap.style.opacity = '.6';
  } else {
    if (tWrap) { tWrap.style.pointerEvents = ''; tWrap.style.opacity = ''; }
  }

  // Include All checkbox: only visible on smridhi profile
  document.getElementById('filter-include-all-group').style.display =
    activeProfile === 'smridhi' ? '' : 'none';

  // Calendar filter: only visible for AllSessions profile
  const calGroup = document.getElementById('filter-calendar-group');
  if (activeProfile === 'allsessions') {
    const calendars = [...new Set(src.map(r => r['Calendar']).filter(Boolean))].sort();
    fillSelect('filter-calendar', calendars, 'All Calendars');
    calGroup.style.display = '';
  } else {
    calGroup.style.display = 'none';
    document.getElementById('filter-calendar').value = '';
  }
}
```

- [ ] **Step 3: Update `getFilterValues()` to return arrays**

Replace `getFilterValues()`:
```js
function getFilterValues() {
  return {
    year:      document.getElementById('filter-year').value,
    month:     [...msSelections.month],
    therapist: [...msSelections.therapist],
    customer:  [...msSelections.customer],
    calendar:  document.getElementById('filter-calendar').value,
    endDate:   document.getElementById('filter-end-date').value,
  };
}
```

- [ ] **Step 4: Update `applyFilters()` to use array includes**

Replace `applyFilters()`:
```js
function applyFilters() {
  const f = getFilterValues();
  const endDate = getEffectiveEndDate();
  filteredRows = allRows.filter(r => {
    if (f.year                        && String(r['Year'])      !== String(f.year))    return false;
    if (f.month.length     > 0        && !f.month.includes(r['Month']))                return false;
    if (f.therapist.length > 0        && !f.therapist.includes(r['Therapist']))        return false;
    if (f.customer.length  > 0        && !f.customer.includes(r['Customer Name']))     return false;
    if (f.calendar                    && r['Calendar']          !== f.calendar)        return false;
    const d = parseDate(r['Date']);
    if (d && d > endDate) return false;
    return true;
  });
  page = 1; custPage = 1;
  render();
}
```

- [ ] **Step 5: Update `computeKPIs()` allRowsRaw filter (~line 1044)**

Replace the filter block inside `computeKPIs()` (the part that filters `allRowsRaw` for outstanding):
```js
  allRowsRaw.forEach(r => {
    if (f.year                    && String(r['Year'])  !== String(f.year))  return;
    if (f.month.length     > 0    && !f.month.includes(r['Month']))           return;
    if (f.therapist.length > 0    && !f.therapist.includes(r['Therapist']))   return;
    if (f.customer.length  > 0    && !f.customer.includes(r['Customer Name']))return;
    if (f.calendar                && r['Calendar']      !== f.calendar)       return;
    const _d = parseDate(r['Date']);
    if (_d && _d > _endDate) return;
    totalReceived += parseNum(r['You Got']);
  });
```

- [ ] **Step 6: Update `buildMonthlyTrend()` projRows filter (~line 1090)**

Replace the `const projRows = showProj ? allRows.filter(r => {` block to use array includes:
```js
  const projRows = showProj ? allRows.filter(r => {
    if (f.year                    && String(r['Year'])      !== String(f.year))    return false;
    if (f.month.length     > 0    && !f.month.includes(r['Month']))                return false;
    if (f.therapist.length > 0    && !f.therapist.includes(r['Therapist']))        return false;
    if (f.customer.length  > 0    && !f.customer.includes(r['Customer Name']))     return false;
    if (f.calendar                && r['Calendar']          !== f.calendar)        return false;
    const d = parseDate(r['Date']);
    if (!d || d <= endDate) return false;
    return true;
  }) : [];
```

- [ ] **Step 7: Remove old filter event listeners**

Find and remove (or replace) the old filter event listener block (~line 3240):
```js
// OLD — remove this:
['filter-year','filter-month','filter-therapist','filter-customer','filter-calendar','filter-end-date'].forEach(id =>
  document.getElementById(id).addEventListener('change', applyFilters)
);
```

Replace with only the still-relevant single-select listeners:
```js
['filter-year','filter-calendar','filter-end-date'].forEach(id => {
  const el = document.getElementById(id);
  if (el) el.addEventListener('change', applyFilters);
});
```

- [ ] **Step 8: Update `openShareView()` subtitle for array filters (~line 3173)**

Replace the parts that build subtitle from `f.therapist`, `f.month`, `f.customer`:
```js
  const parts = [];
  if (f.therapist.length === 1)   parts.push(f.therapist[0]);
  else if (f.therapist.length > 1) parts.push(`${f.therapist.length} therapists`);
  if (f.month.length === 1 && f.year)   parts.push(`${f.month[0]} ${f.year}`);
  else if (f.month.length === 1)         parts.push(f.month[0]);
  else if (f.month.length > 1 && f.year) parts.push(`${f.month.length} months ${f.year}`);
  else if (f.year)                       parts.push(f.year);
  if (f.customer.length === 1)   parts.push(f.customer[0]);
  else if (f.customer.length > 1) parts.push(`${f.customer.length} customers`);
  const subtitle = parts.length ? parts.join(' · ') : 'All sessions';
```

- [ ] **Step 9: Reset msSelections on profile switch or init**

Inside `init()`, at the top (after clearing `allRowsRaw`, `allRows`, `filteredRows`), add:
```js
    msSelections.month     = [];
    msSelections.therapist = [];
    msSelections.customer  = [];
```

- [ ] **Step 10: Verify full filter behavior in browser**

1. Dashboard loads. Month/Therapist/Customer show pill dropdowns. Year and End Date unchanged.
2. Click Month dropdown → options with checkboxes. Select May + Apr → both checked, trigger shows "May +1 more". Table filters to both months.
3. Click Customer dropdown → SRP button visible (if srpClients loaded). Click SRP → SRP clients selected, table filters.
4. Click "Clear" in any dropdown → resets to All.
5. Smridhi profile → Therapist locked to Smridhi Grover.

- [ ] **Step 11: Commit**

```bash
git add dashboard_src.html
git commit -m "feat: replace Month/Therapist/Customer filters with multi-select; add SRP shortcut"
```

---

## Task 8: Encrypt and Verify

**Files:**
- Run build script (check for existing encrypt command in repo)

- [ ] **Step 1: Encrypt dashboard**

```bash
cd /Volumes/D/GIT/bmv && npx staticrypt dashboard_src.html -d . --template-title "Capybara Care — Revenue Dashboard"
```
Or check for existing encrypt script:
```bash
cat .staticrypt.json
```
Use whichever password/command was used before.

- [ ] **Step 2: Open `dashboard.html` and verify all features**

1. Referrals tab visible, summary loads with correct sources and amounts
2. Dr. Garima row shows owed amount (4 clients × 30% per session)
3. Naina row shows "Info only" badge
4. Tushar row shows "Settled"
5. Click Dr. Garima → detail view with KPIs and 4 client rows
6. Multi-select filters work correctly across all tabs
7. SRP button (if Master sheet public) selects SRP clients

- [ ] **Step 3: Final commit**

```bash
git add dashboard.html dashboard_src.html
git commit -m "feat: encrypt dashboard with referral tab and multi-select filters"
```

---

## Self-Review

**Spec coverage:**
- ✅ Clients sheet fetched (Task 2) → `clientsMap`
- ✅ Master!J fetched (Task 2) → `srpClients`
- ✅ Referral computation: one-time/per-session × fixed/percent × cap (Task 3)
- ✅ Referral tab summary + detail (Task 4+5)
- ✅ Info-only sources shown (computeReferralData returns null accrued, rendered as Info)
- ✅ Paid/Owed per client (clientEntry.paid from record['Paid'])
- ✅ Multi-select: Month, Therapist, Customer (Task 6+7)
- ✅ Year stays single-select (Task 7, only year/calendar/endDate listeners updated)
- ✅ SRP button inside Customer dropdown (Task 6, buildMultiSelect hasSrp logic)
- ✅ Referral tab uses lifetime data (computeReferralData uses allRows, no filters applied)
- ✅ applyFilters, computeKPIs, buildMonthlyTrend all updated for array filters (Task 7)

**Placeholder scan:** None found.

**Type consistency:** `msSelections.month/therapist/customer` are arrays throughout. `getFilterValues()` returns `month: string[]` etc. All consumers use `.length > 0` and `.includes()`. Consistent.
