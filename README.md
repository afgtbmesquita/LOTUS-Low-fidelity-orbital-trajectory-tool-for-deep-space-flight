# LOTUS - Closed-source student distribution (standalone executable)

LOTUS is a low-fidelity orbital trajectory tool for deep-space flight. This
distribution ships as a single standalone executable produced with MATLAB
Compiler. The entire `+lotus` pipeline is packed inside the executable's
encrypted CTF archive - no MATLAB source, no P-code, no package folders are
exposed.

Students interact with the program through exactly one editable file:
`user_input_lotus.json`.

## Folder layout

```
compiledsoftware_executable/dist/
    lotus                         (or lotus.exe on Windows)
    user_input_lotus.json         <- the only file you edit
    README.md                     (this file)
    data/
        spice/
            mice/                 <- MICE (MATLAB interface to SPICE)
            generic_kernels/      <- NAIF SPICE kernels (TLS, TPC, BSP, ...)
    outputs/
        trajectory_analyses/      (created on first save)
```

`lotus` must be run from a working directory that contains
`user_input_lotus.json` and a `data/spice/` tree; outputs are written into
`outputs/trajectory_analyses/` next to the executable.

## Requirements

- **MATLAB Runtime** matching the release the executable was built with
  (see the version number the maintainer communicates with the build).
  Download from [mathworks.com/products/compiler/matlab-runtime.html](https://www.mathworks.com/products/compiler/matlab-runtime.html)
  and install the matching version. No MATLAB license is required for the
  runtime.
- **A display / AWT available**. The interactive 3D viewer relies on Java
  AWT; it is present on Windows and most desktop Linux installations by
  default. Running over a pure headless SSH session will skip the viewer.
- Parallel Computing Toolbox support is optional; LOTUS runs
  single-threaded when it is absent.

## One-time data setup

The package ships without ephemeris data. Copy the two subfolders below
from your instructor's master distribution into `dist/data/spice/`:

```
dist/
    data/
        spice/
            mice/                 <- MICE (MATLAB interface to SPICE)
            generic_kernels/      <- NAIF SPICE kernels (TLS, TPC, BSP, ...)
```

These paths match what `user_input_lotus.json` expects out of the box.

## How to run

1. Install the MATLAB Runtime.
2. Open a terminal and `cd` into the `dist/` directory.
3. Run the executable:
   - Windows: `lotus.exe`
   - macOS / Linux: `./lotus`
4. Choose option 1 for a new analysis, or option 2 to visualize a stored
   `.mat` under `outputs/trajectory_analyses/`.

To load a non-default JSON file, pass its path as the first argument:

```
lotus path\to\other_mission.json
```

## Editing a mission

Open `user_input_lotus.json` in any text editor. The file is the single
source of truth for mission identity, encounter slots, time-of-flight
constraints, transfer strategy, auto-flyby policy, run options, and
plotting. Each key is documented in the field reference below.

### JSON conventions

Because JSON has no MATLAB expressions or comments, the following
conventions are used throughout the file. The loader reconstructs the
MATLAB values on the fly.

- **Booleans**: `true` / `false` in JSON (native).
- **Infinity**: represented as JSON `null`. The loader converts `null` to
  `Inf` for every field that accepts it (see the field reference below).
- **MATLAB ranges** like `0:1:1` become nested arrays `[[0, 1]]`. Each
  outer entry is one leg; the inner array is the list of revolution
  counts (or angles) for that leg. So `[[0, 1], [0, 1, 2]]` means leg 1
  allows 0 or 1 revs and leg 2 allows 0, 1, or 2 revs.
- **Pre-computed numeric expressions**. MATLAB expressions like
  `15 * 365` must be pre-computed (write `5475` directly). `deg2rad(0.2)`
  must be written as its numeric value (`0.003490658503988659`) if you
  ever need to override such a run option.
- **No comments**. The JSON file has no comments of any kind. This
  README is the documentation.

### Field reference

Each section below mirrors the JSON structure top-down.

---

#### `name` (string)

Human-readable label used in logs, summaries, and exported metadata.
Update it whenever you start a new campaign so exported results stay
distinguishable.

Example: `"name": "LOTUS-Mission"`

---

#### `ephem` (object)

SPICE kernel and MICE locations. Paths may be absolute or relative;
relative paths are resolved against the working directory where the
executable is launched (so `data/spice/...` just works out of the box).

- `ephem.kernelRoot` - folder containing NAIF SPICE kernels (TLS, TPC,
  BSP files). Default: `"data/spice/generic_kernels"`.
- `ephem.micePath` - MICE distribution root. Default:
  `"data/spice/mice"`.

---

#### `slots` (array of objects)

Ordered list of planetary encounters. The first slot is the departure
body; the last is the arrival body. Intermediate slots (if any) are
**fixed** flybys. Auto-inserted flybys are configured in `autoFlyby`.

Per slot:

- `slotIndex` (integer) - 1-based position of the encounter.
- `bodies` (array of strings) - candidate SPICE body names for this slot
  (e.g. `["EARTH"]` or `["EARTH", "MARS"]`).
- `windowUtc` (array of two strings) - search window as
  `[startUTC, endUTC]` in SPICE UTC format (e.g. `"2025 JAN 01"`).
- `stepDays` (number) - time-sampling step inside the window, in days.

---

#### `tofConstraints` (array of objects)

Pairwise time-of-flight bounds in days between slots. `from` and `to`
reference `slotIndex` values from `slots`. Multiple entries can constrain
non-adjacent pairs.

Per entry:

- `from` (integer), `to` (integer) - slot indices.
- `minDays` (number) - minimum TOF.
- `maxDays` (number or `null`) - maximum TOF; `null` means unbounded
  (`Inf`).

Example: `{"from": 1, "to": 2, "minDays": 100, "maxDays": 5475}`
(5475 = 15 * 365 days).

---

#### `constraints` (object)

Global trajectory filters applied regardless of which transfer solver
was used. These act as hard limits during pruning.

- `constraints.flybyMinAltitude` (number) - minimum flyby altitude [km].
- `constraints.maxFlybyDv` (number or `null`) - max DV per flyby [km/s].
  `null` = `Inf`. Only enforced when `enablePoweredFlyby` is `true`.
- `constraints.maxTotalDv` (number or `null`) - max total mission DV
  [km/s]. `null` = `Inf`.

---

#### `vinfBounds` (array of arrays of two numbers)

Per-slot v-infinity bounds as a 2D matrix. Row `i` is
`[vInfMin_i, vInfMax_i]` for slot `i`.

Example:
```json
"vinfBounds": [
    [0.0, 10.0],
    [0.0, 10.0]
]
```

---

#### `wesc`, `wins` (integers)

Landau 2022 binary weights. Control whether the endpoint DV at
departure / arrival is counted in the total cost. Values `0` (exclude)
or `1` (include). Requires paper-mode or an update in the leg cost model.

---

#### Per-leg transfer strategy

- `longWay` (`"both"` | `true` | `false`) - long-way branch policy.
  `"both"` explores both short- and long-way Lambert arcs.
- `transferType` (array of strings) - one entry per leg. Valid values:
  `"nonresonant"`, `"resonant"`, `"powered"`, `"rendezvous"`,
  `"centralbodyswitch"`, `"auto"`, `"lowthrust"`, `"null"`.
- `Nrev` (array of arrays of integers) - per-leg revolution counts
  (multi-rev). `[[0, 1]]` means one leg that allows 0 or 1 full
  heliocentric revolutions. In flyby-final-lowthrust mode `Nrev`
  applies to the final lowthrust leg; inserted ballistic legs use `[0]`.
- `branchSet` (integer) - global Lambert branch policy.

---

#### Resonant controls

- `dvInf` (number) - resonant v-infinity sampling step [km/s].
- `phi` (array of arrays of numbers) - resonant departure-angle samples
  [rad] per leg. Example: `[[0]]` = one leg, one angle (0 rad).

---

#### Powered controls

Parameters used only when a leg is resolved as a powered transfer
(deep-space maneuver placed on the arc). Setting `dvLeg` to 0 collapses
the powered leg back to the ballistic Lambert solution.

These control the **DSM grid on transfer arcs**. They do **not** enable
powered flybys at encounters - see `enablePoweredFlyby` below.

- `dvLeg` (number) - max sampled DSM magnitude on a powered leg [km/s].
- `dDvLeg` (number) - DSM magnitude sampling step on a powered leg
  [km/s].

---

#### `lowThrust` (object)

Parameters used only for low-thrust rapid-shaping legs.

- `lowThrust.p1Min` (number) - minimum p(0.5) bound for feasibility
  [km].
- `lowThrust.nSamples` (integer) - trapezoid samples over s in [0,1]
  (solver/closure).
- `lowThrust.plotSamples` (integer) - samples used when plotting
  lowthrust arcs.

#### `lowThrust.massModel` (object)

Mass model used to convert low-thrust DV into equivalent fuel-burned
estimates for plot summaries.

- `massModel.m0Kg` (number) - initial wet mass [kg].
- `massModel.earthDepartureInitialMassKg` (number) - initial mass before
  Earth departure [kg]. Used for the first departure burn estimate shown
  in the 3D summary panel.
- `massModel.ispSec` (number) - specific impulse [s].
- `massModel.g0` (number) - standard gravity [m/s^2].

---

#### `legCostModel` (string)

How the pipeline accumulates total mission DV for ranking:

- `"paper"` - Landau 2022 formula.
- `"vinfsum"` - sum of v-infinity magnitudes (default).
- `"parkingOrbit"` - include parking-orbit insertion/departure terms.

---

#### `parkingOrbit` (object)

Parking-orbit altitudes used when `legCostModel` is `"parkingOrbit"`.

- `parkingOrbit.departureAltitudeKm` (number) [km].
- `parkingOrbit.arrivalAltitudeKm` (number) [km].

---

#### `autoFlyby` (object)

Automatic insertion of intermediate flyby bodies between user-specified
slots.

- `autoFlyby.enabled` (bool) - enable automatic flyby search.
- `autoFlyby.maxFlybys` (integer) - maximum number of inserted flybys.
- `autoFlyby.allowRepeatedBodies` (bool) - allow repeated bodies in the
  inserted sequence.
- `autoFlyby.maxCandidateBodies` (integer) - max candidate bodies
  evaluated per insertion step.
- `autoFlyby.candidateBodies` (array of strings) - body pool for
  auto-flyby insertion. Example: `["EARTH", "MARS"]`.

---

#### `enablePoweredFlyby` (bool)

Powered flybys (encounter maneuver DV) are separate from powered legs
(DSM on transfer arcs):

- Powered **legs**: set a `transferType` entry to `"powered"` and use
  `dvLeg` / `dDvLeg`.
- Powered **flybys**: controlled by this flag. When `false`,
  `constraints.maxFlybyDv` is not enforced.

---

#### `runOptions` (object)

Algorithm and performance controls forwarded to the pipeline. Only the
keys set below override defaults; all others fall back to their built-in
values.

- `strictPaperMode` (bool) - paper-profile master switch.
- `combineSegmentsMode` (string) - trajectory assembly mode:
  `"dp"` (exact DP), `"dp-approx"` (DP + optional state cap),
  `"legacy"` (strict paper profile only).
- `combineDpMaxStatesPerLayer` (number or `null`) - DP-approx: max
  states kept per layer. `null` = `Inf` (disables trimming).
- `combineDpMaxFinalCandidates` (integer) - complete trajectories
  reconstructed.
- `combineDpBeamWidthPerState` (integer) - labels kept per exact DP key
  (1 = dominance).
- `diversityMode` (string; default `"exploratory"`) -
  `"refinement"` or `"exploratory"`. `{'refinement','exploratory'}`
  affects top-list selection only (not the underlying DP search).
- `diversityMinSpacingDaysBySlot` (array of numbers; default `[30]`) -
  optional per-slot spacing in days for exploratory mode.
  Single-element array (e.g. `[30]`) applies the same spacing to every
  slot; longer arrays give one value per slot.
- `flybyJoinEnabled` (bool) - flyby-join approximation. **Approximate**;
  may miss feasible flyby connections. Use for broad exploration, then
  disable for refinement runs.
- `flybyJoinKNearest` (number or `null`) - outgoing candidates per
  incoming leg. `null` = `Inf` (no K-cap).
- `flybyJoinMaxAbsVinfDiff` (number or `null`) - optional
  `abs(|vInfOut| - |vInfIn|)` cap [km/s]. `null` = `Inf`.
- `flybyJoinMinPairsToActivate` (integer) - activate only when
  `nIn * nOut >= threshold`.

---

#### `plot` (object)

Optional planet-orbit overlays and plotting knobs used by the
interactive viewer.

- `plot.enablePlanetOrbits` (bool) - overlay planet orbits in 3D plots.
- `plot.planetOrbitBodies` (array of strings) - bodies to draw as
  context.
- `plot.planetOrbitSamples` (integer) - samples per planet orbit curve.
- `plot.planetOrbitMaxStepDays` (number) - max spacing [days] between
  plotted orbit samples.
- `plot.planetOrbitMaxSamples` (integer) - hard cap on plotted samples
  per orbit body.
- `plot.showPlanetWindowEndpoints` (bool) - show slot window endpoint
  markers.
- `plot.enableLowThrustThrustFuelPlot` (bool) - low-thrust mission
  analysis figure (thrust + fuel vs mission time).

---

### Summary of fields that accept `null` for `Inf`

- `constraints.maxFlybyDv`
- `constraints.maxTotalDv`
- `tofConstraints[].maxDays`
- `runOptions.combineDpMaxStatesPerLayer`
- `runOptions.flybyJoinKNearest`
- `runOptions.flybyJoinMaxAbsVinfDiff`

Any other numeric field rejects `null` and expects a finite number.

## What NOT to touch

- Do not rename or delete the executable or any file it sits next to -
  the CTF archive that holds the LOTUS pipeline is bound to the
  executable.
- Do not attempt to unpack the `.ctf`/executable. That is covered by the
  GPL **spirit** of the project - you will get faster answers from your
  instructor than from reverse engineering the archive.

## Troubleshooting

- **"JSON configuration not found"** - run the executable from the
  folder that contains `user_input_lotus.json`, or pass an explicit path
  as the first argument.
- **"Could not initialize MICE path"** - verify the `data/spice/mice/`
  folder is present and populated with the MICE distribution from your
  instructor.
- **"Could not load SPICE kernels"** - verify the kernel folder
  (`data/spice/generic_kernels/`) contains the LSK, PCK, and SPK files
  for your mission window.
- **Interactive plot mode skipped** - your MATLAB Runtime session has
  no AWT / no display. Use a desktop session or export analyses and
  load them (option 2) on a workstation with a display.

Report bugs, feature requests, or crashes to the instructor rather than
trying to patch the executable.
