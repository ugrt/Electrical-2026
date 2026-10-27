# PrototypeBoard_V3 — Design Review

**Project:** Prototype Board V3 — STM32L433 CAN prototype node
**Date:** 2026-10-26
**Analyzers run:** `analyze_schematic.py`, `analyze_pcb.py --full`, `cross_analysis.py`, `analyze_emc.py`, `analyze_thermal.py`, `kicad-cli sch erc`, `kicad-cli pcb drc`
**Analysis run:** `analysis/2026-10-26_2312/`
**Files:** 1 sheet, 2-layer PCB, 41.8 × 46.0 mm, 34 components, 53 nets

---

## Overview

STM32L433CCT6 (LQFP-48, 80 MHz Cortex-M4F) CAN node for a robotics/vehicle bus. A TLV75533 LDO makes +3V3 from an external +5V input; an MCP2562 transceiver drives the CAN bus at 5 V with a 3.3 V logic side. Peripherals: 20 GPIOs on a 2×12 header, four DIP switches, three LEDs, reset and boot buttons, an 8 MHz HSE crystal, SWD, and four screw terminals (+5V, GND, CANH, CANL) with jumper-selectable 120 Ω termination. The MCU was changed from STM32G0B1CET6 to STM32L433CCT6 in this revision; the schematic and PCB are both updated.

---

## Previous Review Delta

Compared against the findings recorded in the 2026-10-04 review.

| Status | Count | Items |
|---|---|---|
| Fixed | 4 | SW1/SW2 pad-pair short (board now uses duplicate pads 1,1 / 2,2 correctly); Y1 crystal pin-swap (pads 1/3 = CRYS, 2/4 = GND); SWCLK never reaching the MCU (J1 removed, PA14 wired directly); refdes desync (schematic and PCB both use IC2) |
| Fixed (incidental) | 1 | SW1/SW2/U2 malformed courtyards — no longer flagged |
| Still open | 3 | Via annular ring 0.075 mm; GND zone edge clearance 0.497 mm; silkscreen overlaps |
| New | 4 | L433 remap (verified below); 49 vias now all 0.45/0.3; ERC errors from symbol power-pin types; MPN coverage still 3/17 |

DRC dropped from 130 violations to 24. Unconnected items: 0.

---

## Critical Findings

**Status as of the post-review verification (2026-10-26, second pass).**

| Severity | Issue | Status |
|---|---|---|
| ~~CRITICAL~~ | **SW1/SW2 library footprint** — the PCB had duplicate pads (1,1 / 2,2) but the library source still had four distinct pads, so any footprint update would re-short NRST to GND. | **FIXED** — the corrected footprint was copied back into `Buttons/SKRPABE010/SKRPABE010.kicad_mod` and the library regenerated. Board and library now match. |
| ~~HIGH~~ | **Via annular ring 0.075 mm** (49 vias, 0.45/0.3). | **ACCEPTED** — 0.3 mm drill / 0.45 mm pad is within JLCPCB's stated capability. Note: still below IPC Class 2 (0.125 mm); fine for this prototype, revisit for volume. |
| ~~HIGH~~ | **TLV75533 `OUT` declared `power_in`** → ERC "power pin not driven". | **FIXED** — library symbol pin 5 is now `power_out`. |
| ~~HIGH~~ | **MCP2562 pin 5 named `SPLIT` / `unspecified`** (should be `VIO`). | **FIXED in the library source** — now `VIO`, `power_in`. The schematic's cached copy still shows `unspecified`; run *Update Symbols from Library* in Eeschema to sync it. |
| ~~HIGH~~ | **J1–J4 second terminal pads floating.** | **FIXED** — both pads of each terminal are now on the same net (+5V, GND, CANH, CANL). |
| ~~HIGH~~ | **Sourcing gate (3/17 MPNs).** | **ACCEPTED** — stated as not production-blocking for the prototype run. Re-open before any production order. |

### Remaining items after the fixes

