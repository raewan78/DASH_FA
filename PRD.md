# PRD — FT Case Dashboard

**Product Requirements Document**  
Version: 1.4 | Created: 2026-07-01 | Last updated: 2026-07-03

---

## 1. Overview

| Item | Detail |
|------|--------|
| Product Name | FT Case Dashboard |
| Target Users | PE Team (Admin), Cross-functional teams & Management (Viewer), External stakeholders (Guest) |
| Platform | Web (single HTML file, Vanilla HTML/CSS/JS + Canvas API) |
| Data Storage | JSONBin.io (primary) → localStorage `FT_CASES` (fallback) |
| Hosting | GitHub Pages (static) |
| Supported Environments | Chrome / Edge latest, desktop |
| Reference | BRD v1.3 |

---

## 2. User Roles

| Role | Entry Method | Permissions |
|------|-------------|-------------|
| Admin | ID: `admin` / PW: `qwerty0987` | Register / Edit / Delete cases, configure JSONBin, force sync |
| Viewer | ID: (future) | View only (read-only) |
| Guest | Click "Guest Access (View only)" | View only (read-only), no login required |

---

## 3. User Stories

| ID | As a… | I want to… | So that… |
|----|--------|------------|----------|
| US-01 | Admin | Register a new FT Case | The team can track the case status |
| US-02 | Admin | Edit case details | I can reflect actual completion dates and stage changes |
| US-03 | Admin | Delete a completed case | The dashboard stays up to date |
| US-04 | All roles | Filter cases by status | I can quickly review Open / Close cases separately |
| US-05 | All roles | Auto-calculate FA Stage target dates | I know accurate deadlines without manual calculation |
| US-06 | All roles | See all Open Case timelines at a glance | I can identify progress and delays across all cases |
| US-07 | All roles | Access the latest data from any device | I can check status on the go |

---

## 4. Functional Requirements

### FR-01 Login & Session

- Admin: ID/PW input → `sessionStorage.isAdmin = true` → redirect to dashboard
- Guest: click "Guest Access (View only)" → `sessionStorage.isGuest = true` → redirect (read-only)
- Failed login: show "Incorrect ID or password.", clear password field
- Successful login: enter dashboard, auto-load JSONBin data
- Logout: clear `sessionStorage`, redirect to login
- Login screen note: "Access restricted to authorized users." shown below login button

### FR-02 Role-Based UI

| UI Element | Admin | Viewer / Guest |
|------------|-------|----------------|
| Add Case button | Shown | Hidden |
| Edit modal Save button | Active | Hidden (Close only) |
| Edit modal Delete button | Shown | Hidden |
| ⚙ JSONBin config button | Shown | Hidden |
| Role badge | ADMIN (orange) | VIEWER / GUEST (grey) |

Login screen buttons:
- `Admin Login` — opens password input
- `Guest Access (View only)` — enter read-only mode without password

### FR-03 Cloud Sync (JSONBin.io)

- Hardcoded Master Key and Bin ID; no user setup required
- On page load: `GET /v3/b/:id/latest` → auto-load latest data
- On Admin save/delete: `PUT /v3/b/:id` → immediate push
- Fetch failure: fallback to localStorage → then empty state
- Browser cache bypass: `cache: 'no-store'`, fetch timeout 5s (AbortController)

**Cloud Storage Constants:**
```
JSONBIN_BIN_ID     = '6a462009f5f4af5e29529336'
JSONBIN_MASTER_KEY = '$2a$10$rSG0Z.2fIo2rLEPCi16LqOygPyzapnBZPPVcIs0hmaEOClSAM/hwK'
```

### FR-04 Sync Status Display

Header `sync-status` span shows real-time state:
- Loading: `Connecting…`
- Success: `Connected!` / `N records synced` (auto-dismisses after 3s)
- Failure: `✗ Sync failed` (red)

### FR-05 Effective Today (`getEffectiveToday()`)

