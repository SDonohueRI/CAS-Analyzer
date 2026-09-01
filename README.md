# CAS·Analyzer — Compressed Air System Analyzer

A browser-based web tool for modeling, analyzing, and optimizing industrial compressed air systems. No installation and no server are required — open the HTML file in any modern browser and start building. Core analysis runs fully in the browser; Excel export uses the ExcelJS browser library.

---

## What It Does

CAS·Analyzer lets you draw a compressed air system as a flow diagram and immediately see the energy, pressure, and flow implications of that system and proposed changes to it. The core questions it answers:

- What is my system's current energy consumption and cost?
- Where is compressed air being wasted (leaks, drains, purge)?
- What pressure does each load actually receive, and is it sufficient?
- How much could I save by lowering supply pressure?
- What is the impact of restaging compressors or adding a VSD?
- How does the system perform across a full year of operation?
- Can the 8760 savings and demand reduction be replicated in a reviewer-friendly Excel workbook?
- Can I save the complete project and reload it later from the static HTML tool?

---

## Getting Started

1. Download or open `index.html`
2. Open it in Chrome, Firefox, Edge, or Safari
3. Drag components from the left palette onto the canvas
4. Connect them by dragging from a green output port to a gray input port
5. Enter equipment data in the right panel
6. Enter **Project Info** such as project name and customer name
7. Click **▶ Analyze** to run the steady-state analysis
8. Click **Detailed Mode** to model Pre/Post cases, schedules, 8760 results, and Excel export

Use **Save Project** to download a versioned JSON project file and **Load Project** to restore it later.

No build step, no npm, no account required.

### Project Files

CAS·Analyzer can save and reload complete projects directly from the browser. The saved file is JSON and includes:

- Project name and customer name
- Components, locations, properties, and connections
- Drain attachments
- Site conditions and settings
- Schedules and equipment assignments
- Pre/Post detailed-mode case data
- Change groups and post-case edits

Saved filenames include a CAS project type/version indicator so they are easy to distinguish from other JSON exports:

```text
plant_air_study_CAS-v1_2026-08-18.json
```

The JSON payload also includes `schema`, `version`, `appName`, `fileType`, and `fileVersion` fields. The loader accepts only the CAS project schema, which reduces the risk of loading the wrong JSON type.

### Hosting

The file is entirely self-contained and deploys anywhere that serves static HTML:

- **GitHub Pages** — push to a repo, enable Pages, done
- **Azure Static Web Apps** — connect to GitHub repo, free tier
- **Any web server** — copy the file to your server root

### External Dependencies

All application code is inside `index.html`. Two third-party libraries are still fetched from public CDNs at runtime:

| Library | Version | Loaded from | Loaded when | If unreachable |
|---|---|---|---|---|
| ExcelJS | 4.4.0 | `cdn.jsdelivr.net` | Page load (`<script>` at end of body) | **Export to Excel** fails — `ExcelJS` is undefined |
| html2canvas | 1.4.1 | `cdnjs.cloudflare.com` | Lazily, on first **Download Report** | Alert: "Could not load screenshot library" |

Everything else — analysis engine, 8760 simulation, monitored-data import, canvas rendering — runs with no network access.

**Fonts do not require the network.** The Google Fonts `@import` was removed. The `IBM Plex Sans` / `IBM Plex Mono` family names are aliased in CSS to locally installed fonts using `@font-face { src: local(...) }`, with `local('IBM Plex ...')` listed first so a real install still wins, then falling back to Segoe UI / Consolas on Windows, SF Pro / SF Mono on macOS, and DejaVu on Linux. Separate faces cover the 100–500 and 600–900 weight ranges so headings are not faux-bolded.

#### Offline fallback

Pinned copies of both libraries are kept in `dependencies/` in case a CDN becomes unreachable, is blocked by a plant network, or the hosted version is pulled:

