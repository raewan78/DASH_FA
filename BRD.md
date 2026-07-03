# BRD — FT Case Dashboard

**Business Requirements Document**  
Version: 1.4 | Created: 2026-07-01 | Last updated: 2026-07-03

---

## 1. Background & Purpose

The PE (Product Engineering) team manages Field Trial (FT) Cases for units returned by customers, and must submit FA (Failure Analysis) reports for each case by defined deadlines at each stage. Previously, cases were tracked through individual spreadsheets and emails, causing the following problems:

- Difficulty identifying overdue cases in advance when multiple cases run concurrently
- Missing or miscalculated target dates per FA Stage
- Lack of team-wide visibility due to inconsistent status-sharing methods
- Deadline mismatches caused by not accounting for Korean public holidays and substitute holidays
- Inability to check status on the go when switching devices

This project builds a web-based dashboard to resolve these issues.

---

## 2. Business Goals

| # | Goal | Success Metric |
|---|------|----------------|
| B1 | Auto-calculate target dates per FA Stage to eliminate manual errors | 0 target date calculation errors |
| B2 | View all Open Case progress on a single screen | 50% reduction in weekly status-check time |
| B3 | Identify overdue (D+) cases proactively for early response | Reduced rate of overdue cases |
| B4 | Protect data integrity through Admin/Viewer role separation | 0 unauthorized data edits |
| B5 | View the same latest data from any device or browser | Cloud sync success rate ≥ 99% |

---

## 3. Stakeholders

| Role | Description | Permissions |
|------|-------------|-------------|
| Admin | PE team engineer — register, edit, delete cases and configure sync | Read + Write + Delete |
| Viewer | Cross-functional teams, management — view progress | Read-only |
| Guest | External parties accessing without login | Read-only |

---

## 4. Business Requirements

### BR-01 Case Lifecycle Management
FA Cases must be tracked in a single system from registration through Final 8D completion.  
Admin must be able to register, edit, and delete cases.  
Cases are identified by Case No. and tracked by FA Stage; no operator (responsible person) field is required.

### BR-02 Automatic FA Stage Target Date Calculation
Target dates for each FA Stage must be automatically calculated in **Korean business days** based on the Units Return Date. Public holidays, substitute holidays, and ad-hoc holidays must be reflected.

**Standard FA Stage offsets (from Units Return Date):**

| FA Stage | Business Days |
|----------|--------------|
| Initial | +1 |
| Interim (1) | +2 |
| Interim (2) | +6 |
| Interim (3) | +8 |
| Interim (4) | +13 |
| Final 4D | +13 |
| Final 8D | +16 |

**MCP Mode target date rules (see BR-07):**

| FA Stage | Target Date |
|----------|------------|
| Initial | None (no target) |
| Interim (1) | Initial Actual + 4 working days |
| Interim (2) | Initial Actual + 7 working days |
| Interim (3) and later | None (stage not applicable in MCP mode) |

### BR-03 Progress Visualization
- Display FA Stage target and actual completion dates for all Open Cases on a timeline (Gantt)
- Each case shows an independent D+N axis:
  - **Open case**: D-0 = today (negative = before today, positive = after)
  - **Close case**: D+0 = Units Arrived date
- If today is not a business day, the next business day is used as the reference (Effective Today)
- Display the next expected report and its D-Day per case
- MCP cases display only their applicable stages (Initial, Interim 1, Interim 2) on all Gantt views

### BR-04 Priority Management
Case priority (H / M / L) must be configurable to identify high-risk cases.

### BR-05 Cross-Device Data Sharing
Any data change made by Admin on any device must be immediately visible to the entire team (including Viewers and Guests) from other devices.  
Auto-sync must occur via URL access only, without additional installation or configuration.

### BR-06 Data Persistence
Registered case data must be retained even after the browser is closed or the device is changed.  
The system must detect when a new version of the page has been deployed (via ETag comparison) and automatically reload to prevent stale cache issues across devices.

### BR-07 MCP Mode
Certain cases follow a Modified Complaint Process (MCP) with a different reporting schedule.

- Admin can flag a case as MCP at registration or edit time (checkbox field)
- In MCP mode:
  - Only Initial, Interim (1), and Interim (2) FA stages apply
  - Initial report has no target date
  - Interim (1) target = Initial Actual completion date + 4 working days
  - Interim (2) target = Initial Actual completion date + 7 working days
  - Stages Interim (3), Interim (4), Final 4D, and Final 8D are hidden from all UI views
- MCP status is stored per case and reflected in all Gantt visualizations

---

## 5. Scope

**In Scope**
- FT Case registration, view, edit, delete
- Automatic target date calculation (Korean business days)
- MCP mode with alternative target date rules
- Timeline visualization (Gantt) — independent D+N axis per case, MCP-aware
- Next expected report display
- KPI summary (Total / Open / Close / High Priority / Returned Units)
- Role-based access control (Admin / Viewer / Guest)
- JSONBin.io cloud sync (auto-save + manual sync)
- ETag-based stale-cache detection and auto-reload
- Fully English UI

**Out of Scope**
- Automated email / Slack notifications
- External SFDC system integration
- Multi-user simultaneous editing (conflict detection)
- Mobile app

---

## 6. Constraints

- Operates as a single HTML file with no separate server or database
- Cloud storage uses JSONBin.io free plan (Master Key embedded in deployment)
- Falls back to localStorage data in offline environments
- Korean holiday data managed as a static Set in source code (2025–2027)
- Minimum supported resolution: 1280px width
- GitHub Pages HTTP cache headers (`Cache-Control: max-age=600`) override HTML meta tags; ETag-based JS reload is used as the authoritative cache-bust mechanism

---

## 7. Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-07-01 | Initial draft |
| 1.1 | 2026-07-02 | Added B5 goal, Guest role, BR-01 delete permission, BR-05 cross-device requirement, BR-06 data persistence redefined, scope/constraints updated |
| 1.2 | 2026-07-03 | BR-03 D+N axis reference updated (open=today-relative, close=arrived-relative), added Effective Today rule for non-business days, UI fully converted to English |
| 1.3 | 2026-07-03 | BR-01 updated (Operator field removed); BR-02 MCP mode target dates added; BR-03 MCP Gantt rule added; BR-06 ETag auto-reload added; BR-07 MCP Mode new requirement; Scope and Constraints updated |
| 1.4 | 2026-07-03 | No business requirement changes — UI refinements only (KST clock, admin credential update); see PRD v1.5 |
