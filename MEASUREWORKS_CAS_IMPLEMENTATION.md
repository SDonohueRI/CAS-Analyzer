# MeasureWorks Platform Application Plan for CAS Analyzer

**Repository:** `SDonohueRI/CAS-Analyzer`  
**Date:** 2026-08-18  
**Status:** Draft implementation plan  
**Source framework:** `MeasureWorks-Lighting-Beta/MEASUREWORKS.md` and related expansion specs

---

## 1. Purpose

This document translates the MeasureWorks platform ideas developed in the MeasureWorks Lighting repo into an implementation plan for CAS Analyzer.

CAS already has a strong compressed-air engineering model: diagram-based system layout, steady-state analysis, 8760 simulation, pressure-path analysis, waste streams, scenarios, and management/engineering reports. The next step is not to replace that tool. The goal is to make CAS behave like a MeasureWorks module: profile-driven, traceable, export-ready, consistent with future modules, and easier to govern across utility programs and engineering workflows.

---

## 2. MeasureWorks theories to carry into CAS

### 2.1 One platform, many measure modules

MeasureWorks should feel like a family of energy-savings tools rather than unrelated calculators. Lighting, CAS, HVAC, refrigeration, and future tools should share familiar project structure, assumptions handling, exports, warnings, and governance.

For CAS, this means keeping the compressed-air-specific diagram and physics, while aligning the surrounding workflow with the platform conventions:

- Project setup
- Inputs
- Assumptions
- Calculations
- 8760
- Results
- Reports/exports

### 2.2 Two workflows, one engine

The Lighting framework defines two user workflows: vendor/rebate intake and audit mode. CAS should use the same idea, adapted to compressed air.

| MeasureWorks concept | CAS expression |
|---|---|
| Vendor/rebate intake | Program screening mode for contractors, account managers, and incentive reviewers |
| Audit mode | Engineering mode for compressed-air specialists modeling real systems |
| Locked defaults | Program profiles, best-practice assumptions, and required citations |
| Flexible engineering | Metered inputs, overrides, custom schedules, scenarios, and notes |
| Same calculation engine | The same CAS engine computes both modes, with profile rules controlling defaults and warnings |

The tool should flag concerns rather than block engineers. Bad or incomplete inputs should produce warnings, review flags, or missing-source indicators, not dead ends.

### 2.3 Static deployment remains a feature

Both Lighting and CAS are designed around static HTML/CSS/JS. That should remain a core platform decision.

CAS should continue to support:

- GitHub Pages, intranet, shared-drive, or local-file deployment
- Browser-only calculations
- No server dependency
- No account requirement
- No automatic data transmission

The implementation can still move toward modular source files for maintainability, as long as distribution can produce a self-contained build when needed.

### 2.4 Program profiles, not client-specific code

Lighting treats client/program differences as swappable profile data. CAS should adopt the same pattern.

Profiles should control assumptions such as:

- Electric rates and demand rates
- Default annual hours or schedule libraries
- Incentive formulas and caps
- Eligible measure categories
- Required report labels and disclaimers
- Warning thresholds
- Default compressor control assumptions
- Leak repair cost defaults
- Zero-loss drain cost defaults
- Pressure-reduction savings limits
- Required citations and review flags

Client-specific rules should enter as profile data whenever possible. CAS engine logic should remain generic compressed-air logic.

### 2.5 The 8760 is the fundamental calculation object

Lighting treats 8760 as the fundamental object, with annual mode as a summary or simplified path. CAS already has an 8760 simulation, so this platform theory maps naturally.

CAS should eventually make all annual results traceable to one of two clearly labeled bases:

- `annual`: simplified flat-hours annual calculation
- `8760`: hour-by-hour simulation with schedules and operating states

Reports should identify which basis was used. Where both are available, annual calculations should act as a reasonableness check against 8760 results.

### 2.6 Pure engine, profile data, presentation layer

Lighting defines three layers:

1. Core engine: pure calculations, no DOM
2. Program profiles: data only
3. Presentation: UI, exports, charts, reports

CAS is currently a single-file application. That is acceptable for distribution, but source development should move toward the same separation.

Recommended CAS source structure:

```text
index.html                    app shell for source build or distribution preview
js/cas-engine.js              pure compressed-air calculation functions
js/cas-state.js               project, components, scenarios, schedules, serialization
js/cas-profiles.js            program/client profiles and schema
js/cas-reference-data.js      component defaults, measure templates, citations
js/cas-ui.js                  diagram, panels, event handling
js/cas-reports.js             report rendering
js/cas-export-excel.js        workbook export with formulas and citations
js/cas-import.js              future paste/import support
```

A build step can inline these into a single static HTML file for easy distribution.

### 2.7 Provenance flags on important inputs

Lighting uses `DEFAULT / OVERRIDE / METERED` flags. CAS should adopt these because compressed-air analysis often mixes manufacturer data, field readings, assumptions, and engineering estimates.

Recommended CAS provenance flags:

| Flag | Meaning | Example |
|---|---|---|
| `DEFAULT` | Value supplied by profile, component default, or best-practice assumption | Default unload power fraction |
| `OVERRIDE` | User changed an assumed value without direct measurement | Estimated leak repair cost |
| `METERED` | Field measured or taken from verified instrumentation | Header pressure, flow study data, logged kW |
| `DATASHEET` | Taken from manufacturer/CAGI data | Rated capacity, rated pressure, package input kW |
| `CALCULATED` | Derived by the engine and not user-editable | Load fraction, SPP, annual MWh |

Each provenance flag should support a source/note field where practical.

### 2.8 Citations and assumptions are first-class outputs

CAS already documents physics methods in README and the Methods tab. MeasureWorks adds a stronger requirement: every profile value and important assumption should carry a source string.

CAS should report:

- Profile id/version/schema
- Engine version
- Calculation basis: annual or 8760
- Source for CAGI data when entered
- Source for metered pressure/flow/power values
- Source for incentive assumptions
- Source for default costs and best-practice thresholds
- List of undocumented overrides

### 2.9 Warnings are non-blocking

CAS already has warnings for leaks, oversized capacity, topology issues, staging concerns, pressure deficiencies, and unaccounted demand. The MeasureWorks theory makes this a governance rule: warnings inform review but do not prevent calculation unless the system cannot be solved.

CAS should classify warnings:

- `error`: calculation cannot proceed, such as cyclic topology
- `warning`: result may be invalid or needs review
- `review`: override, missing source, unusual assumption, program eligibility issue
- `info`: explanatory note or best-practice suggestion

### 2.10 Exports must be reviewer-friendly

Lighting prioritizes Excel outputs with live formulas and a canonical layout. CAS currently generates browser reports. The MeasureWorks version should add exports that reviewers can trace.

CAS export layout should follow:

1. Cover/Project
2. Inputs
3. Assumptions
4. Calculations
5. 8760, when applicable
6. Results
7. Warnings/Review Flags

Excel formulas should reference input cells where practical. Hourly 8760 rows may be exported as values, with their basis cited in Assumptions.

### 2.11 Version stamping and change governance

Profiles should be versioned data packages. User edits to profile assumptions should require a change note and should not silently mutate shipped defaults.

CAS outputs should stamp:

- Module: CAS Analyzer
- MeasureWorks profile id
- Profile version
- Profile schema version
- CAS engine version
- Calculation basis
- Export/report generated date

Profile changes should flow through repo commits or PRs so assumption changes have an audit trail.

---

## 3. CAS product model under MeasureWorks

### 3.1 Project object

CAS should model one project with explicit metadata.

```js
project = {
  id,
  name,
  customer,
  facility,
  profileId,
  calcBasis: 'annual' | '8760',
  createdAt,
  updatedAt,
  notes,
  defaults,
  components,
  connections,
  drainAttachments,
  schedules,
  scenarios,
  assumptions,
  warnings
}
```

### 3.2 Component object

Existing CAS nodes should gain metadata without changing their core physics identity.

```js
component = {
  id,
  type,
  x,
  y,
  props,
  provenance,
  sources,
  notes,
  tags
}
```

`props` remains the domain-specific input bag. `provenance` and `sources` add traceability around those properties.

### 3.3 Scenario object

CAS already has Pre/Post simulation behavior. Under MeasureWorks, scenarios should become explicit objects.

```js
scenario = {
  id,
  name,
  caseType: 'baseline' | 'proposed' | 'alternate',
  componentOverrides,
  removedComponents,
  scheduleAssignments,
  measureActions,
  results,
  warnings
}
```

