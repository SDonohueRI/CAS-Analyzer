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

---

## Getting Started

1. Download or open `index.html`
2. Open it in Chrome, Firefox, Edge, or Safari
3. Drag components from the left palette onto the canvas
4. Connect them by dragging from a green output port to a gray input port
5. Enter equipment data in the right panel
6. Click **▶ Analyze** to run the steady-state analysis
7. Click **Detailed Mode** to model Pre/Post cases, schedules, 8760 results, and Excel export

No build step, no npm, no account required.

### Hosting

The file is entirely self-contained and deploys anywhere that serves static HTML:

- **GitHub Pages** — push to a repo, enable Pages, done
- **Azure Static Web Apps** — connect to GitHub repo, free tier
- **Any web server** — copy the file to your server root

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

**Schedules** define which equipment is on or off during each time block. A schedule is a set of day-type blocks (Monday through Sunday plus Holiday) each with a start time, stop time, and on/off state. Multiple schedules can be defined and assigned independently to each compressor and each load.

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

### Excel Formula Workbook Export

Detailed Mode includes **Export to Excel**, which generates a modern Excel workbook intended for review and replication of the 8760 savings calculation.

Workbook sheets include:

- **Project** — export metadata, basis, ambient inputs, QC threshold
- **Inputs_Pre** and **Inputs_Post** — component inputs by case
- **Schedules** — schedule blocks by day type
- **Assignments** — component-to-schedule mapping
- **Calendar_8760** — hourly calendar used by the formulas
- **Calc_Pre** and **Calc_Post** — 8,760 formula rows for hourly demand, staging, compressor kW, dryer kW, and system kW
- **Results_QC** — annual kWh, peak kW, energy savings, demand reduction, and QC status
- **Warnings_Assumptions** — calculation assumptions and current model warnings

The workbook targets Microsoft 365 / modern Excel and uses formulas without macros. QC columns compare the web result to Excel formula results and flag **ERROR** when the difference exceeds 1%.

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
- **Excel export** — formula workbook generation uses ExcelJS from a browser CDN in the current static build. For fully offline distribution, vendor or inline the ExcelJS bundle.
- **Canvas rendering** — system diagram uses HTML5 Canvas 2D for connections (with DPR scaling) and HTML div elements for nodes.
- **Topology engine** — Kahn's algorithm for topological sort, depth-first search for cycle detection, reverse-pass attributed demand for flow splitting at merge nodes.
- **8760 engine** — 8,760 hourly iterations with demand-driven staging each hour. Typical runtime < 2 seconds in modern browsers.
- **Excel formula engine** — exported workbooks use project-specific fixed columns for visible, reviewable formulas and 1% QC checks against web-calculated results.
- **Browser support** — Chrome 90+, Firefox 88+, Edge 90+, Safari 14+

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
compressed-air-analyzer.html    Single self-contained file
README.md                       This document
```

---

## License

Built for internal team use and sharing. No warranty expressed or implied. Engineering calculations should be verified by a qualified compressed air systems engineer before use in equipment procurement or facility decisions.
