# Higher Ed Benchmarks Prototype — Summary

Owner: Mohib Ullah (Forella Group) · Status: working prototype, static data snapshot · Last updated: Oct 15, 2026

## 1. Purpose
Forella's benchmark data lives in `Benchamrking tool.xlsx`. Finding comparable projects and their $/GSF means filtering the spreadsheet by hand. This prototype puts a filterable UI on top of it: pick a set of comparable projects, normalize them to a common year and location, and read off averages, a median, the spread, and a system-level (UniFormat) breakdown.

The pilot covers **Higher Ed projects only**. The plan is (1) prove the UI on a static snapshot (this), (2) switch to a live data file, (3) expand to other markets.

## 2. Features
- **Filters:** free-text search (name, city, facility type), state, facility category, facility type (narrows by category), project type, source (Bid / Forella estimate / Blended estimate), bid/estimate year range, GSF range, adjusted $/GSF range, priority.
- **Cost scope:** $/GSF for Total project, Architectural, MEP, Structure, Sitework, or a Custom set of UniFormat elements. For scopes other than Total, there's a toggle to include or exclude prorated markups.
- **Cost basis:** escalate to a target year (As recorded, 2026–2030) and normalize to a target city by CCI (or keep each project's own location).
- **Breakouts:** combined projects (e.g. building + garage, new + renovation) also have component rows. These are hidden by default so nothing is double-counted. A checkbox shows them.
- **Project list:** sortable by name, GSF, cost, $/GSF, and % vs. the selection's average. Click a row to see its number, location and CCI, date, source, procurement and CM, as-recorded cost, adjustment factor, notes, and a UniFormat $/GSF bar breakdown grouped by system.
- **Averages panel:** average $/GSF (simple mean, with the GSF-weighted figure underneath), average GSF with range, average and total cost, project count split by bids and estimates, median $/GSF with range. Also average $/GSF by system (click to switch scope), a dot strip of the $/GSF spread (hover for the name, click to open), and the mix by facility category.
- Light and dark mode, and it works on phones.

## 3. Tech stack
- One self-contained HTML file: vanilla JavaScript and CSS, no framework, no build step, no server.
- The data is embedded as a JSON object (`const DATA = {...}`) inside the page. The same data is in `data/benchmark-data.json`.
- The only external dependency is IBM Plex fonts from Google Fonts. Without internet it falls back to system fonts.
- Currently published as a claude.ai artifact. `index.html` also opens directly in a browser.

## 4. Data model
`DATA` has four parts:

| Key | Contents |
|---|---|
| `projects` | 123 rows: 94 whole projects and 29 breakouts |
| `uni` | The 21 UniFormat II Level-2 elements, A10 Foundations through G40 Site Electrical Utilities. The order matches each project's `u` array. |
| `esc` | Annual escalation factors by year, 2014–2043 (30 years), from the workbook's escalation table |
| `cities` | 136 `[state, city, CCI]` entries from the workbook's `Const_Cost_Index` sheet |

Fields on each project:
`id`, `num` (workbook project #, null for breakouts), `comp` (true = breakout), `parent` (the parent's `num`), `name`, `fac` (facility type), `cat` (facility category), `ptype` (new / renovation / addition + renovation …), `st`, `city`, `date`, `yr`, `gsf`, `total` (all-in cost incl. markups, $), `src` (raw source text), `srcg` (grouped source), `pri` (priority 1–3), `proc` (procurement method), `cm` (CM/GC), `cci` (project location's CCI), `notes`, `u` (21 UniFormat trade-cost values in $, before markups).

Coverage: states DC, FL, MD, NC, PA, VA, WV. Years 2015–2028. Categories: Academic, Admin, Athletic, CUP, Dining, Housing, Law, Library, Parking, Performing Arts, STEM, Student Services, Vocational.

## 5. How results are calculated
**Adjustment factor** for each project:
- *Escalation* follows the workbook: multiply by `esc[y]` for every year after the project year, up to and including the target year. Projects dated after the target year are **not** de-escalated. Projects with no date are not escalated.
- *Location:* × (target CCI ÷ project CCI) when a target city is picked.

**Scope cost:**
- *Total* = `total` (the workbook's TOTAL $ row, markups included).
- *A system scope* = the sum of that scope's UniFormat elements from `u`. With markups on, that sum is scaled by `total ÷ sum(u)`, so GCs, fee and contingency are spread pro rata. Systems then add up to the total.
- Scope groups: **Structure** A10–B10; **Architectural** B20, B30, C10–C30, D10, E10–E20, F10–F20; **MEP** D20–D50; **Sitework** G10–G40.

**Per project:** adjusted cost = scope cost × factor; $/GSF = adjusted cost ÷ GSF. Projects with zero cost in the chosen scope are left out, and the count says how many.

**Selection statistics:**
- Average $/GSF is the **simple mean** of each project's $/GSF. The GSF-weighted figure (Σcost ÷ ΣGSF) is shown underneath.
- The median and the "vs avg" % (project $/GSF ÷ mean − 1) are computed the same way.

## 6. What's unfinished / known issues
- **Static snapshot.** The data is pasted into the page, so changes to the workbook don't show up until the data is re-exported. Next step: read a live data file (e.g. CSV/JSON exported from the workbook, or SharePoint/Excel directly).
- **Higher Ed only.** Other markets aren't loaded yet.
- **No export.** You can't yet download a filtered selection to Excel/CSV or print a comparison sheet.
- **Escalation is one-directional** (no de-escalation for future-dated projects), matching the workbook. Confirm this is the intended rule.
- **Data checks needed:**
  - *Georgetown Univ – Henle Village (#8):* the UniFormat values look shifted by a column. For example, C20 Stairs is about $11.2M, D40 Fire Protection about $17.9M and E10 Equipment about $13.2M, while B10 Superstructure is only about $1.7M on a concrete PT building. Verify against the workbook.
  - *"JMU AMR1 Housing…"* is a breakout of **JHU** AMR1 + Dining, so this is likely a typo in the source.
  - *BSU Student Union* has no date, so it's excluded whenever a year filter is set and is never escalated.
- **Not yet added:** saved filter presets, side-by-side compare of selected projects, and per-project links back to the source estimate.