This creates a foundation for program-ready measure packages: leak repair, pressure reduction, drain replacement, compressor controls, VSD, storage, sequencing, and shutdown schedules.

### 3.4 Measure action object

Recommendations should evolve from generated text into structured measure actions.

```js
measureAction = {
  id,
  type,
  title,
  affectedComponents,
  baselineAssumption,
  proposedAssumption,
  savingsBasis,
  costBasis,
  incentiveBasis,
  sources,
  warnings,
  results
}
```

This lets reports and exports explain not only what the savings are, but what changed.

---

## 4. CAS profile schema direction

Profiles should begin as simple JSON-compatible objects and expand over time.

```js
profile = {
  schemaVersion: 1,
  id: 'default_cas',
  name: 'Default CAS Engineering Profile',
  version: '2026.1',
  mode: 'annual',
  rates: {
    electricRate: 0.10,
    demandRate: null
  },
  schedules: {
    defaultHoursPerYear: 6000,
    defaultScheduleId: 'business_extended'
  },
  incentives: {
    enabled: false,
    rules: []
  },
  assumptions: {
    leakBestPracticePct: 10,
    pressureSavingsPctPerPsi: 0.005,
    defaultLeakRepairCost: 500,
    defaultZeroLossDrainCost: 2000,
    defaultUnloadPowerFraction: 0.20
  },
  warningThresholds: {
    leakPctWarn: 10,
    oversizedCapacityPct: 50,
    minimumPressureMarginPsi: 0
  },
  sources: {
    leakThreshold: 'Compressed Air Challenge best practices',
    siteCorrection: 'ISO 1217',
    moisture: 'ASHRAE Fundamentals'
  },
  reportLabels: {
    savingsLabel: 'Estimated savings pending program review'
  },
  changeLog: []
}
```

---

## 5. Implementation phases

### Phase 0 - Preserve current CAS behavior

Goal: Establish a stable baseline before platform refactoring.

Tasks:

- Keep current `index.html` working as the distribution artifact.
- Save a known example system and expected headline outputs.
- Record current steady-state and 8760 behavior as regression checks.
- Avoid changing equations while introducing platform structure.

Acceptance checks:

- Current app opens from `index.html`.
- Load Example still works.
- Analyze still produces flow, pressure, energy, and report outputs.

### Phase 1 - Add CAS MeasureWorks design record

Goal: Make CAS platform rules explicit.

Tasks:

- Keep this document in the repo root.
- Add a shorter `CAS_MEASUREWORKS.md` or README section after decisions are finalized.
- Define CAS engine/profile/export version constants.
- Document which current assumptions are defaults versus standards versus user inputs.

Acceptance checks:

- New contributors can tell which behavior is CAS-specific and which is MeasureWorks platform behavior.
- Current limitations and profile-driven roadmap are visible.

### Phase 2 - Introduce project and profile state inside the single file

Goal: Add platform concepts with minimal disruption.

Tasks:

- Add a `project` object around existing `nodes`, `connections`, `drainAttachments`, schedules, and scenarios.
- Add a default CAS profile object.
- Replace hardcoded user-facing assumptions where practical with profile references.
- Add profile/version/calculation-basis stamps to reports.
- Add a basic assumptions summary panel or report section.

Acceptance checks:

- Existing UI still works.
- Reports show profile id/version and engine version.
- Electric rate, hours, warning thresholds, and cost defaults can be traced to project/profile/user input.

### Phase 3 - Add provenance and source tracking

Goal: Make inputs reviewable.

Tasks:

- Add provenance metadata for component properties.
- Default new component values to `DEFAULT`.
- Mark CAGI compressor fields as `DATASHEET` when users confirm source.
- Mark derived outputs as `CALCULATED`.
- Add source/note fields for important overrides.
- Add warnings for undocumented overrides.

Acceptance checks:

- At least compressor, leak, drain, line-loss, and schedule assumptions show provenance.
- Reports list undocumented overrides.
- Provenance does not prevent editing.

### Phase 4 - Convert recommendations into structured measure actions

Goal: Move from text recommendations to reusable measure definitions.

Tasks:

- Define measure action types for:
  - leak repair
  - zero-loss drain upgrade
  - pressure reduction
  - compressor restaging
  - VSD/trim compressor retrofit
  - night/weekend shutdown
  - dryer purge reduction
- Store savings basis, cost basis, affected components, and warnings per action.
- Generate management recommendations from measure actions.
- Let scenarios apply one or more measure actions to a proposed case.

