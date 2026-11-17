# PCB-Laser — Implementation Plan

Source idea (Todoist, "Better desktop PCB fabrication"):
> One-click workflow converting a set of Gerbers into commands for the ComMarker B4 to etch PCBs.
> Simple solder-mask screen-printing jig, then etch the solder mask with the laser too.
> Can we command the laser directly instead of using EzCAD?

## 1. Target outcome

```
pcblaser run board.zip --side top --process copper,mask,cut
```
→ parse Gerber/Excellon → generate per-layer toolpaths → preview → red-dot frame for alignment → fire laser.
Single tool, Python, no EzCAD in the loop (EzCAD export kept as a fallback).

## 2. Key technical decisions

| Topic | Decision | Rationale |
|---|---|---|
| Laser control | `galvoplotter` (meerk40t project) talking USB to the BJJCZ LMC board | Open-source Python driver for the same JCZ controller EzCAD2 uses; supports marking, light (red-dot) mode, speed/power/freq/pulse settings, and EzCAD `.cor` correction files. |
| Fallback output | DXF per layer with pen assignments for EzCAD import | Unblocks testing before direct control works, and if the board turns out to be EzCAD3/DLC. |
| Gerber parsing | `gerbonara` → `shapely` geometry | Handles RS-274X + Excellon, polarity, apertures; shapely gives booleans/offsets. |
| Copper process | Direct ablation of copper with 1064 nm fiber (no chemistry) | Q-switched pulses ablate 1 oz Cu; isolation + optional full clearing. Chemical route (ablate paint resist → FeCl₃/persulfate) kept as alt process for large boards. |
| Solder mask | Screen/squeegee UV-curable mask, UV cure, laser-ablate pad openings | Mask openings come straight from the GTS/GBS layer; light second pass cleans residue off pads. |
| Silkscreen | Low-power mark on cured mask (color change), optional | Free once the mask process works. |
| Holes / outline | Laser spiral-cut holes ≥ ~0.6 mm and board outline; small holes optionally just center-marked for a drill | Fiber chars FR4 slowly; measure before committing. |

**Verify first:** controller board in the B4 (Settings/About in EzCAD, or USB VID/PID: JCZ LMC = `9588:9899`). EzCAD2 → galvoplotter works. EzCAD3/DLC2 → direct control is much harder; fall back to DXF export.

## 3. Architecture

```
pcblaser/
  ingest.py        # unzip, identify layers by name/X2 attributes, load with gerbonara
  geometry.py      # shapely: copper polygons, isolation regions, mask openings, outline, drills
  toolpath.py      # isolation contours (n offsets), hatch fills (angle/spacing), hole spirals, ordering
  transform.py     # board → lens-field coords: mirror for bottom, rotation, fiducial alignment, field fit check
  process.py       # parameter library (YAML): power, speed, freq, pulse width, passes, hatch spacing per op
  drivers/
    galvo.py       # galvoplotter backend: light/frame, mark, abort, status
    dxf.py         # EzCAD fallback export
  preview.py       # matplotlib render of all ops over the board
  testgrid.py      # power×speed×freq matrix generator for material tests
  cli.py           # `pcblaser run|preview|frame|testgrid|calibrate`
processes/
  fr4_1oz_cu.yaml
  uv_mask_green.yaml
jigs/              # CAD for fixture + screen jig (Onshape export / STL)
```

Data flow per job: `layers → ops[] (each: geometry, process params, pass count) → field coords → driver`.
Every op is a list of polylines + a param set; the driver only knows "set params, mark polylines".

## 4. Phases

### Phase 0 — Machine bring-up (1–2 evenings)
- Identify controller/board, lens (field size, focal length), laser source (Raycus Q-switch vs JPT MOPA), wattage.
- Install galvoplotter; on Windows swap EzCAD driver for WinUSB via Zadig (reversible) or run from Linux with libusb.
- Load EzCAD's `.cor` file; draw a 50 mm square in light mode and marked on anodized Al; verify dimensions with calipers.
- **Exit:** script marks a known square to ±0.05 mm, red-dot framing works.

