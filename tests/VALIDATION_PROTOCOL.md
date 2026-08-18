# CAS Analyzer Validation Protocol

This protocol defines the testing needed to answer three review questions:

1. Are the reported energy savings reliable and defensible?
2. Can tool results be validated with real-world operating data?
3. Does the exported Excel workbook contain enough information to review the calculations for accuracy?

The current executable check is [tests/cas-validation-harness.html](cas-validation-harness.html). Open it in a browser and click **Run Validation Checks**.

## 1. Defensible Energy Savings

### Required Checks

| Check | Purpose | Passing Criteria |
|---|---|---|
| Deterministic compressor fixture | Confirms staging, site correction, isentropic pressure scaling, and part-load power curves produce repeatable values. | Fixture output matches independent expected values within floating point tolerance. |
| 8760 Pre/Post fixture | Confirms annual energy savings are the sum of hourly system kW differences. | Pre kWh, post kWh, and kWh savings match independent fixture calculations within 0.001 kWh. |
| Schedule boundary fixture | Confirms hourly schedules turn loads/compressors on and off at expected days and hours. | Weekday active hour = 1, off hour = 0, weekend = 0 for the fixture. |
| Schedule percentage fixture | Confirms load schedule factors scale hourly demand rather than acting only as on/off gates. | A 62% schedule block produces 62% of duty-adjusted load demand and lower compressor kW than a 100% block. |
| Holiday/off-hour Excel parity fixture | Confirms browser and Excel schedule logic match for holidays and whole-hour interpretation. | Assigned compressor schedule with no holiday rows produces holiday system kW = 0; minute values are ignored consistently. |
| Receiver pass-through fixture | Confirms receiver-only topology between compressor and load does not zero 8760 results. | Compressor -> receiver -> load produces nonzero annual kWh and peak kW matching expected values. |
| Sensitivity review | Shows which assumptions materially affect savings. | Report at least pressure, annual hours, electric rate, leak flow, control type, and load schedule sensitivity. |
| Warning review | Ensures defensibility flags are reviewed before accepting savings. | No unresolved `error`; all `warning` and `review` items either corrected or documented. |

### Minimum Test Fixtures To Maintain

Use the harness fixture as the first baseline:

- Pre: one 100 scfm load/unload compressor, 60 scfm productive load, 10 scfm leak, all hours on.
- Post: same compressor changed to VSD and leak removed.
- Expected result: post kWh must be lower than pre kWh and savings must equal the hourly sum.

Add future fixtures when calculation logic changes:

- Multiple compressors with staging priority and unloaded load/unload units.
- Heatless dryer purge versus refrigerated dryer electric load.
- Timer drain, failed-open float drain, zero-loss drain, and manual drain.
- Pressure reduction case with a binding downstream load.
- Insufficient pressure case that must produce a warning.
- Schedule cases with holiday handling, load percentage scaling, and hour-only start/stop interpretation.
- Project save/load round trip with metadata, schedules, equipment assignments, and post case state.

## 2. Real-World Validation

The tool should be compared against measured data using an IPMVP-style measurement and verification approach. Compressed air systems usually fit Option B when compressor kW and flow are metered, or Option C when whole-facility interval data is the only source.

### Field Data Required

Collect these fields before using a project as a validation case:

| Data | Preferred Source | Minimum Resolution |
|---|---|---|
| Compressor package kW | Power logger, VFD trend, utility submeter | 15-minute interval or better |
| Header pressure | Pressure logger at compressor discharge/header | 15-minute interval or better |
| Flow | Thermal mass flow meter or calibrated flow study | 15-minute interval or representative test periods |
| Compressor status/control mode | Controller trend or observation log | Same period as kW where possible |
| Production schedule | BAS, shift schedule, production log | Hourly or shift block |
| Leak survey results | Ultrasonic survey with estimated scfm | Before and after implementation |
| Dryer/drain status | Inspection, logger, controller trend | Documented operating state |
| Weather/ambient conditions | Site measurement or nearest weather station | Daily or interval if available |

### Calibration Criteria

For a validation case, compare modeled baseline to measured baseline before evaluating savings.

| Metric | Target | Notes |
|---|---|---|
| Annual or period kWh error | <= 5% preferred, <= 10% acceptable with explanation | Use same operating period as model when possible. |
| Peak kW error | <= 10% | Compare like-for-like demand interval. |
| Hourly/interval CVRMSE | <= 15% when interval data is available | Use ASHRAE Guideline 14 style screening. |
| NMBE | within +/-5% when interval data is available | Bias should be investigated even when total kWh is close. |
| Pressure agreement | within +/-3 psi at comparable loads | Larger deviation suggests missing pressure drop or regulator behavior. |
| Flow agreement | within +/-10% or meter uncertainty | Document meter uncertainty and placement. |