- Returns today's date if today is a working day
- If today is a non-working day (weekend or public holiday), returns the **next working day**
- Used as D-0 reference for open cases
- Used for "Next expected report" D-Day calculation

### FR-06 Cache Prevention & Auto-Reload

HTML `<head>` includes meta tags (advisory, may be ignored by CDN/server):
```html
<meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">
<meta http-equiv="Pragma" content="no-cache">
<meta http-equiv="Expires" content="0">
```

**ETag-based auto-reload** (authoritative mechanism for GitHub Pages):  
An inline script in `<head>` fires on every page load:
1. `fetch(location.href, { method: 'HEAD', cache: 'no-store' })` → reads `etag` or `last-modified` response header
2. Compares to value stored in `sessionStorage._ftver`
3. If stored value exists and differs → `sessionStorage.setItem` + `location.reload()`
4. If not yet stored → sets `sessionStorage._ftver` with current value (baseline for next load)

This ensures users on other devices automatically receive the latest deployment without a manual refresh.

### FR-07 Header Layout

```
LIVE● | [ADMIN/VIEWER/GUEST badge] | [sync status] | [⚙ Admin only]
```

- LIVE: animated green dot indicating live operation
- ⚙ button: Admin only, opens JSONBin config modal

### FR-08 KPI Bar (5 cards)

| Card | Calculation |
|------|------------|
| Total | `CASES.length` |
| Open | Count where `status === 'Open'` |
| Close | Count where `status === 'Close'` |
| High Priority | Count where `priority === 'H'` |
| Returned Units | Sum of `unitCount` across all cases |

### FR-09 Case Table Columns

| Column | Field | Sortable | Notes |
|--------|-------|----------|-------|
| Case No. | caseNo | ✓ | Clickable → detail screen |
| Customer | customer | ✓ | |
| Product | product | ✓ | |
| Arrived | arrivedDate | ✓ | Units return date MM/DD |
| FA Stage | faStage | ✓ | Stage label badge |
| Priority | priority | ✓ | H / M / L badge |
| Status | status | ✓ | Open / Close badge |
| Progress | — | ✗ | Visual bar: completed / total stages |
| Timeline | — | ✗ | Mini Gantt: case-level overview |

> **Operator column removed.** No responsible engineer field is displayed or stored.

Filter tabs: **All** / **Open** / **Close** (default: All)  
Search: real-time filter on `caseNo` + `description` (case-insensitive)  
Row click: navigate to detail screen (`title="Click for details"`)

### FR-10 Add Case Modal (Admin only)

Modal title: **"Add Case"**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| Case No. | text | ✓ | Unique |
| MCP | checkbox | | MCP flag; triggers MCP mode when checked (see FR-23) |
| Customer | text | ✓ | |
| Product | text | ✓ | |
| Units Arrived Date | date | ✓ | Triggers auto target date calculation |
| Priority | toggle H/M/L | ✓ | Default: M |
| Status | toggle Open/Close | ✓ | Default: Open |
| FA Stage | button group | ✓ | 7 stages (3 in MCP mode) |
| Actual Date (per stage) | date | | |

Submit button: **"Add"** | Cancel button: **"Cancel"**

### FR-11 Target Date Auto-Calculation

- Triggered on `arrivedDate` change
- `addWorkingDays(date, n)`: skips weekends + `KR_HOLIDAYS` set
- Korean holidays: static Set of YYYY-MM-DD strings (2025–2027), including substitute holidays
- `TARGET_OFFSETS` per stage (§ 3 FA Stage Definitions)
- Displays calculated target dates instantly in modal

### FR-12 Edit Case Modal

- Same layout as Add modal, pre-filled with case data
- Includes MCP checkbox; state preserved and applied to Gantt on open
- Delete button (Admin only): `confirm()` dialog → delete → sync → re-render
- Save button: **"Save & Sync"** — saves and triggers cloud sync
- Cancel/Close: **"Cancel"** / **"Close"**

### FR-13 FA Stage Target Dates (in modal Gantt preview)