| File | Size | SRI hash |
|---|---|---|
| `dependencies/exceljs-4.4.0.min.js` | 926 KB | `sha384-Pqp51FUN2/qzfxZxBCtF0stpc9ONI6MYZpVqmo8m20SoaQCzf+arZvACkLkirlPz` |
| `dependencies/html2canvas-1.4.1.min.js` | 194 KB | `sha384-ZZ1pncU3bQe8y31yfZdMFdSpttDoPmOZg2wguVK9almUodir1PghgT0eY7Mrty8H` |

To switch the app to the local copies, change two `src` values in `index.html`:

1. The ExcelJS `<script src="https://cdn.jsdelivr.net/npm/exceljs@4.4.0/dist/exceljs.min.js">` tag near the end of the body → `dependencies/exceljs-4.4.0.min.js`
2. The lazy loader inside `printCalibrationReport()` (`script.src = 'https://cdnjs.cloudflare.com/...'`) → `dependencies/html2canvas-1.4.1.min.js`

Both are classic scripts, so relative paths work over `file://` as well as HTTP. The trade-off is that the app is no longer a single emailable file — `index.html` must travel with the `dependencies/` folder. To keep single-file distribution while going offline, paste the two bundles into inline `<script>` blocks instead (roughly +1.1 MB to `index.html`).

The SRI hashes above can also be added to the CDN tags as `integrity="..." crossorigin="anonymous"` to protect against a compromised or modified CDN payload.

To refresh the backups after a version bump:

```powershell
Invoke-WebRequest -Uri 'https://cdn.jsdelivr.net/npm/exceljs@4.4.0/dist/exceljs.min.js' -OutFile 'dependencies\exceljs-4.4.0.min.js' -UseBasicParsing
Invoke-WebRequest -Uri 'https://cdnjs.cloudflare.com/ajax/libs/html2canvas/1.4.1/html2canvas.min.js' -OutFile 'dependencies\html2canvas-1.4.1.min.js' -UseBasicParsing
```

Because the copies are pinned manually, security patches for these libraries are not picked up automatically — re-check versions periodically.

---

## System Design

### Component Types

| Component | Description | Key Inputs |
|-----------|-------------|------------|
| **Compressor** | Rotary screw or similar positive-displacement unit | Rated capacity (scfm), CAGI rated pressure (psi), operating pressure (psi), package input power (kW), control type, staging priority |
| **Dryer** | Refrigerated, heated desiccant, or heatless desiccant | Flow capacity, rated dP, pressure dewpoint (°F PDP), electrical power or purge fraction |
| **Filter** | Coalescing, particulate, or mist eliminator | Flow capacity, rated dP |
| **Receiver** | Storage vessel | Volume (gal), working pressure |
| **Line Loss** | Pipe runs, fittings, distribution losses | Rated dP, rated flow — or measured fixed dP |
| **Regulator** | Pressure-reducing valve for zone control | Set pressure, flow capacity, upstream margin |
| **Load** | Point-of-use demand | Flow rate (scfm), required pressure (psi), duty cycle (%) |
| **Leak** | Identified leak point | Flow rate (scfm) |
| **Drain** | Condensate drain at low points | Drain type, orifice size, cycle parameters |

### Connecting Components

Drag from the **green output port** (right side of a node) to the **gray input port** (left side) of the next component. Air flows in the direction of the connection. Double-click a connection to delete it.

**Topology rules:**
- Compressors are sources (output port only)
- Loads and leaks are sinks (input port only)
- Drains attach to host nodes (receiver, dryer, filter) via a dashed line — not in-series
- Ring mains are not supported; the system must be a directed acyclic graph

### Compressor Data Entry

Compressor properties come from the **CAGI data sheet** for the specific machine:

| Property | Where to Find It |
|----------|-----------------|
| Rated Capacity (scfm) | CAGI Performance Data Sheet |
| CAGI Rated Pressure (psi) | CAGI Performance Data Sheet — the test discharge pressure |
| Package Input Power (kW) | CAGI Performance Data Sheet |
| Operating Pressure (psi) | Your pressure gauge / controller setpoint |
| Control Type | Equipment nameplate or manufacturer spec |
| Rated Inlet Pressure (psia) | CAGI Data Sheet — typically 14.696 (sea level) |
| Rated Inlet Temperature (°F) | CAGI Data Sheet — typically 68°F |

