# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this tool is

A **meter-switch break-even** calculator for Thai households: *"Should I change my electricity
meter from the normal (progressive) type to a TOU (Time-of-Use) type?"* It is **not** an
investment-payback tool. The decision hinges on **when** power is consumed — TOU is only worth it if
enough usage falls in the cheap Off-Peak window (22:00–09:00 weekdays + all weekends/holidays).

The tool compares the annual bill on each meter from the same usage, then shows whether (and how
fast) the one-time meter-change fee is recovered, and tells daytime-heavy households plainly **not**
to switch.

Forked from `solar_payback_thailand` — it reuses that tool's appliance load-profile engine, TOU
period logic, CSS, and chart scaffolding, and strips out everything solar-specific (PV generation,
battery, export feed-in, RD-805 tax, IRR).

## Project Guidelines

- **Scope:** Household/residential use only — no commercial or industrial scenarios.
- **Language:** UI and all in-app text must remain in Thai. Code comments and this file stay in English.
- **No build step.** Single `index.html` with embedded CSS + JS. Open directly in a browser. CDN deps
  (Chart.js, Font Awesome, Google Fonts) loaded with `defer`.

## Running the App

No server required — open `index.html` in any modern browser. Or use the `tou_meter_payback` config
in the workspace `.claude/launch.json` (static server on port 8124).

## Architecture (single-file SPA)

### Reused from the solar tool (do not reinvent)
- `applianceMasterList`, `presetConfigurations`, `normalizeApplianceState`, `renderSingleTable`,
  `adjustQty`/`adjustHours`, `setPreset` — the weekday/holiday appliance matrix.
- `build24HourLoadProfile(qtyMap, hoursMap)` — maps appliance usage into a 24-element watt array via
  priority slot arrays. `AC_THERMAL_FACTOR` still shapes AC draw per period.
- `moveRoiDashboardToPrimaryPosition`, `switchTab`, `setChartDayType`.

### Period → TOU window mapping (the key idea)
`build24HourLoadProfile`'s slots map cleanly onto TOU windows, so the day/night split is already
computed: `p1` slots (22:00–09:00) → **Off-Peak**; `p2`+`p3` slots (09:00–22:00) → **Peak**.
`isPeakHour(h)` = `9 <= h < 22`.

### Two input modes (`setInputMode('detailed'|'quick')`)
- **Detailed (default):** the appliance matrix (`#detailedInputCard`). Off-Peak share is derived
  automatically from when appliances run.
- **Quick (`#quickInputCard`):** total kWh/month (`#quickMonthlyKwh`) + an Off-Peak share slider
  (`#quickOffPeakShare`). The weekday/holiday chart toggle is hidden in this mode.

### Calculation core
- `deriveUsageSplit()` — single source of usage figures for both modes: `annualKwh`, `annualPeak`,
  `annualOffPeakWd`/`annualOffPeakHd`, `monthlyKwh`, and representative weekday/holiday profiles.
  Detailed mode weights weekday usage ×242 and holiday ×123; quick mode lumps all Off-Peak as
  weekday Off-Peak.
- `getNormalBlocks()` + `tieredEnergyBase(monthlyKwh, blocks)` — the **progressive (step) tariff**:
  editable per-unit base rates for 0–150 / 151–400 / 401+ units, applied to **monthly** kWh. This is
  what makes the normal meter "time-blind" — only total units matter.
- `calculateMeterComparison()` — the orchestrator:
  - Normal annual = `12 × (tieredEnergy(monthlyKwh) + Ft×monthlyKwh + serviceNormal) × 1.07`.
  - TOU annual = `(peakKwh×peakBase + offPeakWd×offBase + offPeakHd×offHolBase + Ft×annualKwh) × 1.07
    + serviceTOU×12×1.07`.
  - `annualSavings = normal − TOU` (can be negative); `breakEvenYears = meterFee / annualSavings`
    (`Infinity` when savings ≤ 0); `offPeakShare = annualOffPeak / annualKwh`.
- VAT 7% and Ft are applied as in the solar tool: `(base + Ft) × 1.07`; service charge ×1.07 too.

### Rendering & charts
- `renderComparison(r)` — writes metric cards, the verdict panel (switch / don't-switch with reason),
  the summary table, and the meter-vs-meter comparison table.
- `buildLoadProfileChart(profile)` — 24h demand split into a **Peak (gold)** region and an
  **Off-Peak (green)** region (two masked datasets, null outside their window).
- `buildCostCompareChart(normalAnnual, touAnnual, meterFee)` — cumulative spend over 10 years; the
  TOU line starts at the meter fee, and the crossover is the break-even point.
- Both charts update in place (mutate `data` + `chart.update()`) after the first build.

### Editable defaults (verify against current MEA/PEA figures)
Progressive blocks `3.2484 / 4.2218 / 4.4217`. Service charges differ by tariff type: normal meter
(type 1.1.2, >150 units) `24.62`/month; TOU meter (type 1.2.2, LV) `38.22`/month. Meter-change fee
dropdown uses the PEA rates effective 23 Feb 2569 (1-phase LV `3,300`, 3-phase LV `3,750`, plus CT
variants; excl. VAT). TOU base rates `5.7982` peak / `2.6369` off-peak. Ft default `0.1623`
(พ.ค.–ส.ค. 2569). All are editable inputs — defaults only.

## License

Creative Commons CC BY-NC-SA 4.0 — non-commercial use only; attribution required.