- Canvas-based Gantt preview inside Add/Edit modal
- Target ◆ (blue), Actual ● (green/red), Today (orange dashed line)
- Updates live as Actual Date inputs change

### FR-14 Export / Import

- Export: downloads `ft_cases_YYYYMMDD.json`
- Import: reads JSON → validates structure → overwrites `CASES` → re-renders
  - Invalid structure: "Not a valid FT Cases JSON."
  - Parse error: "JSON parse error: please check the format."

### FR-15 Copy to Clipboard

- 📋 **Copy** button on detail screen for Case No.
- On success: changes to "✓ Copied" for 1.5s then reverts

### FR-16 Case Detail Screen

- Navigated by clicking a case table row
- `detailCaseId` stored as module-level JS variable (not DOM hidden input)
- `isClosed` state read from `CASES` array via `detailCaseId`
- Back button returns to dashboard

Zones:
| Zone | Content |
|------|---------|
| Header | Case No., Customer, Product, FA Stage, Unit Count, Status |
| Progress | Step dots per FA Stage: filled = completed, current = highlighted |
| Single-case Gantt | See FR-17, FR-18 |
| Next Report | Next FA Stage label + D-Day (e.g., "Final 8D — D+3") |
| Case Info | All field values + Edit button (Admin only) |

### FR-17 Label Orientation

All FA Stage labels on all Gantt charts rendered **horizontally** (no rotation).

### FR-18 Single-Case Gantt (`renderGanttChart`) — D+N Axis

**Reference points:**

| Case Status | D Reference | Label |
|-------------|-------------|-------|
| Open | `getEffectiveToday()` | D-0 = today |
| Close | Arrived date (`base`) | D+0 = arrived |

**Counter-based calculation (`wd_axis`):**
- `axisRef = isClosed ? base : today`
- Count working days from `originDay2` to `axisRef` → `wdToRef`
- Initialize `wd_axis = -wdToRef`; increment by 1 per working day
- At `axisRef`: `wd_axis === 0`

**Label format:**
- `wd_axis === 0`: `D+0` (closed) or `D-0` (open)
- `wd_axis > 0`: `D+N`
- `wd_axis < 0`: `D-N` (e.g., `D-2`)

**Tick rendering:**
- Every working day in range gets a tick mark
- Today tick: orange (`C.today`), 1.5px stroke
- Arrived tick: orange-red (`C.arrived`), 1.5px stroke
- Target date ticks: accent color, bold label
- `toX()` returns `null` for out-of-range dates (no clipping artifact)
- NONWD_RATIO = 0.25 (non-working days rendered at 25% width)

### FR-19 Single-Case Gantt — X-Axis Range

| Case Status | `originDay2` | `endDay2` |
|-------------|-------------|----------|
| Open | arrived − 1 working day | max(today + 5 wd, last target date + 5 wd) |
| Close | arrived − 1 working day | last FA stage with actual data + 2 calendar days |

All working days within range shown as tick marks.

### FR-20 Overall Gantt (`renderOverallGantt`) — Per-Case Mini Axis

Rendered in the Timeline column of the case table.

- Each case row has its own independent D+N mini axis (today-relative):
  - `wd_i = 0` at today (D-0, orange)
  - Before today: D-N (grey), after today: D+N (grey)
  - Counter-based: `wd_i` incremented per working day from `tickStart`
- Arrived date line: orange-red (`C_arrived`)
- Today line: orange (`C.today`)
- NONWD_RATIO = 0.25

### FR-21 STAGE_NEXT Routing

| Current Stage | Next Stage |
|---------------|-----------|
| initial | interim-1 |
| interim-1 | interim-2 |
| interim-2 | interim-3 |
| interim-3 | interim-4 |
| interim-4 | **final-8d** |
| final-4d | **final-8d** |
| final-8d | (end — no next report) |

> Both Interim (4) and Final 4D route directly to Final 8D.

### FR-22 Next Expected Report Section