**CAGI Rated Pressure vs Operating Pressure:** The CAGI data sheet is measured at a specific test pressure (e.g. 125 psi). Your system may actually run at 110 psi. Enter both — the tool corrects power using isentropic scaling between the two conditions.

**Load fraction is computed, not entered.** The tool determines each compressor's operating load from system demand using the staging logic. You cannot type in a load fraction.

---

## Analysis Modes

### Steady State Analysis

Runs instantly when you click **▶ Analyze**. Shows the current operating point:

- **Flow Balance** — supply, productive demand, leaks, drains, dryer purge, and net balance per component
- **Path Pressure Analysis** — delivered pressure at every load with margin vs requirement; flags insufficient pressure
- **Pressure Zone Colour** — toggle connections to show pressure gradient (blue = high, orange = low)
- **Pressure Reduction Opportunity** — minimum achievable header pressure, binding constraint, savings per psi
- **Moisture & Drains** — aftercooler condensate rate, dryer type and PDP, condensation risk, drain air loss cost
- **Efficiency** — productive efficiency %, leak-free score, system SPP

### Detailed Mode and 8760 Annual Simulation

Launched from the **Detailed Mode** button in the header. Detailed Mode keeps the 8760 engine, but gives it a canvas-based Pre/Post workflow.

**Schedules** define how equipment operates during each time block. A schedule is a set of day-type blocks (Monday through Sunday plus Holiday) each with a start hour, stop hour, and factor percentage. Multiple schedules can be defined and assigned independently to each compressor and each load.

Schedule factors are interpreted differently by equipment type:

- **Compressor schedules** are on/off availability. Any block factor greater than 0 means the compressor is available for staging during that hour.
- **Load schedules** scale demand. Hourly demand is `flow rate × duty cycle × schedule factor`, so a 62% block runs a 100 scfm, 100% duty load at 62 scfm.

Schedule calculations use the **hour of day only** and ignore minutes. For example, a `00:30` start is treated as hour `0`; a `06:15` stop is treated as hour `6`. This keeps the browser and Excel formula workbook aligned.

**Pre Case** — the system as currently drawn. Schedule assignments define the baseline operating pattern.

**Post Case** — a deep copy of the Pre case that you can modify freely on its own canvas: change component properties, move equipment, delete/reconnect components, replace compatible equipment types, or reassign schedules. The Post canvas can show the Pre case as a faded non-editable reference layer.

**Replacement workflow** — in Post Case, select a component and use **Change Tools** to apply a compatible replacement. The tool preserves compatible connections and schedule assignments while clearing editable numeric/text inputs so the replacement equipment must be defined. Dropdown-style fields such as compressor control type remain valid defaults.

**Running the simulation:**
1. Define schedules and assign equipment
2. Click **Duplicate Pre→Post** and make proposed changes
3. Click **▶ Run Simulation**
4. Results show annual MWh, peak kW, supply Mscf, leak loss, average pressure, and specific energy
5. Pre/Post delta shows energy and cost savings
6. Charts show load duration curve, monthly energy, average daily profile, and annual flow balance
7. Download 8,760-row CSV for each case or click **Export to Excel** for a formula workbook

**Federal holidays** are computed automatically for the selected year using standard US federal holiday rules.

### Monitored Data Import

Detailed Mode can import measured operating data for model calibration and M&V review. The importer is designed for **CSV-style text exports** from compressor loggers, DDC/BAS trend logs, VFD/controller trends, utility interval meters, submeters, pressure loggers, and flow meters.

Supported upload formats:

- **CSV (`.csv`)** — the primary supported format.
- **Text (`.txt`)** — supported when the file contains comma-separated rows.
- **Multiple files** — supported; each selected source is imported and mapped in sequence.

The file picker currently shows `.xlsx`, but native Excel workbook parsing is not implemented. Convert Excel workbooks to CSV before importing.