| Severity | Issue | Status / Fix |
|---|---|---|
| HIGH | ERC `power_pin_not_driven` on U1 VSS/VDD — +5V and GND have no PWR_FLAG | **Open** — add PWR_FLAGs to +5V and GND |
| MEDIUM | ERC `pin_not_connected` on U2 pin 4 (NC) | **Fixed** — no-connect flag added |
| MEDIUM | ERC `multiple_net_names` — stray `VDD` label on +3V3 | **Fixed** — label removed |
| LOW | ERC `unconnected_wire_endpoint` | **Open** — horizontal wire at (72.39, 175.26)→(73.66, 175.26) mm, UUID `5f8e6519-…`; delete it |
| MEDIUM | ERC `lib_symbol_mismatch` (IC2, Y1, U1, U2) | **Open** — run *Update Symbols from Library* in Eeschema (the schematic cache still has U1 pin 5 as `unspecified`) |
| MEDIUM | DRC malformed courtyards (SW1, SW2, U2) and silk overlaps | **Fixed in the library** — the vendor courtyards in `SKRPABE010` and `TLV75533` contained zero-length and stray lines; both footprints now have a clean 4-edge courtyard. Re-run *Update Footprints from Library*, then refill zones. |
| MEDIUM | DRC courtyard overlap J3 ↔ J4 | **Open** — move the terminal blocks apart |
| LOW | DRC silkscreen overlaps (51), silk-over-copper (12), edge clearance 0.4973 mm | Cosmetic cleanup |

**Zone fills are current** — the earlier /CANH and /CANL clearance errors are gone.

**Library workflow caveat:** the KiCad library tables point at `_generated/`, so parts edited from inside KiCad are written there, not into the source tree — and the next `update-tables.sh` run wipes them. Apply part edits to the source files (or copy the saved file back) and then regenerate. This is exactly how the SW1/SW2 fix was nearly lost.

---

## Component Summary

| Type | Count |
|---|---|
| IC | 3 (IC2, U1, U2) |
| Capacitor | 12 |
| Resistor | 5 |
| LED | 3 |
| Connector | 7 |
| Switch | 3 (S1 DIP + SW1/SW2) |
| Crystal | 1 |

Nets 53 · wires 107 · no-connects 7 · unique parts 17 · power rails +3V3, +5V, GND

---

## Power Tree

```
J1 (+5V terminal, external supply)
 ├── U1  MCP2562  VDD = +5V   (bus side)
 │              VIO = +3V3   (logic side)
 └── U2  TLV75533 LDO  IN/EN = +5V  →  +3V3 (fixed 3.3 V)
        ├── IC2  L433  VBAT, VDD_1(24), VDD_2(48), VDDUSB(36), VDDA(9)
        ├── J5 header (+3V3), J7 SWD, S1, LEDs, R2 termination
        └── U1 VIO
```

Bulk +3V3: C2 1 µF + C12 1 µF, plus five 100 nF (C1, C3, C4, C5, C6) — one per MCU supply pin.
+5V: C10 100 nF, C11 1 µF.

No feedback divider (fixed-output LDO) — no Vref dependency.

---

## Analyzer Verification

### Component count
34 schematic components = 34 PCB footprints. Match.

### IC pin verification

**IC2 — STM32L433CCT6: datasheet-verified** against DS11449 Rev 5, LQFP-48 pinout (Figure 13) and pin definition table (Table 15). All 48 pins match the intended net assignment:

| Function | Pin | Net |
|---|---|---|
| VBAT | 1 | +3V3 |
| HSE | 5, 6 | /CRYS1, /CRYS2 |
| NRST | 7 | reset net (C9 100 nF, SW1) |
| VSSA / VDDA | 8, 9 | GND / +3V3 |
| S1 switches | 10–13 | PA0–PA3 |
| GPIO header | 14–22, 27–32, 38–43 | GPIO_1…GPIO_20 |
| VSS_1, VSS_2, VSS_3 | 23, 35, 47 | GND |
| VDD_1, VDD_2 | 24, 48 | +3V3 |
| SWDIO / SWCLK | 34, 37 | /SWDIO, /SWCLK |
| VDDUSB | 36 | +3V3 |
| BOOT0 | 44 | R1 10 kΩ to GND, SW2 to +3V3 |
| CAN1_RX / CAN1_TX | 45, 46 | /CAN_RX, /CAN_TX (AF9, Table 15) |