Below the single-case Gantt on the detail screen:
- Shows: Next FA Stage label + Target Date + D-Day countdown
- D-Day = working days from today to target date
- Color: green (D-4+), orange (D-3 to D-0), red (overdue)
- Hidden if current stage = Final 8D (end of chain)

### FR-23 MCP Mode

When a case's MCP checkbox is checked, the following rules apply across all UI:

**FA Stage visibility:**
- Shown: Initial, Interim (1), Interim (2)
- Hidden: Interim (3), Interim (4), Final 4D, Final 8D

**Target date rules:**

| Stage | Target Date |
|-------|------------|
| Initial | None — no target (offset label "+1 wd" hidden) |
| Interim (1) | Initial Actual date + 4 working days; label shows "+4 wd*" |
| Interim (2) | Initial Actual date + 7 working days; label shows "+7 wd*" |

Target dates for Interim (1) and Interim (2) are recalculated live whenever the Initial Actual date input changes (`onInitialActualChange(prefix)`).

**Modal Gantt preview (`renderGanttChart`):**
- Canvas height shrinks to 3 rows (instead of 6 for normal mode)
- Rows: Initial (no target diamond), Interim (1), Interim (2)
- `targetDate: null` for Initial → target diamond suppressed; `toX()` null-guarded throughout

**Overall Gantt (`renderOverallGantt`):**
- MCP cases (`c.mcpFlag === true`) render only 3 stages
- Stage target dates computed using `_addWdG(initActual, n)` instead of `TARGET_OFFSETS`
- `null` targetDate values guarded in `allMs` accumulation and marker drawing

**`onMcpChange(prefix)` handler** triggers on checkbox change:
- Hides/shows stage rows and FA Stage buttons
- Resets Initial target display to `—`
- Toggles visibility of Initial "+1 wd" offset label
- Calls `renderMcpInterimTargets(prefix)` or `renderTargetDates(prefix, returnDate)` depending on state
- Calls `renderGanttChart(prefix)` to redraw

---

## 5. FA Stage Definitions

| Stage Key | Display Label | Working Days from Arrived |
|-----------|--------------|--------------------------|
| initial | Initial | +1 |
| interim-1 | Interim (1) | +2 |
| interim-2 | Interim (2) | +6 |
| interim-3 | Interim (3) | +8 |
| interim-4 | Interim (4) | +13 |
| final-4d | Final 4D | +13 |
| final-8d | Final 8D | +16 |

---

## 6. Data Model

### Case Object

```js
{
  id:          string,         // UUID, PK
  caseNo:      string,         // e.g., "FT-2024-001"
  customer:    string,
  product:     string,
  model:       string,
  arrivedDate: 'YYYY-MM-DD',  // Units return date
  unitCount:   number,
  mcpFlag:     boolean,        // true = MCP mode active
  priority:    'H' | 'M' | 'L',
  status:      'Open' | 'Close',
  faStage:     string,         // Current FA stage key
  actualDates: {               // Actual completion dates per stage
    initial:     'YYYY-MM-DD' | '',
    'interim-1': '...',
    'interim-2': '...',
    'interim-3': '...',
    'interim-4': '...',
    'final-4d':  '...',
    'final-8d':  '...',
  },
  description: string,
  createdAt:   'ISO string',
}
```

> **Operator field removed.** No `operator` property exists in the data model.

---

## 7. Color System

### Stage Colors

| Stage | Color |
|-------|-------|
| Initial | `#38BDF8` (sky blue) |
| Interim (1–3) | `#4DB3F5` (blue) |
| Interim (4) | `#F09040` (orange) |
| Final 4D | `#1A8FE3` (accent blue) |
| Final 8D | `#F09040` (orange) |

### Status Colors

| State | Color |
|-------|-------|
| On time / Actual ≤ Target | `#22C78A` (green) |
| Overdue / Actual > Target | `#F25069` (red) |
| Today / D-0 reference | `#F09040` (orange) |
| Arrived / D+0 reference | orange-red |
| Target diamond | `#1A8FE3` (accent blue) |

---

## 8. Screen Layout