Each import should contain a header row and timestamped data rows. Timestamps can be in one combined column or split across date and time columns, with timezone information in the timestamp, a header, a column, or a manual mapping setting. Measurement columns can be mapped to system or equipment channels such as power/kW, amps/current with conversion to kW, pressure, flow, status/runtime, temperature, energy, and ambient conditions. Raw data from high-frequency intervals through hourly readings is resampled to 15-minute intervals for calibration.

**Time zone resolution.** The importer decides the UTC offset in this order: an offset embedded in the timestamp value, then the selected **Time zone source** (header / column / manual), and finally the browser's local zone if nothing is found. The decision is shown at the bottom of the Timestamp Parsing block as **Resolved UTC offset (editable)** together with where it came from. Typing a different offset there (`GMT-06:00`, `-0600`, and `-06:00` are all accepted) overrides everything in the file; **Use detected** reverts. Only fixed offsets are supported, so a dataset spanning a DST transition keeps one offset for the whole period. The resolved offset and its source are stored with the dataset and appear in the Source Manager and the Excel export.

**Pre or Post case mapping.** Each imported source is assigned to the **Pre Case** (baseline) or **Post Case**, either during import or later in the Source Manager. Calibration then runs per case: pick the case next to the **Calibrate** button and only the sources mapped to it are compared against that case's 8760 results. Results are stored separately for each case, so a baseline calibration and a post-retrofit verification can coexist in one project.

**Storage.** Only the 15-minute resampled summary is retained after import. Raw metered rows are discarded once resampling completes and are never written to the project JSON, which keeps saved projects small even when the source file held millions of high-frequency rows.

**Units.** Channels are converted to the engine's target units on import (kW, psig, SCFM, degF, %RH, kWh). Mapped channels are shown as `power (kW)` style labels, the applied conversion is reported as `A -> kW (amps_to_kw_3ph)`, and 15-minute export headers carry their units.

If your data is not already in the right format:

- Export logger, controller, BAS, or utility portal data as CSV when that option is available.
- In Excel, use **Save As CSV** after placing one header row above the data and one timestamp per row.
- If date and time are separate, keep both columns; map them separately during import.
- Remove totals, subtotals, report footers, charts, merged cells, and blank title blocks when possible. Short metadata/title lines before the header are tolerated, but clean tabular data is more reliable.
- Quote headers or values that contain commas, or keep the original logger export if it already quotes those fields.
- Convert cumulative meter reads to interval rows only if needed; the importer can map cumulative energy/volume-style channels, but timestamp order and meter resets should be checked before calibration.
- If only bills or monthly utility totals are available, use them as a reasonableness check outside the import workflow; they are not detailed enough for the 15-minute monitored-data calibration path.

### Excel Formula Workbook Export

Detailed Mode includes **Export to Excel**, which generates a modern Excel workbook intended for review and replication of the 8760 savings calculation.

Workbook sheets include:

- **Project** — export metadata, basis, ambient inputs, QC threshold
- **System_Diagram** — generated canvas-style PNG image of the current system model
- **Inputs_Pre** and **Inputs_Post** — component inputs by case
- **Schedules** — schedule blocks by day type
- **Assignments** — component-to-schedule mapping
- **Calendar_8760** — hourly calendar used by the formulas
- **Calc_Pre** and **Calc_Post** — 8,760 formula rows for hourly demand, staging, compressor kW, dryer kW, and system kW
- **Results_QC** — annual kWh, peak kW, energy savings, demand reduction, and QC status
- **MonitoredData** — present only when monitored data is loaded: source summary (sim case, period, UTC offset and its source, mapped channels with units, source-to-target conversions, completeness), Pre and/or Post calibration metrics, the merged 15-minute resampled data with unit-labelled headers, and an hourly measured-vs-modeled audit table
- **Warnings_Assumptions** — calculation assumptions and current model warnings

The workbook targets Microsoft 365 / modern Excel and uses formulas without macros. QC columns compare the web result to Excel formula results and flag **ERROR** when the difference exceeds 1%.

