# PCB-Laser — Implementation Plan

Source idea (Todoist, "Better desktop PCB fabrication"):
> One-click workflow converting a set of Gerbers into commands for the ComMarker B4 to etch PCBs.
> Simple solder-mask screen-printing jig, then etch the solder mask with the laser too.
> Can we command the laser directly instead of using EzCAD?

## 1. Target outcome

```
pcblaser run board.zip --side top      # copper → mask → cut, pauses for operator steps
pcblaser run board.zip --side bottom   # mirrored about the registration-pin axis
```
Linux, Python, no EzCAD in the loop. Gerber/Excellon → toolpaths → preview → red-dot frame → mark.

## 2. Requirements (from you)

- Laser-only process (no chemical etch).
- Double-sided required.
- Finest feature: 0.5 mm pitch parts (≈0.25 mm pads / 0.2–0.25 mm gaps).
- Runs on Linux.

## 3. Machine facts (from the B4 manual + your notes)

| Item | Value | Source |
|---|---|---|
| Source | 30 W fiber, 1064 nm, Q-switched (not MOPA, no pulse-width control) | you; manual (LightBurn import: `IPG_YLP`, Q-pulse width disabled) |
| PRF range | 20–200 kHz | manual (LightBurn device setup) |
| Controller | BJJCZ LMC ("Laser Mark Control Board V2 [USB]", `USBLMCV2`), EzCAD 2.14.11 | manual |
| USB ID | `9588:9899` | galvoplotter source |
| Galvo mapping | Galvo 2 = X | manual |
| Lenses | 110×110 mm (F≈160) and 200×200 mm | manual / you |
| Correction files | `jcz11.cor` (110 mm folder), separate `.cor` in 200 mm folder, on the USB stick | manual |
| EzCAD config | `plug/markcfg7` in each lens folder | manual (LightBurn import step) |
| EzCAD default timing | Start TC −300 µs, Laser-off TC 150 µs, End TC 300 µs, Polygon TC 100 µs | manual screenshots |

**Lens choice:** use the 110 mm lens for all PCB work. Spot size scales ~linearly with focal length, so the 200 mm lens gives roughly 1.6× the spot diameter and ~2.5× lower fluence. That is worse for 0.2 mm gaps and slower for ablation. 110 mm usable field fits boards up to ~90 × 90 mm with margin; larger boards are out of scope for v1 (galvo + no XY stage = no tiling).

## 4. Starting recipe (your "Laser Engraver Recipie List" → PCB Manufacturing)

| Op | Speed [mm/s] | Power [%] | Freq [kHz] | Hatch [mm] | Passes | Notes |
|---|---|---|---|---|---|---|
| Copper cut | 1500 | 100 | 30 | 0.05 | 20 | rotate hatch angle periodically |
| Mask ablate | 1000 | 50 | 100 | 0.05 | 4 | |
| Pad clean | 1000 | 75 | 50 | 0.05 | 1 | light etch of pads |
| Drilling | — | — | — | — | — | not developed yet |

Lens these were tuned with is unrecorded; assume 110 mm, check in Phase 1.
Seeded into `processes/b4_30w_110mm.yaml`.

Time scale: full-clearing a 50 × 50 mm area at 0.05 mm hatch = 50 m of path/pass → 33 s/pass → ~11 min for 20 passes. Isolation-only (a ~0.3 mm band around each net) is several times faster. Default: isolation band + optional full clear.

## 5. Key technical decisions

| Topic | Decision |
|---|---|
| Laser control | `galvoplotter` 0.2.0 (meerk40t project). Pure Python over `pyusb`; on Linux just needs a udev rule, no Zadig/driver swap. Supports mark/goto/light (red-dot), speed/power/freq/delays, loads EzCAD `.cor` (both `LMC1COR_1.0` float and int formats), and has a **mock connection** for offline testing. Pin the version; vendor and patch if needed (it's a small, lightly maintained library, e.g. `set(delay_polygon=…)` currently doesn't store the value). |
| Coordinates | Galvo units 0–65535 across the field; 110 mm lens → ~595.8 units/mm. Our transform: board mm → field mm → galvo units, with `.cor` applied in-controller and an optional residual correction on top (Phase 0b). |
| Gerber parsing | `gerbonara` → `shapely`. |
| Copper | Direct ablation, cross-hatched with rotating angle (per your recipe). |
| Holes / vias / outline | Laser spiral-cut holes and outline. **Use 0.8 mm FR4** for v1: halves the cut depth through glass/epoxy vs 1.6 mm. Vias are unplated: through-wire or rivets, sized ≥0.4 mm hole. |
| Solder mask | Screen-print UV-curable mask on the pin fixture, UV cure, laser-ablate openings from GTS/GBS (+expansion), pad-clean pass. |
| Silkscreen | Optional low-power mark on cured mask from GTO/GBO. |
| Fallback | DXF export per op for EzCAD on Windows (useful for A/B comparison during process tuning). |

## 6. Double-sided registration (the critical part)

Fixture plate bolted to the B4 table's threaded hole grid, never moved during a job. Two pin holes are **cut by the laser itself** into the plate. That puts pin positions in the laser frame by construction, with no measurement step.

Per board:
1. Pins out. Blank (oversized ~10 mm per side) placed roughly over the pin holes, held by tape/stops.
2. Laser cuts the two registration holes through the blank at the pin coordinates.
3. Pins pushed through blank into plate. Run top side: copper → holes/vias → (outline last, after bottom).
4. Lift, flip about the line through both pins, re-seat on pins. Software mirrors bottom layers about that same line. Run bottom copper.
5. Mask: print on fixture (screen frame registers to the same pins), cure, ablate openings per side.
6. Outline cut last; tabs optional.