Acceptance checks:

- Existing recommendation cards still appear.
- Each recommendation has structured assumptions behind it.
- Pre/Post reports can identify which actions created the savings.

### Phase 5 - Align reports and exports with MeasureWorks layout

Goal: Make outputs consistent across modules.

Tasks:

- Rework report sections into the canonical order:
  - Cover/Project
  - Inputs
  - Assumptions
  - Calculations
  - 8760, when applicable
  - Results
  - Warnings/Review Flags
- Add Excel export using ExcelJS if the repo moves to modular source/build tooling.
- Keep browser print reports for static/offline use.
- Label incentives and savings as estimates pending program review where profiles require it.

Acceptance checks:

- Reports are traceable from result back to inputs and assumptions.
- Profile/version stamps appear in every report/export.
- Review flags are visible, not hidden in console or UI-only state.

### Phase 6 - Modularize source while preserving single-file distribution

Goal: Make CAS maintainable as a MeasureWorks module.

Tasks:

- Extract pure calculation functions into `js/cas-engine.js`.
- Extract state factories and serializers into `js/cas-state.js`.
- Extract profile schema/data into `js/cas-profiles.js`.
- Extract report/export logic into dedicated modules.
- Add a build script that generates the self-contained `index.html` distribution file.

Acceptance checks:

- Engine functions can be tested without DOM.
- Profile data is editable without touching engine logic.
- Static distribution still works from one HTML file.

### Phase 7 - Strengthen 8760 as the core basis

Goal: Bring CAS closer to the MeasureWorks calculation philosophy.

Tasks:

- Store schedules in project data with clear provenance.
- Allow annual results to cite whether they came from flat annual mode or 8760 aggregation.
- Add seasonal ambient conditions when profile or project data supports it.
- Export 8760 hourly rows with timestamps, compressor states, load, pressure, power, cost, and warnings.
- Use annual mode as a reasonableness check when 8760 is active.

Acceptance checks:

- Results clearly identify annual versus 8760 basis.
- 8760 output can be reconstructed from exported project data.
- Schedule assumptions appear in reports.

### Phase 8 - Add program-governed profile editing

Goal: Support utility/client profile maintenance without code changes.

Tasks:

- Add a schema-driven profile settings editor patterned after Lighting.
- Require change notes for edited profile assumptions.
- Auto-mark edited profiles as local drafts.
- Provide export JSON for profile commits.
- Provide revert-to-shipped behavior.

Acceptance checks:

- Most assumption updates require no engine changes.
- Edited profiles cannot silently masquerade as shipped profiles.
- Profile JSON diff becomes the assumption audit trail.

---

## 6. Near-term CAS backlog

Recommended first implementation slice:

1. Add engine/profile/version constants to `index.html`.
2. Add a default CAS profile object.
3. Route electric rate, annual hours, leak threshold, default repair cost, drain upgrade cost, and pressure-savings defaults through the profile.
4. Add profile/version/calculation-basis text to the management and engineering reports.
5. Add an Assumptions section to both reports.
6. Add a review flag when a recommendation uses profile default costs instead of user-entered costs.

This gives CAS immediate MeasureWorks behavior without requiring a full modular refactor.

---

## 7. Open decisions

These should be resolved before deeper implementation:

1. Should CAS remain distributed as `index.html`, or should source become modular with a generated single-file build?
2. Which utility/program profiles should CAS support first?
3. What incentive structures matter for compressed-air measures?
4. Which inputs must distinguish `DATASHEET`, `METERED`, and `OVERRIDE` in Phase 1?
5. Should CAS add project/customer metadata before Excel export work?
6. Should 8760 become the default calculation basis for program profiles, or remain opt-in for engineering audits?
7. Should profile editing live in CAS immediately, or should profiles initially be edited as data files only?

---

## 8. Success definition

CAS will feel like a MeasureWorks module when:

- Users recognize the same project, assumptions, results, warnings, and export patterns from Lighting.
- Program assumptions live in profiles rather than scattered UI code.
- Engineering inputs are flexible and traceable.
- Reviewers can see sources, overrides, profile versions, and calculation basis.
- The compressed-air physics remain domain-specific and defensible.
- Static/offline deployment still works.
- The same project data can drive UI, reports, exports, and future program integrations.
