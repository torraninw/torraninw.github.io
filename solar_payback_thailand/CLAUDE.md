# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Guidelines

- **Scope:** Household/residential use only — no commercial or industrial scenarios.
- **Language:** UI and all in-app text must remain in Thai. Code comments and this file stay in English. Respond to the user in English unless they explicitly ask otherwise.
- **Design philosophy:** Highly detailed dashboard tailored to the Thai electricity environment (MEA/PEA tariffs, TOU structure, โซลาร์ภาคประชาชน feed-in scheme, personal income tax deduction under พ.ร.ฎ. 805) while keeping the interface approachable and user-friendly — surface complexity progressively, not all at once.
- **Roadmap:**
  - **Phase 1 (done):** Grid-tied system, no battery storage.
  - **Phase 2 (current):** Optional battery storage. Implemented as an off-by-default toggle on the same dashboard (progressive disclosure), not a separate page, so the headline value is a **with/without** comparison. When enabled, surplus solar charges the pack (self-consumption priority) and is discharged to offset grid draw in deficit hours. Battery capex is **excluded** from the RD 805 tax deduction.

## Running the App

No build system or server required. Open `index.html` directly in any modern browser (Chrome, Edge, Firefox, Safari). All dependencies are loaded via CDN at runtime:
- **Chart.js** — interactive charts (loaded with `defer` so a slow CDN can never block HTML parsing)
- **Font Awesome 6.4** — icons
- **Plus Jakarta Sans** — Google Fonts typeface

## Architecture

This is a single-file SPA (`index.html`) with embedded CSS (`<style>`) and JS (`<script>`). There is no build step, no modules, and no external files.

### Data layer (top of `<script>`)

| Variable | Purpose |
|---|---|
| `applianceMasterList` | Static list of 18 appliance types with wattage, per-period max hours, and priority slot arrays for each time period. AC units (9k/12k/18k/24k BTU, Inverter + Non-Inverter) carry `isAc: true`; the EV charger carries `isEv: true`. Computers are split into `pc` (desktop/office, 50W) and `gaming_pc` (200W); `dryer` is modeled as a heat-pump dryer (900W) |
| `presetConfigurations` | 5 household profiles (`default`, `wfh`, `office`, `heavy`, `eco`) — each holds weekday and holiday qty/hours maps. Presets need not list every appliance: `loadPresetState()` calls `normalizeApplianceState()` to fill any missing id with safe defaults (qty 0, hours 0), so newly added appliances (e.g. `gaming_pc`) default to off everywhere |
| `cleanSkyCurve` | Hourly solar generation multipliers (hours 6–17 only; all others produce 0) |
| `AC_THERMAL_FACTOR` | Per-period multipliers applied only to `isAc` appliances, modeling that compressor draw tracks outdoor temperature: `p1` night 0.70, `p2` midday 1.05, `p3` evening 0.85. Users still enter a single nameplate wattage |
| `BATTERY_TYPES` | Phase 2 per-type battery characteristics keyed by `lfp`/`nmc`/`lead`. Each holds `usableFraction` (depth of discharge), `roundTrip` (efficiency), and `cRate` (power as a fraction of nameplate). The user only picks a type + enters **nameplate** capacity; `getBatteryConfig()` derives usable kWh, charge/discharge power, and round-trip from this table so the UI never asks for those technical values |
| `stateWeekdayQty/Hours`, `stateHolidayQty/Hours` | Live mutable state: qty and hours-per-period for each appliance, for weekday and holiday schedules separately |

### Recalculation trigger (manual)

To stay responsive on mobile, recalculation is **not** run on every input event. A floating "คำนวณผลลัพธ์" button (`#calcBtn`, calls `runCalculation()`) drives the heavy pass. Editing actions (appliance +/- via `adjustQty`/`adjustHours`, sliders, typed number inputs) instead call `markResultsStale()`, which flags `resultsAreStale` and pulses the button. Discrete view/load actions — preset buttons, the tariff-type select, and the weekday/holiday chart toggle — call `runCalculation()` immediately. The initial load defers the first pass via `setTimeout(runCalculation, 0)` so layout paints first — **not** `requestAnimationFrame`, because rAF is paused while the tab is hidden/backgrounded and would leave the dashboard blank until interaction.

### Calculation pipeline

