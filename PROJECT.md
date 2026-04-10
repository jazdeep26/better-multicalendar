# Calendar View — Project Reference

A single-file, client-side Google Calendar viewer optimised for therapy clinics that need to see multiple calendars side-by-side, organised by therapist or by client/child.

## Overview

| | |
|---|---|
| **Type** | Static single-page web app (no build, no backend) |
| **Files** | [index.html](index.html) (~2940 lines), [favicon.svg](favicon.svg) |
| **Stack** | Vanilla HTML / CSS / JavaScript (no frameworks, no dependencies) |
| **API** | Google Calendar API v3 + OAuth 2.0 (PKCE flow) |
| **Persistence** | `localStorage` for tokens and preferences; all events live in Google Calendar |
| **Domain** | Therapy/clinic scheduling — events follow the convention `"ChildName - TherapyType - Therapist"` |

## Tech notes

- Single self-contained `index.html` — CSS, HTML and JS are all inline.
- No build step. Open in a browser (or serve statically) and it runs.
- OAuth 2.0 Authorization Code flow with PKCE; refresh tokens are persisted in `localStorage` for silent re-auth.
- Hardcoded Google OAuth `CLIENT_ID` / `CLIENT_SECRET` at [index.html:1033-1034](index.html#L1033-L1034). ⚠️ Note: shipping a client secret in client-side code is unusual — Google issues these for "Web application" type credentials and they aren't truly secret in this flow, but it's worth being aware of.

## Top-level UI

- **Auth screen** — "Sign in with Google" landing if no token is stored.
- **Top bar (row 1)** — view switcher (Day / 3 Days / Week), client-view buttons, prev/today/next, date label, refresh, dark-mode toggle, settings gear, sign-out.
- **Top bar (row 2)** — chips for whatever entity the current view groups by (calendars, clients, or session types).
- **Calendar grid** — sticky header row, sticky time gutter, day columns subdivided into lanes, absolutely-positioned event blocks, "now" line for today.
- **Event modal** — create/edit form with title, start/end, calendar, location, description, tentative toggle, and Meet/Teams/Zoom link surfacing.
- **Settings panel** — slide-in panel for calendar enable/disable, view feature flags, and CRUD permission toggles.

## Three view modes

The grid renders one of three completely separate layouts based on toggle state:

### 1. By Calendar (default) — [`renderGrid()`](index.html#L1491)
- Each visible day shows one **lane per active calendar**.
- Lane label and event border use the calendar's Google color.
- This is the default if neither client mode is active.

### 2. By Client — [`renderClientGrid()`](index.html#L1783)
- Gated by **Settings → Enable By Client View**.
- Each day shows one **lane per client (child)**, derived by parsing event titles `"Name - Type"`.
- Events whose first segment contains a comma or `++` are treated as **shared/group events** (e.g. `"Aria, Ben - Group"`) and span all client lanes for that day as a single absolutely-positioned overlay.
- Client pill colors are deterministically generated from a hash of the name ([`cvColorForName`](index.html#L1763)).
- Only clients that actually have a non-shared event on a given day get a lane that day.

### 3. One Client — [`renderSingleClientGrid()`](index.html#L2094)
- Gated by **Settings → Enable One Client View**.
- Focuses on a single child via a `<select>` dropdown.
- Top bar exposes pills to filter by **session type** (the `TherapyType` segment) and a **"Hide therapist"** checkbox that strips the `parts[2]` segment from displayed titles.
- Includes group events where the selected child appears in the comma-list.

The three modes are mutually exclusive — turning one on disables the others ([index.html:2840](index.html#L2840), [index.html:2858](index.html#L2858)).

## Permissions model

Settings → Permissions exposes four independent toggles ([index.html:2621](index.html#L2621)) that gate write operations across the whole app:

| Toggle | Effect |
|---|---|
| `add` | Click-to-create on empty lane areas (`renderGrid` attaches click handlers to lanes) |
| `edit` | Click event opens edit modal; resize handles attached |
| `move` | `mousedown`/`touchstart` on event body initiates drag-to-move with ghost preview |
| `delete` | Delete button appears in edit modal |

All four are off by default — the app is read-only until the user explicitly opts in.

## Data flow

1. **Boot** ([index.html:2903](index.html#L2903)) — restore dark mode + view from prefs, check URL for OAuth `code`, attempt silent token reuse.
2. **Sign-in** — PKCE flow → authorization code → token exchange → store access + refresh tokens.
3. **`initApp()`** ([index.html:1318](index.html#L1318)) — fetch full calendar list, restore prefs, kick off `refreshEvents()`.
4. **`loadAllEvents()`** ([index.html:1302](index.html#L1302)) — for each *enabled* calendar, fetch events for `[anchor−7d, anchor+viewDays+7d]` in parallel via `Promise.allSettled`. Results land in `cachedEvents[calId]`.
5. **`renderGrid()`** — pure render off `cachedEvents` + current view state. No re-fetch on view changes; navigation just re-lays out the cache.
6. **Mutations** (create/edit/move/delete/resize) — write through to Google Calendar API, then patch `cachedEvents` in place and re-render. Drag-and-drop does an **optimistic update + revert on failure**.

## Event title parsing convention

The app assumes events follow `"ChildName - TherapyType - Therapist"`:

- `parseEventTitle()` ([index.html:1732](index.html#L1732)) splits on `" - "`.
- A "shared" event is one where the child slot contains a comma or `++` ([`cvIsShared`](index.html#L1747)) — used to detect Group / Lunch / Break events that involve multiple kids.
- The reminder-message generator ([`buildReminderMessage`](index.html#L2266)) builds a formatted "Hi! {child}'s sessions today/tomorrow/on {date}: …" text block listing all of one child's sessions for a day, including any group events they're part of. Triggered by the small copy icon on every event — copies to clipboard.

## Tentative events

- Stored as a `"Tentative"` line prepended to the event description in Google Calendar (so it survives a round-trip and is human-readable in the native UI).
- Detected on read via a regex on the description ([index.html:1279](index.html#L1279)).
- Rendered with diagonal stripe overlay, dashed left border, and reduced opacity ([CSS at index.html:604](index.html#L604)).
- Toggleable in the edit modal via a switch.

## Meeting link surfacing

When an event has either `hangoutLink` or a Meet/Teams/Zoom URL in its description:

- A 📹 icon appears on the event card (clicking opens the link in a new tab, stops propagation so it doesn't trigger edit).
- The edit modal shows a dedicated row with the link and a Copy button.
- Detection regex matches `meet.google.com`, `teams.microsoft.com`, `teams.live.com`, `zoom.us/j`, `usNN.zoom.us/j` ([index.html:1283](index.html#L1283)).

## Drag, resize, and reorder

- **Calendar/client pills** — HTML5 drag-and-drop to reorder; persists to `calOrder` in prefs.
- **Events — move** — `mousedown`/`touchstart` on event body creates a ghost element, follows pointer with 15-min snapping, on release patches start/end via API. Touch has a 150 ms hold delay to distinguish tap from drag.
- **Events — resize** — top/bottom 6px handles, snap to 15-min grid, only resize the edge being dragged.
- All of the above are guarded by the matching permission flag.

## Preferences

Persisted to `localStorage` under key `calview_prefs` ([index.html:1059](index.html#L1059)):

```js
{
  darkMode, viewDays, calOrder,
  activeCalendars, enabledCalendars, permissions,
  clientViewEnabled, clientViewMode, activeClients,
  singleClientEnabled, singleClientMode,
  singleClientName, singleClientFilter, singleClientHideTherapist
}
```

The distinction between `enabledCalendars` (gates whether we sync the calendar at all) and `activeCalendars` (gates whether it's shown in the current view) is important — disabling in settings drops it from the cache and from the network sync; toggling its pill just hides it from the grid.

## Layout algorithm — `layoutEvents()`

[index.html:1461](index.html#L1461) — packs overlapping events into columns within a single lane:

1. Sort by start time, then by descending duration.
2. Walk events grouping into clusters of mutually overlapping events.
3. Within each cluster, greedily assign each event to the first column whose last event has already ended.
4. All events in a cluster get the same `numCols` so they share the lane width evenly.

## Conventions / things to know

- **Display window**: 7 AM – 8 PM hardcoded (`START_HOUR`, `END_HOUR` at [index.html:1457](index.html#L1457)).
- **Grid resolution**: hour rows of 64 px on desktop, 52 px on mobile (CSS variables at [index.html:11](index.html#L11)).
- **Time zone**: events are written using `toOffsetISO()` which produces `YYYY-MM-DDTHH:mm:ss±HH:MM` in **local** time, never UTC ([index.html:2453](index.html#L2453)).
- **Recurring events**: read with `singleEvents=true` so the API expands them; mutations always target the single occurrence (`?sendUpdates=none` to avoid notification spam, never touch the recurrence master).
- **Cross-calendar move**: implemented as create-in-destination + delete-from-source, since the Calendar API has no native "move" for events ([index.html:2493](index.html#L2493)).
- **Now-line**: red horizontal bar with a circular dot, refreshed every minute via `setInterval` ([index.html:2894](index.html#L2894)).
- **Dark mode**: a `body.dark` class with parallel CSS rules for almost every component.

## File map

| Section | Lines | What lives there |
|---|---|---|
| `<style>` | [8–861](index.html#L8-L861) | All CSS — layout, components, dark mode, responsive, modals, drag |
| Auth screen markup | [865–875](index.html#L865-L875) | Sign-in landing |
| App markup | [877–929](index.html#L877-L929) | Top bar, grid container |
| Modal markup | [933–981](index.html#L933-L981) | Event create/edit modal |
| Settings markup | [983–1025](index.html#L983-L1025) | Slide-in settings panel |
| Config + state | [1028–1056](index.html#L1028-L1056) | OAuth constants, in-memory globals |
| Prefs persistence | [1058–1084](index.html#L1058-L1084) | `localStorage` save/load |
| Date helpers | [1086–1109](index.html#L1086-L1109) | Pure date utilities |
| OAuth (PKCE + refresh) | [1111–1253](index.html#L1111-L1253) | Login, callback, silent refresh, `apiGet` / `apiWrite` |
| Calendar/event fetch | [1255–1315](index.html#L1255-L1315) | List + paginated event fetch + parallel sync |
| App init / refresh | [1317–1387](index.html#L1317-L1387) | Boot orchestration |
| Toggles render | [1389–1451](index.html#L1389-L1451) | Calendar pill rendering + drag reorder |
| `renderGrid` (by-calendar) | [1491–1690](index.html#L1491-L1690) | Default grid renderer |
| Tooltip + UI helpers | [1692–1740](index.html#L1692-L1740) | Hover tooltip, color helpers, title parser |
| Client view | [1742–2005](index.html#L1742-L2005) | By-client grid, toggles, shared-event handling |
| Single client view | [2006–2264](index.html#L2006-L2264) | One-client grid, session-type filtering |
| Reminder message | [2266–2328](index.html#L2266-L2328) | "Hi! …'s sessions today: …" copy text |
| Modal logic | [2357–2567](index.html#L2357-L2567) | Open/save/delete, including cross-calendar move |
| Settings panel | [2569–2627](index.html#L2569-L2627) | Calendar enable/disable + permission wiring |
| Drag/resize | [2629–2814](index.html#L2629-L2814) | Move + resize-top + resize-bottom with optimistic updates |
| Event handlers | [2816–2891](index.html#L2816-L2891) | DOM wiring for buttons, toggles, view switcher |
| Boot IIFE | [2902–2939](index.html#L2902-L2939) | OAuth callback handling + initial render |

## Recent history (from `git log`)

- `e37efd8` — favicon (blue→green gradient SVG)
- `1c0c063` — "Hide therapist" checkbox in One Client view
- `3bb55a8` — removed redundant client-name labels row in by-client / one-client views