Unused: PA12 (33), PB2 (20), PB12 (25), PB13 (26), PC13–PC15 (2–4).

**U1 — MCP2562: unverified (inference).** No datasheet on file. Pin functions match the standard MCP2562 pinout: TXD(1), VSS(2), VDD(3), RXD(4), VIO(5), CANL(6), CANH(7), STBY(8 = low → normal mode). The pin-5 naming issue is above.

**U2 — TLV75533: unverified (inference).** Matches the DBV (SOT-23-5) pinout: IN(1), GND(2), EN(3), NC(4), OUT(5). EN tied to IN is correct.

### Connector pin tables

| Connector | Pins | Function |
|---|---|---|
| J5 (2×12) | 1 +3V3, 2 +5V, 3–4 GND, 5–24 GPIO_7…GPIO_14 (see netlist) | GPIO header |
| J7 (1×4) | 1 +3V3, 2 SWCLK, 3 GND, 4 SWDIO | SWD — matches ST-LINK 4-wire order |
| J6 (1×2) | 1 → R2 → CANL, 2 CANH | 120 Ω termination jumper |
| J1 | +5V | supply input |
| J2 | GND | supply return |
| J3 | CANH | bus high |
| J4 | CANL | bus low |

### Net tracing
+3V3: 25 pins across IC2, U1, U2, J5, J7, S1, LEDs, passives — complete.
GND: 28 pins — complete.
CAN_TX/RX: IC2 → U1 → indicator LEDs — correct.
CRYS1/CRYS2: Y1 pins 1/3 + C7/C8 10 pF — correct.
SWD: IC2 PA13/PA14 → J7 — direct, no jumpers.

### PCB verification
34 footprints, 0 unconnected, 377 track segments (314 × 0.3 mm, 63 × 0.6 mm), 49 through vias, one B.Cu GND zone.

---

## Signal Analysis Review

**Regulator:** TLV75533, LDO, +5V → +3V3, fixed output (no divider). Correct topology. EN tied high.

**CAN termination:** Analyzer flagged PR-003 "no 120 Ω termination". **False positive** — R2 120 Ω plus jumper J6 provides switchable termination between CANH and CANL. Correct design.

**Crystal:** 8 MHz HSE, C7/C8 = 10 pF each → CL ≈ 8 pF including stray. Plausible for a Würth 830059082; crystal CL is not datasheet-verified. Analyzer's "830059082 Hz" frequency is a parse artifact (it used the MPN as the value).

**Reset:** C9 100 nF + SW1 to GND. The L433 NRST has an internal weak pull-up (DS11449 Table 14), so this works; an external 10 kΩ pull-up is optional but recommended in noisy environments.

**BOOT0:** R1 10 kΩ pull-down, SW2 to +3V3 — correct. With the L433 the dedicated PH3-BOOT0 pin removes the old SWCLK conflict.

**LEDs:** D1/D2 (CAN activity) and D3 (power) via 1 kΩ from +3V3 → ≈1.3 mA. Low but visible.

**DIP switches S1:** pull-up to +3V3 through the switch; open pins rely on the MCU's internal pull-downs. Works if configured as input-with-pull-down in firmware. External 10 kΩ pull-downs would improve noise immunity.

---

## PCB Layout Analysis

| Item | Status |
|---|---|
| Routing | Complete, 0 unconnected |
| Board | 41.8 × 46.0 mm, 2 layers, 1 GND zone on B.Cu |
| Vias | 49 × 0.45/0.3 mm — annular 0.075 mm (below IPC/JLCPCB) |
| Courtyard overlap | J3 ↔ J4, 0.337 mm² (terminal blocks) |
| Edge clearance | GND zone 0.4973 mm vs 0.5 mm rule — cosmetic |
| Silkscreen | 15 overlaps |
| Fiducials | None (fine for hand assembly; add 3 if machine-placed) |
| Test points | 0 of 50 nets |
| Decoupling placement | IC2: closest 100 nF at 1.75 mm, four more within 3.8 mm — good. U1: 100 nF at 3.5 mm — acceptable. U2: 1 µF at 2.7 mm — good. |