When field data fails these criteria, do not present savings as validated. Record the likely cause: missing load, unmodeled compressor control behavior, seasonal ambient variation, incorrect schedule, pressure setpoint mismatch, metering boundary mismatch, or unverified leak estimate.

### Savings Validation Method

1. Model the baseline using pre-implementation metered inputs where available.
2. Calibrate baseline schedules, compressor control type, load demand, and leak estimates until the model meets the calibration criteria.
3. Freeze the calibrated baseline and document all changes from default assumptions.
4. Model post-implementation measures using post data or verified installation records.
5. Compare modeled post kWh to post metered kWh when available.
6. Report savings as validated only when both baseline and post case pass calibration or when the difference is reconciled with documented evidence.

## 3. Excel Workbook Audit Completeness

The workbook is reviewable only if an independent reviewer can trace the calculation from inputs to results without using the web app.

### Required Workbook Contents

| Workbook Area | Required Evidence |
|---|---|
| Project | Project name, customer name, export date, calculation basis, ambient pressure, ambient temperature, QC threshold. |
| System_Diagram | Generated visual of the current system layout embedded as an image. |
| Inputs_Pre / Inputs_Post | Component ID, name, type, schedule ID, flow, pressure, duty cycle, power, control type, unload power, staging priority, operating pressure, rated inlet conditions, pressure drops, dryer/drain fields. |
| Schedules | Schedule ID, day type, start hour, stop hour, factor. |
| Assignments | Case, component ID, type, schedule ID. |
| Calendar_8760 | Hour index, date, day type, hour of day, month, holiday flag. |
| Calc_Pre / Calc_Post | 8,760 formula rows, hourly demand, leak/purge, compressor on state, capacity, load fraction, flow, kW, supply, system kW, web system kW, QC difference, QC status. |
| Results_QC | Annual kWh, peak kW, energy savings, demand reduction, web result, Excel formula result, difference, difference percent, QC status. |
| Warnings_Assumptions | Assumptions and current tool warnings/errors. |

### Workbook Passing Criteria

- Required sheets are present.
- `Calc_Pre` and `Calc_Post` contain header plus 8,760 rows.
- `System_Diagram` is present and contains the generated system image when the project has components.
- Formula workbook contains no macros.
- `Results_QC` compares web results to Excel formula results.
- Hourly QC status flags `ERROR` when formula and web kW differ by more than 1%.
- Annual QC status flags `ERROR` when annual formula and web result differ by more than 1%.
- Any unresolved warning is visible in `Warnings_Assumptions` or in a companion project review note.

## 4. Project File Persistence

Saved project JSON files are review artifacts because they preserve the exact model used to generate outputs.

### Required Project File Contents

| Area | Required Evidence |
|---|---|
| File identity | `schema = measureworks-cas-project`, numeric `version`, `appName`, `fileType`, and `fileVersion` such as `CAS-v1`. |
| Filename | Includes project slug, CAS file indicator/version, and save date, for example `plant_air_study_CAS-v1_2026-08-18.json`. |
| Project metadata | Project name and customer name. |
| Model state | Components, connections, drain attachments, node counter, and analysis state. |
| Settings | Site conditions, electric rate, annual hours, climate zone. |
| 8760 state | Schedules, equipment assignments, post case nodes/connections/assignments, and change groups. |

### Project File Passing Criteria

- Saving produces a JSON file with the CAS project schema and version indicator.
- Loading rejects non-CAS project schemas.
- A save/load round trip restores project/customer metadata, node count, connection count, schedules, equipment assignments, and post case state.
- 8760 cached results are recalculated after load rather than trusted from stale saved state.

## Review Decision Levels

| Level | Meaning | Evidence Required |
|---|---|---|
| Regression-passed | Tool logic has passed deterministic fixtures. | Harness pass screenshot or saved result text. |
| Engineering-defensible | Inputs, assumptions, and warnings have been reviewed by a qualified reviewer. | Completed workbook audit and assumption review. |
| Field-calibrated | Baseline model matches measured baseline within criteria. | Metered data, calibration record, error metrics. |
| Field-validated | Baseline and post results are reconciled with measured data. | Pre/post metered data, validated savings calculation, uncertainty notes. |

Savings should not be described as field-validated unless the final level is met.