1. **`build24HourLoadProfile(qtyMap, hoursMap)`** — maps appliance usage into a 24-element watt array using priority slot arrays (not contiguous blocks), producing a realistic daily load curve. AC watts are scaled by `AC_THERMAL_FACTOR` per period.
2. **`simulateAnnualEnergy(profileWeekday, profileHoliday, tariffType, rates, sys)`** — simulates 365 days × 24 hours and returns annual aggregates (gen, demand, direct self-consumption, battery discharge, export, savings, export revenue). When `sys.battery` is supplied it runs a **stateful** dispatch: a single `soc` (state of charge) carries across hours **and the midnight boundary** — surplus charges the pack first (round-trip loss applied on the way in), remainder exports; deficit discharges the pack to avoid buying grid power at that hour's rate (this is where the TOU peak spread is captured). Pass `battery: null` for the no-battery baseline.
3. **`computeFinancials(agg, fin)`** — turns annual aggregates into payback / ROI / 10-year IRR. Shared by the baseline and battery cases so both are computed identically. `taxEligibleCost` is the RD 805-deductible portion (solar only — battery capex is passed in `grossCost` but **not** `taxEligibleCost`).
4. **`calculateSolarPayback()`** — main orchestrator:
   - Reads all form inputs; applies VAT 7% to tariff rates for self-consumption savings (`(base + Ft) * 1.07`)
   - **Always** computes the no-battery baseline (`baseAgg`/`baseFin`). Calls `getBatteryConfig()`; if it returns a config (battery on), also computes the battery case (`battAgg`/`battFin`) with `grossCost = investment + batteryCfg.cost`. The active case (battery if enabled, else baseline) drives the headline ROI cards and สรุปผล table.
   - Writes results directly to DOM element IDs. The สรุปผล summary and the detailed annual table both show an explicit maintenance line (`tblMaintenance` / `tdMaintenanceCash`) so the displayed total reconciles: bill savings + export revenue − maintenance = net annual benefit (`netAnnualBenefit`). When a battery is modeled, bill savings includes the discharge value and a breakdown sub-row (`#tdBatteryRow`) shows the kWh routed through the battery.
   - `renderBatteryComparison()` populates the `#batteryComparePanel` (hidden unless a battery is modeled): payback / ROI / IRR / net annual benefit / self-sufficiency, baseline vs battery, plus a plain-language verdict.
5. **`calculateIRR(cashFlows)`** — bisection method, 100 iterations, returns `null` if no sign change.
6. **`buildInteractiveLoadCharts(profile, kwp, weatherMultiplier, lossFactor, battery)`** / **`buildInteractivePaybackTrends()`** — create the Chart.js instances on first run, then **update them in place** (mutate `data`, call `chart.update()`) on subsequent recalculations rather than destroying/recreating, which keeps recalculation fast. The load chart always carries 3 datasets (demand, solar, battery discharge); the battery dataset is computed via a single representative-day dispatch and hidden via `dataset.hidden = !battery` when no battery is modeled.

### Thai regulatory constants (hardcoded)

- `TOU_WEEKDAY_DAYS = 242`, `TOU_HOLIDAY_DAYS = 123` (MEA/PEA official schedule)
- Export feed-in tariff: **2.20 THB/kWh** (no VAT on export revenue)
- Tax deduction ceiling: **200,000 THB** per Royal Decree 805 (พ.ร.ฎ. 805)
- Panel degradation: **0.5%/year**
- System losses: **lossFactor = 0.80**

### TOU vs Normal meter mode

`toggleTariffInputs()` shows/hides the TOU rate inputs and the holiday appliance matrix. When Normal (flat-rate) mode is active, `profileHoliday` is set equal to `profileWeekday` and the holiday matrix is hidden.

### Battery storage (Phase 2)

The battery card asks only what an installer's quote actually states — kept deliberately simple for non-expert users. Inputs: `#batteryEnabled` (off by default), `#batteryType` (`lfp`/`nmc`/`lead`), `#batteryCapacity` (**nameplate** kWh, the figure on the quote), `#batteryCost` (THB, capex). Everything technical (usable capacity, charge/discharge power, round-trip efficiency) is **derived in the background** from `BATTERY_TYPES` — there are intentionally no power/efficiency/usable-% inputs.

`getBatteryConfig()` is the single source of truth: returns `null` when off, otherwise `{nameplateKwh, capacityKwh (usable), powerKw, roundTrip, cost, spec}`. The calc, the daily chart, and the assumption note all read from it. `updateBatteryAssumptionNote(cfg)` writes a transparency line (`#batteryAssumptionNote`) spelling out the derived usable kWh / round-trip % / power kW for the chosen type, plus the fixed caveats. `toggleBatteryInputs()` shows/hides the parameter block (called on load and on toggle change). The dispatch itself lives in `simulateAnnualEnergy`, which still receives `{capacityKwh, powerKw, roundTrip}` exactly as before.

Modeling assumptions to keep honest: nameplate × `usableFraction` = usable capacity (the user enters nameplate, not usable), round-trip loss is applied once on charge, and the 10-year horizon assumes battery life ≥ 10 years (**no replacement cost** is modeled within the window). Because the export feed-in rate (2.20 THB/kWh) is well below the avoided grid rate, battery economics improve self-sufficiency but typically lengthen payback at current Thai battery prices — the comparison panel says so explicitly rather than overselling.

### Layout

Two-column `dashboard-grid`. On `DOMContentLoaded`, `moveRoiDashboardToPrimaryPosition()` programmatically prepends the results card to the left column, so the ROI dashboard appears first despite being defined last in the HTML source.

## Known Issue

A Kaspersky browser-extension script tag (`gc.kis.v2.scr.kaspersky-labs.com/...`) can be auto-injected into `<head>` when the file is opened/saved with Kaspersky active. It is **not** harmless: as a synchronous head script on an unreliable host it blocked HTML parsing for ~21s in testing (measured via the Navigation Timing API). It has been removed; if it reappears, delete the `<script src="https://gc.kis...">` tag.

## License

Creative Commons CC BY-NC-SA 4.0 — non-commercial use only; attribution required.