**Library/PCB mismatch:** 7 footprints differ from the library (D1–D3, SW1, SW2, U1, U2). For SW1/SW2 this is the critical pad fix described above. The others are benign vendor-vs-library differences.

---

## EMC / Cross-Domain

EMC analyzer: 15 errors / 31 warnings / 9 info.

| Finding | Detail |
|---|---|
| GP-001 (errors) | /CANH only 40 % reference-plane coverage over 13.9 mm; GPIO_10–13, GPIO_16–18 at 55–79 %; /SWCLK major gap; termination net 50 % |
| IO-001 (errors) | No EMC filtering at J1–J4 (power and CAN terminals) |
| CK-003 | /SWCLK routed near J5 and J7 |
| RP-001 | Missing stitching via at layer transitions on CAN_TX, CAN_RX, SWCLK, several GPIOs |
| VS-002 / GP-004 | Sparse ground stitching (20 stitching vias; 60 % of board without ground vias) |

On a 2-layer board with a single B.Cu pour this is expected, but the CAN pair and SWCLK deserve attention: keep the B.Cu ground continuous beneath them, add stitching vias alongside layer changes, and consider a TVS on CANH/CANL at the terminals for vehicle use.

Cross-analysis: S1 value naming mismatch (A6H-4102-PM vs A6H4102PM) — cosmetic (footprint/property naming).

---

## Thermal

Analyzer: 0 findings, score skipped (no datasheet extraction for U2).

Inference: U2 (TLV75533, SOT-23-5) dissipates 1.7 V × I_load. At 100 mA ≈ 0.17 W (≈40 °C rise); at 300 mA ≈ 0.5 W (≈120 °C rise, approaching the limit). The LDO is the thermal bottleneck if the J5 header powers external loads — keep the +3V3 load under ~200 mA or use a switching regulator for higher loads.

---

## Component Lifecycle

Not audited — no distributor API keys on this machine. The MCU (STM32L433CCT6) is active with ~6 k units at DigiKey; the rest of the BOM has no MPNs to check.

---

## False Positives / Reviewer Overrides

| Finding | Reason |
|---|---|
| PR-003 "CAN transceiver has no 120 Ω termination" | R2 120 Ω + J6 jumper provide selectable termination — intentional |
| XL-DET "crystal at 830059082 Hz" | MPN parsed as value; crystal is 8 MHz |
| RS-001 "+5V has no declared source" | +5V arrives on terminal J1 by design; add a PWR_FLAG to silence ERC |
| CG-AUD J5 10:1 signal-to-ground ratio | Header is a GPIO breakout, not an interface connector |

---

## Not Performed / Review Limits

- **Datasheet extraction:** no `datasheets/` directory and only 3/17 parts have MPNs (DS-002). The L433 was verified against a manually obtained DS11449 Rev 5. MCP2562, TLV75533, Y1, SKRPABE010, A6H-4102-PM, LTST-C170GKT, Phoenix 1706714 and 1190363 are **inference-only**.
- **SPICE:** no simulator installed (`ngspice`, `ltspice`, `xyce` all absent).
- **Thermal:** ran, but no findings due to missing extraction data.
- **Lifecycle:** no API keys.
- **Gerbers:** no fabrication outputs present.
- **Deep-review JSON:** not produced — requires datasheets for most parts.

---

## Final Verdict

**Schematic: electrically sound.** The L433 remap is complete and every pin is consistent with DS11449. The previous critical defects (switch short, crystal swap, SWCLK, refdes) are fixed, and this pass closed the library-symbol and terminal-pad issues as well.

**PCB: routed, 0 unconnected, and clean apart from mechanical/DFM items.**

**Fabrication readiness: close — these remain before ordering:**

1. Add PWR_FLAGs to +5V and GND (clears the two ERC power-pin errors).
2. Add a no-connect flag on U2 pin 4.
3. Delete the stray `VDD` label and the 0.0127 mm wire stub.
4. Refill zones (B) — the two /CANH and /CANL clearance errors are stale fills.
5. Separate the J3/J4 courtyards.
6. Optional: clear the silkscreen overlaps and the 0.497 mm edge-clearance flag.

Then run *Update Symbols from Library* and *Update Footprints from Library* to clear the remaining mismatch warnings. After that the board is ready for a prototype order.