Project name and customer name are included in the **Project** sheet and reports. When a project name is present, the Excel filename uses it as the base name.

---

## Physics Model

### Compressor Performance

**Site correction (ISO 1217):**
```
corr = (P_site / P_ref) × √(T_ref / T_site)
```
Applied to flow capacity only. Colder, denser air increases volumetric delivery. Power is determined by isentropic work at the operating pressure ratio, not by inlet density.

**Isentropic pressure scaling:**
```
scale = [(P_op/P_in)^0.286 - 1] / [(P_rated/P_in)^0.286 - 1]
actual_power = rated_power × scale × part_load_factor
```
Corrects rated package power for operation at a different discharge pressure than the CAGI test condition. Exponent 0.286 = (k−1)/k for k=1.4 (dry air).

**Part-load power curves by control type:**

| Type | Equation | Notes |
|------|----------|-------|
| Load/Unload | `f × 1.0 + (1−f) × p_ul` | p_ul = unload power fraction (default 20%) |
| Modulating | `0.30 + 0.70 × f` | Throttles inlet; power stays high at part load |
| VSD | `f^1.3` | Speed varies with demand; most efficient at part load |

**Staging logic:**
Compressors are sorted by staging priority (1 = base load). The staging engine fills demand from priority 1 upward at 100% until demand is met, with the last unit running as the trim unit at partial load. L/U units that are online but unneeded run at unload power; VSD/modulating units go to zero. A staging warning fires if any unit's load fraction falls below its minimum stable load threshold.

### Pressure Drop

**Treatment equipment and line losses (quadratic):**
```
dP_actual = dP_rated × (Q_actual / Q_rated)²
```

**Regulator:**
```
dP = P_upstream - P_setpoint  (when upstream > setpoint + margin)
```
Fixed pressure step-down to setpoint; no flow restriction.

### Drain Air Loss

**Orifice equation (subcritical):**
```
Q = 0.65 × π/4 × d² × P_gauge × 0.0685  (scfm)
```

| Drain Type | Air Loss |
|-----------|----------|
| Timer | Q × (t_on / t_cycle) |
| Float — normal | 0 |
| Float — failed open | Full Q continuously |
| Zero-loss electronic | 0 |
| Manual | Q × (opens × duration) / 1440 |

### Moisture Model

Uses Antoine equation for saturation pressure and standard psychrometric relations:
```
W = 0.622 × Pw / (P_total - Pw)
condensate = (W_inlet - W_aftercooler_sat) × 0.075 × 60 × 0.12  gal/hr per scfm
```

Condensation risk is assessed by comparing ambient pipe temperature to pressure dewpoint downstream of the dryer:
- **LOW**: ambient > PDP + 15°F
- **MEDIUM**: ambient > PDP
- **HIGH**: ambient ≤ PDP (condensation likely in distribution)

---

## Site Conditions

Set in the **System Design** tab. These affect all compressor corrections and the moisture model.

| Setting | Effect |
|---------|--------|
| Climate Zone (ASHRAE 169) | Sets ambient temperature and humidity defaults |
| Ambient Pressure (psia) | Altitude correction — sea level = 14.696 psia |
| Ambient Temperature (°F) | Inlet density correction for flow capacity |
| Relative Humidity (%) | Moisture load calculation |
| Aftercooler Outlet Temp (°F) | Condensate at aftercooler |

**Climate zones available:** 1 (Very Hot Humid) through 8 (Subarctic). Selecting a zone sets temperature and humidity automatically; individual values can be overridden.

---

## Reports

Generated from the **Scenarios** tab.

**Management Summary** — one-page executive report with three headline numbers (annual cost, waste cost, identified savings) and top recommendations ranked by payback period.

**Engineering Report** — full technical detail including system inventory with CAGI data, flow balance table, path pressure analysis, and energy summary broken out by compressor and dryer type. Open in the browser and use Print → Save as PDF.

---

## Guide & Methods Tabs