```
[Login Screen]
  Admin Login (password) | Guest Access (View only)
  "Access restricted to authorized users."
        ↓
[Dashboard]
┌──────────────────────────────────────────────────────────────┐
│  LIVE● | [ADMIN/VIEWER/GUEST] | [sync status] | [⚙]         │
├──────────────────────────────────────────────────────────────┤
│  KPI: Total | Open | Close | High Priority | Returned Units  │
├──────────────────────────────────────────────────────────────┤
│  [All | Open | Close] filter tabs    [Search]                │
│  Case No. | Customer | Product | Arrived | Stage |            │
│           Priority | Status | Progress | Timeline            │
│  [+ Add Case] (Admin only)                                   │
├──────────────────────────────────────────────────────────────┤
│  Open Case Timeline (overall Gantt)                          │
│  ┌─ Case ID ─[Arrived◆]──[Init◆]──[I(1)◆]──…──────────── ─┐ │
│  │  D-1   D-0  D+1  D+2  D+3  D+4  D+5  …                  │ │
│  └───────────────────────────────────────────────────────── ┘ │
└──────────────────────────────────────────────────────────────┘

[Case Detail Screen]
┌──────────────────────────────────────────────────────────────┐
│  ← Back | Case No. | Customer | Product | Status | Units    │
├──────────────────────────────────────────────────────────────┤
│  Progress: ●──●──●──○──○──○──○  (stage dots)                 │
├──────────────────────────────────────────────────────────────┤
│  Single-Case Gantt (D+N axis, working days)                  │
│  Initial  ◆ target  ● actual                                 │
│  Interim1 ◆ target                                           │
│  …                                                           │
│  D-2  D-1  D-0  D+1  D+2  D+3  …                            │
├──────────────────────────────────────────────────────────────┤
│  Next Report: Final 8D — D+3                                 │
└──────────────────────────────────────────────────────────────┘
```

---

## 9. Non-Functional Requirements

| # | Requirement | Spec |
|---|-------------|------|
| NFR-01 | Language | UI fully in English (all labels, messages, buttons, placeholders) |
| NFR-02 | Browser support | Chrome / Edge latest; no IE |
| NFR-03 | Min. resolution | 1280px width |
| NFR-04 | Performance | List render < 300ms; Gantt render < 500ms; JSONBin timeout 5s |
| NFR-05 | Accessibility | Keyboard-navigable modals; ARIA labels on icon-only buttons |
| NFR-06 | Security | No sensitive data in URL; Master Key in source only |
| NFR-07 | Offline mode | Falls back to localStorage if cloud unavailable |

---

## 10. Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-07-01 | Initial draft |
| 1.1 | 2026-07-02 | Added FR-03~06 cloud sync, Guest role, export/import |
| 1.2 | 2026-07-02 | Added FR-15 (isClosed from CASES array), FR-16 progress dots, FR-17 horizontal labels, FR-18 D+N axis initial spec, FR-19 closed case range, NONWD_RATIO |
| 1.3 | 2026-07-03 | Full English rewrite to match English-only UI; FR-05 getEffectiveToday() added; FR-09 column renames (Operator/Status/Priority/Progress/Timeline); FR-18 revised — counter-based wd_axis, open=today-relative D-0, closed=arrived-relative D+0; FR-21 STAGE_NEXT corrected (Interim 4 → Final 8D, Final 4D → Final 8D); FR-06 cache prevention meta tags added; FR-02 Guest button text updated; FR-20 overall Gantt mini axis clarified (today-relative per row) |
| 1.4 | 2026-07-03 | FR-06 extended with ETag-based auto-reload (HEAD fetch + sessionStorage); FR-09 Operator column removed; FR-10 Operator field removed, MCP checkbox added; FR-12 MCP checkbox note added; FR-23 MCP Mode new FR (target rules, Gantt behavior, handler logic); Data model updated (operator removed, mcpFlag added); Screen layout updated (Operator column removed); Reference updated to BRD v1.3 |