Error budget: pin/hole clearance (target ≤0.05 mm, tune hole diameter by test) + galvo repeatability (µm-level) + residual field distortion at mirrored positions (Phase 0b). Target ≤0.1 mm side-to-side, which is fine for 0.5 mm-pitch parts and unplated vias with ≥0.15 mm annular ring.

Because the flip maps a point at (x, y) to (x, 2·a − y) in the field, any uncorrected field distortion shows up twice in misregistration. Calibrating the field is therefore part of the double-sided work, not optional.

## 7. Architecture

```
pcblaser/
  ingest.py        # unzip, identify layers (filename / X2 attrs), load with gerbonara
  geometry.py      # shapely: copper, isolation regions, mask openings, outline, drills
  toolpath.py      # isolation bands, hatch fills (angle rotation per pass), hole spirals, path ordering
  transform.py     # board → field mm (placement, pin-axis mirror) → galvo units; residual correction
  process.py       # load recipe YAML; per-op param sets
  calibrate.py     # grid mark + scanner-image fit → residual correction map
  drivers/
    galvo.py       # galvoplotter backend: frame (red dot), mark, abort, status
    mock.py        # galvoplotter mock + path recorder for tests/preview
    dxf.py         # EzCAD fallback
  preview.py       # matplotlib render of ops over board
  testgrid.py      # power × speed × freq × passes matrix generator
  cli.py           # pcblaser run | preview | frame | testgrid | calibrate | fixture
machine/           # copied from USB stick: .cor files, markcfg7 per lens
processes/         # recipe YAMLs
fixtures/          # CAD: pin plate, screen-print frame
```

Every op = (list of polylines, param set, pass count). The driver only knows "set params, mark polylines".

## 8. Phases

### Phase 0 — Linux bring-up
- Copy `EZCAD for ComMarker B4/` (both lens folders) from the USB stick into `machine/`.
- udev rule for `9588:9899`; `galvoplotter` with `cor_file=machine/110mm/jcz11.cor`, galvos_per_mm from field size, timing from `markcfg7`.
- Red-dot frame a square, then mark 50 mm square on anodized Al / black card; verify with calipers.
- **Exit:** ±0.05 mm on a 50 mm square, red-dot framing works, abort works.

### Phase 0b — Field calibration (replaces Windows-only CorFile2)
- Mark a 9×9 (or denser) crosshair grid over the usable field on black card or anodized Al.
- Scan on a flatbed at 1200 dpi (~21 µm/px), detect crosshairs, fit residual correction (2D polynomial or bilinear grid) on top of `.cor`.
- **Exit:** residual < 0.03 mm across the central 90 × 90 mm, verified with the mirror-symmetric test (mark, re-mark mirrored, compare).

### Phase 1 — Process verification on Linux
- Reproduce your recipe through galvoplotter; confirm it matches EzCAD results (same params should give the same result, but power/frequency conversions in the library are worth checking).
- Test grid around the recipe on 0.8 mm FR4: min clean gap, char, undercut.
- Develop the missing **drilling** recipe (spiral pitch, passes) and outline cut.
- **Exit:** 0.2 mm trace / 0.2 mm gap isolates reliably; 0.4–1.0 mm holes through 0.8 mm FR4.

### Phase 2 — Gerber → toolpath pipeline
- Ingest KiCad/EasyEDA zips; isolation band + optional clear; hatch rotation per pass; nearest-neighbor ordering; field-fit and min-feature checks.
- Preview PNG; mock-driver tests on 3 reference boards.
- **Exit:** preview matches the Gerber viewer; mock output path length/time estimates are sane.

### Phase 3 — Single-sided one-command run
- `pcblaser run` with frame → confirm → mark → abort; operator pauses between ops.
- **Exit:** single-sided board from zip to isolated, drilled, outlined board with one command.

### Phase 4 — Fixture + double-sided
- Pin plate (Al or FR4 sheet on standoffs bolted to the table), laser-cut pin holes, press-fit dowels sized by test.
- Registration-hole op, pin-axis mirror transform, flip workflow.
- **Exit:** ≤0.1 mm top/bottom misregistration on a test coupon (concentric rings top vs bottom, measured under a microscope/scanner).

### Phase 5 — Solder mask
- Screen-print frame hinged on the pin plate; UV mask, UV LED cure.
- Mask-opening ops from GTS/GBS with expansion; pad-clean pass; optional silkscreen marking.
- **Exit:** clean, solderable openings on a 0.5 mm-pitch footprint, both sides.

### Phase 6 — Polish
- Minimal GUI or keep CLI; job log (params, time, outcome) to feed the recipe file.

## 9. Risks

| Risk | Mitigation |
|---|---|
| galvoplotter gaps/bugs (small project) | Mock-driver tests; vendor + patch; meerk40t (same protocol, GUI) as a cross-check on Linux. EzCAD on Windows stays available. |
| Field distortion → misregistration on flip | Phase 0b calibration; keep boards centered on the pin axis. |
| FR4 charring at holes/outline | 0.8 mm stock, more passes at lower energy, air assist. |
| Fumes (brominated FR4, mask resin) | Enclosure + extraction/filtration before any FR4 work. |
| Recipe tuned on unknown lens | Verify in Phase 1 before building on it. |

## 10. Open items
1. Which lens was the recipe tuned with?
2. Copy the USB stick's `EZCAD for ComMarker B4` folder into `machine/` (or Drive's "ComMarker B4 Laser Engraver" folder, then I can pull it).
3. Do you have a flatbed scanner for Phase 0b? (Otherwise a microscope + calipers works, slower.)