**Guide** — step-by-step walkthrough of the tool workflow. Each step that involves a calculation links to the relevant Methods section.

**Methods** — 11 documented calculation sections (M1–M11) covering every engine function with equations in standard notation, variable definitions, assumptions, and references to source standards (ISO 1217, ASHRAE Fundamentals, Compressed Air Challenge best practices). Links open an inline modal without leaving the Guide.

---

## Calculation References

| Method | Standard / Reference |
|--------|---------------------|
| Site correction | ISO 1217:2009 Displacement Compressors — Acceptance Tests |
| Isentropic scaling | ISO 1217:2009; Compressed Air Challenge Fundamentals |
| Part-load curves | CAGI Data Sheet methodology; CAC Advanced Management |
| Staging logic | CAC Best Practices, Chapter 7 |
| Path pressure drop | ISO 6358; CAC Best Practices, Chapter 5 |
| Psychrometrics | ASHRAE Fundamentals Handbook (2021), Chapter 1 |
| Drain air loss | CAC Best Practices, Chapter 6; Parker Hannifin Engineering Handbook |
| Climate zones | ASHRAE 169-2020 |

---

## Technical Notes

- **Single file app shell** — all application HTML, CSS, and JavaScript are in `index.html`.
- **No server required** — all computation runs in the browser. No data is transmitted anywhere.
- **Excel export** — formula workbook generation uses ExcelJS from a browser CDN in the current static build. Pinned local copies are kept in `dependencies/`; see [External Dependencies](#external-dependencies) for the offline fallback procedure.
- **Fonts** — no webfont request. `IBM Plex Sans`/`IBM Plex Mono` are aliased to locally installed fonts via `@font-face { src: local(...) }`.
- **Canvas rendering** — system diagram uses HTML5 Canvas 2D for connections (with DPR scaling) and HTML div elements for nodes.
- **Topology engine** — Kahn's algorithm for topological sort, depth-first search for cycle detection, reverse-pass attributed demand for flow splitting at merge nodes.
- **8760 engine** — 8,760 hourly iterations with demand-driven staging each hour. Typical runtime < 2 seconds in modern browsers.
- **Excel formula engine** — exported workbooks use project-specific fixed columns for visible, reviewable formulas and 1% QC checks against web-calculated results.
- **Browser support** — Chrome 90+, Firefox 88+, Edge 90+, Safari 14+

---

## Validation and Testing

Testing materials live in `tests/`:

- `tests/cas-validation-harness.html` — self-contained browser checks for compressor staging, 8760 Pre/Post savings, schedule boundaries, and Excel workbook audit structure.
- `tests/VALIDATION_PROTOCOL.md` — review protocol for defensible savings, field validation with measured data, and Excel workbook calculation audit requirements.

To run the executable checks, open `tests/cas-validation-harness.html` in a browser and click **Run Validation Checks**. The harness loads the local `index.html` and reports pass/fail results.

---

## Limitations

- Ring mains (looped piping) are not supported — system topology must be a directed acyclic graph
- Compressor part-load curves are empirical approximations; for highest accuracy use CAGI full part-load data sheets
- Moisture model uses simplified psychrometrics adequate for condensate sizing; not a substitute for detailed dryer selection calculations
- Pressure drop calculations use rated dP scaled quadratically; does not perform Darcy-Weisbach pipe sizing from diameter and length
- The 8760 simulation uses a single set of ambient conditions year-round; seasonal variation in ambient temperature and humidity is not modelled

---

## File Structure

```
index.html                          Single self-contained app (HTML + CSS + JS)
README.md                           This document
dependencies/                       Pinned offline backups of the CDN libraries
  exceljs-4.4.0.min.js
  html2canvas-1.4.1.min.js
tests/
  cas-validation-harness.html       Executable browser checks
  VALIDATION_PROTOCOL.md            Review protocol
```

---

## License

Built for internal team use and sharing. No warranty expressed or implied. Engineering calculations should be verified by a qualified compressed air systems engineer before use in equipment procurement or facility decisions.