### Phase 1 — Copper process development (a few evenings)
- `testgrid.py`: matrix of power/speed/freq/passes/hatch spacing on scrap FR4.
- Measure: Cu fully cleared (continuity test across gap), FR4 charring, min clean gap, trace undercut.
- Expect ~20–30 µm hatch spacing, multiple passes, cross-hatching at 0/90°; a final low-power "clean" pass.
- **Exit:** `fr4_1oz_cu.yaml` that reliably isolates 0.2 mm traces/0.2 mm gaps (stretch: 0.15 mm).

### Phase 2 — Gerber → toolpath pipeline (core software)
- Ingest KiCad/EasyEDA zips, auto-detect layers.
- Isolation mode: N offset contours around copper (fast). Clear mode: hatch-fill all non-copper inside the outline (cleaner, slower). Hybrid: isolation + clear within X mm.
- Path ordering (nearest-neighbor) to cut galvo jump time; split jobs that exceed the field.
- Preview PNG; field-fit and min-feature warnings (feature < spot size).
- **Exit:** `pcblaser preview` matches the KiCad 3D/Gerber viewer for 3 test boards; DXF export imports into EzCAD correctly.

### Phase 3 — Direct control + one-click run
- `drivers/galvo.py` with frame → confirm → run → abort.
- Op sequencing per process (e.g. copper → outline → holes), pauses for operator steps.
- **Exit:** single-sided board from Gerber zip to isolated copper with one command, no EzCAD.

### Phase 4 — Fixtures and double-sided registration
- Bed fixture: aluminum plate with 2 dowel pins at known positions in lens field, board blanks pre-drilled (or laser-cut) with matching registration holes.
- Pins placed symmetric about the flip axis so bottom side registers by flipping on the same pins; software mirrors bottom layers about that axis.
- Optional fiducial correction: mark/locate 2 fiducials with the red dot, solve a 2D rigid transform.
- Vias: unplated — rivets, or wire through + solder.
- **Exit:** double-sided test board with ≤0.1 mm layer-to-layer misregistration.

### Phase 5 — Solder mask
- Screen-print jig: hinged frame on the same dowel-pin plate, 90–120 mesh screen (or stencil + squeegee for uniform thickness), UV-curable mask, UV cure (395/405 nm LED).
- Process grid for mask ablation over copper: remove mask fully without blackening pads.
- Generate mask-opening ops from GTS/GBS (with expansion offset param), optional silkscreen marking from GTO/GBO.
- **Exit:** solderable pads on a masked board, 0.5 mm pitch TSSOP pads open cleanly.

### Phase 6 — Polish
- Simple GUI (single window: load zip, choose side/ops, preview, frame, run) — or keep CLI.
- Job log (params used, time, notes) for process tuning.
- Hole drilling evaluation: laser spiral vs mark-and-drill.

## 5. Risks

| Risk | Mitigation |
|---|---|
| B4 ships with EzCAD3/non-LMC board | Check in Phase 0; DXF fallback keeps the rest of the pipeline useful. |
| FR4 charring / delamination from Cu ablation | Lower energy, more passes, air assist + fume extraction; prefer isolation over full clearing. |
| Copper reflectivity → inconsistent ablation | Tune on real stock; a light pre-oxidize/matte pass can raise absorption. |
| Galvo field distortion at edges | Use `.cor`, keep boards near field center, calibrate with a measured grid. |
| Fumes (brominated FR4, mask resin) | Mandatory fume extraction/filtering; enclosure. |

## 6. Open questions for you
1. B4 wattage, source type (MOPA?), and lens/field size?
2. EzCAD2 or EzCAD3 on your unit?
3. Windows or Linux on the laser PC?
4. Target min trace/space, and is double-sided required for v1?
5. Is a chemical etch step acceptable, or strictly dry/laser-only?
