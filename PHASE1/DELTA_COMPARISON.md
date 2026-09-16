# Delta Comparison — Beckhoff/York Vision Code vs Ponda URS

**Purpose:** map what the current Beckhoff vision **IPC** and **vision HMI** do, what the Ponda URS requires, and what must change. Based on:
- Code: `Vanguard_Wafer_Inspection.library` (`ProcessSKU.Process`) + `Nestle_Base.library` + TwinCAT HMI project (`Wafer_Module_HMI` / TF2000)
- URS: `documents/PHASE_1/PHASE_1 - Manual Documents/URS Bar Rejection Vision System R7_Software devlopment.pdf`
- Delta sheet: `documents/PHASE_1/Faclon_Now_to_W1_WBS.xlsx` → `F.2 vision deltas` (Vision IPC #1–17 and Vision HMI #1–13)

**Scope of this file:**
- **Vision IPC** = inspect stack on the C6030 (grab, `ProcessSKU`, EIP tags, reject store).
- **Vision HMI** = 1× camera/program-access screen at inspect (URS 0200). **Not** the Maxtron operator PanelView.
- **Not in F.2:** operator HMI enable/stats, ControlLogix shift register, airknife valves (those are F.3 / Maxtron).

---

## 1. Executive summary

The Beckhoff code is a **working classical-CV wafer inspector**, but it implements the **York/Peacock** rules, not the **Ponda URS 4.2** rules. The biggest gaps are:

1. **Pass/fail law** — York uses height-band + area-% + delamination; URS 4.2 uses explicit pass categories (full bar, 80% bar, one-layer-missing).
2. **Geometry** — York assumes 27 cavities; Ponda uses SKU packs (2F 2×30, Mini 3×30, 3F 2×20, Chunky).
3. **SKU coverage** — York inspects Chunky only, others are Scan-Only; Ponda requires Mini, 2F, 3F and Chunky all to inspect.
4. **Nozzle mask** — York produces a 27-bit map; Ponda needs a 48-bit mask grouped by SKU.
5. **Tracking / handshake** — QR + shift-register tracking and mould good/bad DO are not in the IPC code.

---

## 2. URS 4.2 pass/fail vs current code

### URS 4.2 requirement

| Condition | Result |
|---|---|
| Full bar with all wafer layers in place | **PASS** |
| 80% bar with all wafer layers in place | **PASS** |
| Full bar with one wafer layer missing (delaminated) | **PASS** |
| All other delaminated or short bars | **FAIL** |

### Current code logic

```iecst
WaferResults[CurrentWaferIndex].WaferStatus :=
    ProcessWaferResults[CurrentWaferIndex].WaferPercentageOK
    AND ProcessWaferResults[CurrentWaferIndex].WaferHeightOK
    AND ProcessWaferResults[CurrentWaferIndex].WaferDelamOk
    AND ProcessWaferResults[CurrentWaferIndex].WaferDimensionsOk;
```

Where:
- `WaferPercentageOK` = `PercentageWaferInCavity > PassingWaferCavityPercentage`
- `WaferHeightOK` = mean/SD height within min/max limits
- `WaferDelamOk` = laminate layer sum within min/max
- `WaferDimensionsOk` = length/width within min/max

### Gap

- **One-layer-missing is currently a FAIL** (via `WaferDelamOk` or `WaferHeightOK`), but URS says it must be **PASS** if the bar is full length.
- **80% rule is a free slider** (`PassingWaferCavityPercentage`), but URS locks it at **80% wafer/cavity area** and requires **all layers present**.
- **No explicit "sticker" (double wafer) check**; only generic height limits.

---

## 3. Delta sheet → Vision IPC mapping

These 17 rows are F.2 **Vision IPC** only. PLC-side DI/DO/tracking is F.3.

| Delta # | Point | Current code location | Required change | Size |
|---|---|---|---|---|
| 1 | Code base | Whole `Vanguard_Wafer_Inspection.library` | Keep + patch. Do not replace pipeline. | Small |
| 2 | Grab / acquire | `SimpleCamera` + `VisionChannel` | Keep. Add replay wrapper for offline images. | Small |
| 3 | Pocket / contour | Fiducial + fixed `BarLocations`/`CavitesOffsets` | Retune for Goa moulds; add contour/interpolation fallback. | Medium |
| 4 | Pass/fail law | `WaferStatus` assignment in `ProcessSKU.Process` | Replace with URS 4.2 logic. | Medium |
| 5 | One layer missing | `WaferDelamOk` / `WaferHeightOK` | Change to PASS when full length + exactly one layer missing. | Medium |
| 6 | 80% rule | `PassingWaferCavityPercentage` | Lock to 80% wafer/cavity area; combine with layer check. | Small |
| 7 | SKU set | Recipe-driven, but no Scan-Only policy in code | Ensure all SKUs run inspect (not Scan-Only). | Medium |
| 8 | Geometry | `VisionRecipe.SKU.NumberOfBars`, `BarLocations`, `CavitesOffsets` | Load per-SKU packs (2F 2×30, Mini 3×30, 3F 2×20, Chunky). | Large |
| 9 | Rate | Timing FBs present but not exposed | Verify 135 rows/min for Mini, 90 for others. | Medium |
| 10 | FOV | `VisionRecipe.Pixels.X_mm` / image width | Set inspect FOV to 1078 mm (not 922 mm airknife span). | Small |
| 11 | Nozzle mask | Not in vision code | Add 48-bit mask builder grouped by SKU (pairs/triplets/quads). | Large |
| 12 | SKU select | No DI handling in `ProcessSKU` | Add DI → recipe/program load. | Medium |
| 13 | Line handshake | `EncodeBeckhoffData` sends `WaferResultStruct` array | Add single mould good/bad bit to EIP output. | Medium |
| 14 | Tracking | Not in vision code | Keep QR attach; support shift-register mode without blocking on NoRead. | Small (IPC) |
| 15 | Reject images | Not in vision code | Add rolling 3-shift reject image store. | Medium |
| 16 | Burst alarm | Not in vision code | Add 1-minute detect counter >10 → alarm. | Small |
| 17 | Bottom/top reveal | Not present | Out of Phase 1 scope. | None |

### 3.1 Vision HMI deltas (F.2 — was missing from earlier markdown)

The original analysis skipped HMI. These rows are the full F.2 Vision HMI keep / change / add list from the WBS. **Operator PanelView is not in this list.**

| HMI # | Point | York today | Required change | Action | Size |
|---|---|---|---|---|---|
| 1 | Role | 1× vision HMI at inspect, separate from PLC HMI | Keep one vision HMI at inspect. Do not merge into operator PanelView. | Keep | None |
| 2 | Access | Operator = results; Engineer = params + extra pages | Keep Operator vs Engineer login. Engineer = program access, URS 4.2 config, monitoring. Operator cannot change pass/fail values. | Keep + patch | Small |
| 3 | Live view | Red/green cavities + 3D image; 27-box grid | Keep live 3D + red/green. Change grid from 27 boxes to loaded SKU pack (Mini 3×30, 2F 2×30, 3F 2×20, Chunky). Counts follow pack. | Keep + patch | Medium |
| 4 | Banner | Vision RUNNING/OFF; camera CLEAR/BLOCKED | Keep as **status only**. Do not add the inspect-enable toggle here. | Keep | None |
| 5 | SKU programs | Not per Mini/2F/3F/Chunky | Add selectable vision programs Mini / 2F / 3F / Chunky (tied to SKU DI). Live view and engineer values switch with the program. | Add | Medium |
| 6 | Engineer values | Camera height, min/max wafer height, area % | Replace with URS 4.2 controls: layers, 80% wafer/cavity area, one-layer-missing = PASS. | Change | Medium |
| 7 | Reject image access | Debug dump folder on IPC | Add browse of last 3 shifts of reject images from this HMI (overwrite oldest). Engineer opens it here, not via a debug folder. | Add | Medium |
| 8 | Clean screen | 10 s lock | Keep. Not in URS. | Keep | None |
| 9 | Flag next mould | Test flag → reject alerts | Keep for FAT/SAT. Not in URS. | Keep | None |
| 10 | Engineering live steps | Acquire → threshold → contour → cavity | Keep. Relabel so steps match URS 4.2, not only York height/area. | Keep + patch | Small |
| 11 | Kiosk | Chrome fullscreen; Exit to Windows on IPC | Keep Chrome kiosk unless Nestlé mandates TF2000 in writing. | Keep | None |
| 12 | Language | English | Keep English (URS 9.7). | Keep | None |
| 13 | Enable inspect | Toggled from operator HMI; banner shows status | Keep inspect-enable as **status on this HMI only**. Toggle stays on operator HMI (Maxtron). | Keep | None |

**HMI work that is Change or Add (must be built/tested):** #3 grid, #5 SKU programs, #6 URS 4.2 engineer pages, #7 reject-image browser. Everything else is keep (or keep + small relabel).

---

## 4. Geometry / SKU comparison

| Parameter | York (current) | Ponda URS |
|---|---|---|
| Mould plate | 922 mm (assumed) | 1122 mm |
| Inspect FOV | Not explicit in code | 1078 mm |
| Cavities | 27 | 2F: 2×30=60, Mini: 3×30=90, 3F: 2×20=40, Chunky: from overlap |
| Bars/row | 27 | 24 (2F/Mini), 16 (3F), 12 (Chunky) |
| Fingers/row | 27 | 48 |
| Nozzles | 27 bits | 48 bits, grouped by SKU |
| Line rate | Chunky-class | Mini 135 rows/min, others 90 |

**Action:** the `VisionRecipe.SKU` structure must be extended or replaced with per-SKU packs loaded from CAD/JSON, and the inspection loop must use the pack geometry instead of fixed 27-cavity assumptions.

---

## 5. What does NOT need to change

- **PackML framework** (`ModeController`, `StateController`, `Wafer_Inspection_Module`) — solid, keep.
- **EIP encode/decode** (`DecodeRockwellData`, `EncodeBeckhoffData`) — keep, extend for new tags.
- **Calibration / median / fiducial / occlusion removal** — sound classical CV, keep.
- **Per-wafer measurement code** (fill %, dimensions, delamination, height stats) — keep as measurement primitives; only the **decision logic** changes.

---

## 6. Recommended patch strategy

1. **Keep the library split.** Patch `ProcessSKU` and recipe structures inside `Vanguard_Wafer_Inspection.library`; do not fork the whole stack.
2. **Introduce a `URS_PassLaw` method** inside `ProcessSKU` that takes the existing measurements and returns PASS/FAIL per URS 4.2.
3. **Add SKU pack loader** (JSON or recipe FB) that populates `NumberOfBars`, `BarLocations`, `CavitesOffsets`, nozzle grouping, and FOV for Mini/2F/3F/Chunky.
4. **Add 48-bit nozzle mask builder** in the IPC (or PLC if Maxtron owns it) that maps failed bars to grouped nozzles.
5. **Add mould good/bad DO** to `EncodeBeckhoffData` for the line shift register.
6. **Add rolling reject image store** and burst counter as separate FBs in the library.
7. **Patch vision HMI** for SKU grid, Mini/2F/3F/Chunky programs, URS 4.2 engineer values, and 3-shift reject-image browse. Do not move operator enable / stats onto this screen.

---

## 7. Open questions / risks

- **Chunky geometry:** is Chunky treated as 4F (quads) or a separate pack? Confirm from CAD.
- **Sticker detection:** URS mentions stickers; current code has no explicit check. Confirm required height/layer limits.
- **Encoder/PE interface:** the code assumes a camera trigger; how the SICK Ruler-3000 is triggered by the line encoder is not visible in this library. Confirm with Maxtron/Beckhoff.
- **Recipe source:** are SKU packs loaded from JSON on disk, from the PLC, or from the HMI? The commented-out JSON code in the library suggests a JSON loader was planned.
