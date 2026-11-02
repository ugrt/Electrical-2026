# PrototypeBoard_V3 — Hand-Assembly Functional Review

**Project:** STM32L433 CAN prototype node
**Date:** 2026-11-01
**Question:** Will this board work when hand-assembled, reliably and cleanly?
**Analyzers run:** `analyze_schematic.py`, `analyze_pcb.py --full --proximity`, `cross_analysis.py`, `analyze_emc.py`, `analyze_thermal.py`
**Prior review:** `docs/design-review-2026-10-26.md` (the 2026-10-26 pass)
**Files:** 1 sheet, 2-layer PCB, 42.8 × 46.0 mm, 34 components, 49 nets

---

## Verdict

**Yes — the circuit is electrically sound and should work when hand-assembled.**

There are **no circuit-level blockers**: power, reset, boot, clock, SWD, CAN, GPIO, switches, and LEDs are all wired correctly, and every one of the 34 PCB footprints matches its schematic net assignment. The 2026-10-26 review's critical bugs (switch pad short, crystal pin swap, SWCLK open) are fixed.

The real risks are **hand-assembly and wiring mistakes**, not the schematic:

1. The terminal blocks are unlabeled and each block has **both pins on the same net**.
2. The DIP switch inputs **float unless firmware enables internal pull-downs**.
3. There is **no reverse-polarity protection** on the 5 V input.
4. The LQFP-48 is the only genuinely challenging solder joint.
5. A few layout items are "works, but not clean" (crystal routing, U1 VIO bypass, no mounting holes).

---

## What to do before you power it up

| # | Check | Why |
|---|-------|-----|
| 1 | **Label or meter-check J1–J4** | J1 is +5 V on **both** pins, J2 is GND on both, J3 is CANH on both, J4 is CANL on both. A 2-pin power cable plugged into J1 would short 5 V to GND. |
| 2 | **Check 5 V polarity before applying power** | No reverse-polarity protection. Reverse 5 V will kill U2, U1, and likely the MCU. |
| 3 | **Enable internal pull-downs on PA0–PA3** | S1 connects PA0–PA3 to +3 V3 when closed; when open, the pins float unless firmware sets `GPIO_PULLDOWN`. Add 4× 10 kΩ external pull-downs if you want hardware certainty. |
| 4 | **Decide on J6 termination** | Install a shunt for 120 Ω across CANH/CANL if this board is a bus end. Leave open if the bus is already terminated elsewhere. |
| 5 | **Wire J7 by its labels, not a pre-made cable** | J7 is 3V3 / SWCLK / GND / SWDIO. Many 4-pin ST-Link cables are 3V3 / SWDIO / GND / SWCLK — pins 2 and 4 are swapped. |
| 6 | **Inspect the LQFP-48 for bridges** | 0.5 mm pitch, 48 pins. Use flux + drag solder or hot air, then inspect with magnification. |
| 7 | **Use C0G/NP0 for C7 and C8** | These are the 10 pF crystal load caps; X7R is less stable. The 100 nF and 1 µF caps can be X7R. |
| 8 | **Plan mounting** | There are no mounting holes. Use standoffs, a bracket, or rubber feet. |

### First-power checklist

1. With the board unpowered, measure resistance from +5 V (J1) to GND (J2). It should not be a dead short; the 1.1 µF input caps will charge during the measurement.
2. Measure +3V3 (J5 pin 1 or C12) to GND. It should not be a dead short.
3. Apply 5 V to J1 (J1 both pins) and GND to J2 (J2 both pins).
4. Measure +3V3 at J5 pin 1: expect 3.3 V ±2%.
5. Measure current: expect roughly 20–60 mA with the MCU idle and LEDs on.
6. Connect ST-Link to J7: 3V3 → pin 1, SWCLK → pin 2, GND → pin 3, SWDIO → pin 4.
7. Verify SWD connects. If it does not, hold SW1 (reset) while connecting, then release.
8. Program a blink test on a GPIO or the power LED, then connect CAN to J3/J4 and verify traffic.

---

## Connector wiring

| Connector | Pin(s) | Net | Function |
|-----------|--------|-----|----------|
| **J1** | 1_1, 1_2 (both) | +5V | 5 V supply input |
| **J2** | 1_1, 1_2 (both) | GND | Supply ground |
| **J3** | 1_1, 1_2 (both) | CANH | CAN high — daisy-chain both positions |
| **J4** | 1_1, 1_2 (both) | CANL | CAN low — daisy-chain both positions |
| **J5** | 1 | +3V3 | 3.3 V rail |
| **J5** | 2 | +5V | 5 V rail |
| **J5** | 3, 4 | GND | Ground |
| **J5** | 5–24 | GPIOs | See GPIO table below |
| **J6** | 1 → R2 → CANL | Termination | 120 Ω selectable termination |
| **J6** | 2 | CANH | Termination |
| **J7** | 1 | +3V3 | SWD 3.3 V |
| **J7** | 2 | SWCLK | SWD clock (PA14) |
| **J7** | 3 | GND | SWD ground |
| **J7** | 4 | SWDIO | SWD data (PA13) |

### GPIO header J5 (2×12)

| J5 pin | Net | MCU pin | J5 pin | Net | MCU pin |
|--------|-----|---------|--------|-----|---------|
| 1 | +3V3 | — | 13 | GPIO_15 | PB11 (22) |
| 2 | +5V | — | 14 | GPIO_9 | PA11 (32) |
| 3 | GND | — | 15 | GPIO_16 | PB10 (21) |
| 4 | GND | — | 16 | GPIO_10 | PA10 (31) |
| 5 | GPIO_7 | PB3 (39) | 17 | GPIO_17 | PB1 (19) |
| 6 | GPIO_8 | PA15 (38) | 18 | GPIO_11 | PA9 (30) |
| 7 | GPIO_5 | PB5 (41) | 19 | GPIO_18 | PB0 (18) |
| 8 | GPIO_6 | PB4 (40) | 20 | GPIO_12 | PA8 (29) |
| 9 | GPIO_3 | PB7 (43) | 21 | GPIO_19 | PA7 (17) |
| 10 | GPIO_4 | PB6 (42) | 22 | GPIO_13 | PB15 (28) |
| 11 | GPIO_1 | PA4 (14) | 23 | GPIO_20 | PA6 (16) |
| 12 | GPIO_2 | PA5 (15) | 24 | GPIO_14 | PB14 (27) |

### DIP switch S1

| S1 pin | Net | Function |
|--------|-----|----------|
| 1–4 | +3V3 | Common |
| 5 | PA0 (10) | Switch input |
| 6 | PA1 (11) | Switch input |
| 7 | PA2 (12) | Switch input |
| 8 | PA3 (13) | Switch input |

Configure PA0–PA3 as **digital inputs with internal pull-down**. Do not configure them as outputs while a switch is closed.

---

## Circuit verification

### Power

```
J1 +5V
 ├── U1  MCP2562  VDD = +5V   (CAN bus side)
 │              VIO = +3V3   (logic side)
 └── U2  TLV75533 LDO  IN/EN = +5V  →  +3V3 (fixed 3.3 V)
        ├── IC2  STM32L433  VBAT(1), VDD_1(24), VDD_2(48), VDDUSB(36), VDDA(9)
        ├── J5 pin 1, J7 pin 1
        ├── S1 commons, SW2, R3/R4/R5
        └── U1 VIO
```

- **U2 TLV75533PDBVR:** SOT-23-5 DBV pinout 1 IN, 2 GND, 3 EN, 4 NC, 5 OUT — matches the PCB. Input 1 µF (C11) + 100 nF (C10); output 1 µF (C12) + 1 µF (C2) + five 100 nF (C1, C3–C6). The datasheet requires ≥1 µF input and 1–200 µF output; this is comfortably inside spec. EN tied to +5 V, so 3.3 V comes up with 5 V.
- **Dissipation:** at the board's ~50 mA load, U2 drops 1.7 V × 50 mA ≈ 85 mW. SOT-23-5 θJA ≈ 60 °C/W → ~5 °C rise. Even at 300 mA it is ~31 °C rise. No thermal concern.
- **Decoupling:** C1/C3/C6 are 1.75–3.2 mm from IC2; C5 is 3.8 mm from the VDDA side. U2's input/output caps are 2.7–2.8 mm away. Good.

### MCU (IC2, STM32L433CCT6, LQFP-48)

- Pinout matches DS11449 (LQFP-48): all VDD/VSS/VDDA/VDDUSB/VBAT pins are connected; PA0–PA3 = S1; PA4–PA7/PB0/PB1/PB3–PB7/PB10/PB11/PB14/PB15/PA8–PA11/PA15 = J5; PA13/PA14 = SWD; PB8/PB9 = CAN; PH0/PH1 = HSE; PH3 = BOOT0.
- **NRST:** C9 100 nF to GND, SW1 to GND. The STM32L433 has an internal ~40 kΩ pull-up on NRST; the datasheet explicitly says the only external component needed is the 100 nF cap. **No external pull-up is required** — the analyzer's `PU-001` is a false positive. Adding a strong external pull-up is not recommended.
- **BOOT0:** R1 10 kΩ to GND, SW2 to +3V3. Press SW2 during reset to enter the bootloader. Correct.
- **HSE:** Würth 830059082, 8.0 MHz, CL = 10 pF, ESR ≤ 80 Ω. C7/C8 = 10 pF each. Effective load is 5 pF + stray (3–5 pF) = 8–10 pF, close to the 10 pF spec. The STM32L4 HSE has gm(min) = 7.5 mA/V; gm_crit for this crystal is ~0.23 mA/V, so the gain margin is ~32× — the oscillator has plenty of startup margin.
- **SWD:** PA13/PA14 to J7, no series resistors. Correct.
- **Unused pins:** PC13–PC15, PB2, PB12, PB13, PA12 are no-connect. Correct.

### CAN (U1, MCP2562-E/SN)

- Pinout matches the datasheet: 1 TXD, 2 VSS, 3 VDD, 4 RXD, 5 VIO, 6 CANL, 7 CANH, 8 STBY.
- VDD = +5 V, VIO = +3V3, STBY = GND (normal mode). Correct for a 3.3 V MCU on a 5 V bus.
- TXD → PB9, RXD → PB8. These are valid CAN1 alternate functions on the STM32L433.
- **Termination:** R2 120 Ω is between CANL and J6 pin 1; J6 pin 2 is CANH. Installing the J6 shunt puts 120 Ω across the bus. **The analyzer's `PR-003` "no 120 Ω termination" is a false positive** — the termination is jumper-selectable by design.
- The CANH/CANL traces are 19.0 mm and 23.7 mm, with 4.7 mm of length mismatch. At the MCP2562's 1 Mbit/s maximum, this is negligible.
- No common-mode choke or TVS on CANH/CANL. Fine for a bench prototype; add TVS/common-mode choke if the bus leaves the bench or runs in a vehicle.

### Switches, LEDs, and connectors

- **SW1** reset: NRST to GND, with C9 100 nF. Correct.
- **SW2** boot: BOOT0 to +3V3, with R1 10 kΩ pull-down. Correct.
- **D1/D2** CAN activity LEDs: cathode to CAN_TX/CAN_RX, anode through 1 kΩ to +3V3. ≈1.3 mA when active. The MCP2562's TXD internal 33 kΩ pull-up and RXD output can handle this.
- **D3** power LED: cathode to GND, anode through 1 kΩ to +3V3. ≈1.3 mA. Visible but dim.
- **J1–J4** Phoenix 1706714 terminal blocks: both pins of each block are on the same net. This is intentional (daisy-chain/current sharing), but see the wiring warning above.
- **J5** 2×12 GPIO header: 20 GPIOs, +3V3, +5V, 2× GND. Correct.
- **J6** 1×2 termination header: correct.
- **J7** 1×4 SWD header: correct, but no NRST. See the bring-up note below.

### Schematic ↔ PCB cross-check

- **34 schematic components = 34 PCB footprints.** Match.
- **All 34 footprints' pad-to-net assignments match the schematic exactly** — including IC2, U1, U2, Y1, S1, J1–J7, D1–D3, R1–R5, and C1–C12. No swapped pins, no library footprint pad-numbering errors.
- **0 unrouted nets**, 382 track segments, 49 through vias, one B.Cu GND zone. GND and +5 V are single connectivity islands; +3V3 is continuous (the analyzer's "3 islands" is a geometry artifact — the +3V3 trace runs straight through S1 pads 1–4 at y = 102.175 mm, so all four are connected).
- **Component count, board size, and net count** were re-checked against the raw files: 42.8 × 46.0 mm, 2-layer, 1.6 mm FR4.

---

## PCB layout notes

### What is good

- 1206 HandSolder footprints for all passives — easy to hand-solder.
- 1.27 mm SOIC-8, 0.95 mm SOT-23-5, 2.54 mm headers/terminal blocks — all easy.
- Decoupling caps are close to the MCU (1.75–3.8 mm).
- GND is a single continuous B.Cu island (20 stitching vias), and +5 V is a single island.
- The crystal's ground pads are 1.6–1.7 mm from GND stitching vias.
- The LQFP-48 fanout uses 28 vias for 23 nets — normal for a 2-layer breakout.
- The board is only 42.8 × 46.0 mm, so traces are short.

### What is not ideal (but not a functional blocker)

| Item | Detail | Impact |
|------|--------|--------|
| **CAN traces under the crystal** | `/CAN_TX` and `/CAN_RX` run on B.Cu at x ≈ 73.5/74.0 mm, y ≈ 92–101 mm — directly under Y1 at (73.75, 96.29). This cuts the B.Cu ground plane beneath the oscillator. | The HSE gain margin is ~32×, so it should start. If you ever see crystal startup or CAN-noise problems, this is the first thing to fix. On a revision, move those two traces or add a ground island under Y1. |
| **U1 VIO bypass** | C10 100 nF is on **+5 V (VDD)**, not +3V3 (VIO). The nearest +3V3 cap to U1 pin 5 is ~17–22 mm away. | VIO current is only 4–85 µA, so it will work. Add a 100 nF from U1 pin 5 to GND on a revision for cleanliness. |
| **2-layer ground plane** | A single B.Cu GND zone is cut by many signal traces (fill ratio 0.473). EMC score 32.5 with 13 errors / 39 warnings, mostly reference-plane gaps and no connector filtering. | Typical for a 2-layer board. Fine for a bench prototype; not a production EMC design. |
| **Via annular ring** | 0.45 mm pad / 0.3 mm drill = 0.075 mm annular ring. | Below IPC Class 2 (0.125 mm), but within JLCPCB's capability (the prior review accepted it). If your fab has a 0.1 mm minimum, they may flag it. |
| **J3/J4 courtyard overlap** | 0.337 mm² overlap = 0.02 mm × 16.85 mm. | The two terminal blocks are designed to butt together; this is intentional adjacency, not a real collision. |
| **J2 edge clearance** | 0.79 mm from the board edge. | The terminal block's wire entry faces the edge; the pads are still inside the fab's edge clearance. Fine. |
| **No mounting holes** | — | The board cannot be screwed down. Use standoffs or a bracket. |
| **No fiducials / test points** | — | Irrelevant for hand assembly. |
| **No revision marking** | — | Add "V3" or a date code if you make another run. |

---

## Hand-assembly notes

- **LQFP-48 (IC2):** 0.5 mm pitch. This is the only hard part. Use plenty of flux, drag-solder with a fine tip, and wick the bridges. Check pins 1–48 for continuity to adjacent pins before powering.
- **SOIC-8 (U1) and SOT-23-5 (U2):** easy with drag soldering. Watch pin 1 orientation.
- **1206 passives:** all the same body size. Sort C7/C8 (10 pF), C2/C12 (1 µF), and the 100 nF caps before soldering — they are visually identical.
- **Resistors:** R1 = 10 kΩ, R2 = 120 Ω, R3–R5 = 1 kΩ. All 1206. Sort them.
- **LEDs:** 0805. Match the cathode marking to pad 1.
- **Crystal Y1:** 7×5 mm, 4-pad. Match pin 1. Keep the area clean; no extra flux residue across the pads.
- **S1:** 1.27 mm pitch SMD DIP switch. Small but hand-solderable. Match pin 1.
- **Terminal blocks J1–J4:** through-hole, 5.08 mm pitch. Solder from the bottom; the wire entry faces the board edge.
- **Headers J5–J7:** through-hole, 2.54 mm. Solder from the bottom.
- **SW1/SW2:** 4-pad SMD tact switches with duplicate pads (1,1 / 2,2). The footprint is correct — pads 1 are one contact, pads 2 are the other. Do not bridge across the middle.

---

## Analyzer false positives / reviewer overrides

| Finding | Why it is not a problem |
|---------|--------------------------|
| `PU-001` — IC2 NRST missing pull-up | The STM32L433 has an internal ~40 kΩ pull-up; DS11449 says only the 100 nF cap is needed. |
| `PR-003` — CAN transceiver has no 120 Ω termination | R2 120 Ω + J6 jumper provide switchable termination. |
| `RS-001` — +5 V has no declared source | +5 V arrives on J1 by design. Add a PWR_FLAG to silence ERC only. |
| `XL-DET` — crystal at 830,059,082 Hz | The analyzer parsed the MPN as the value. The part is 8.0 MHz. |
| `+3V3` connectivity — 3 islands | The +3V3 trace runs through S1 pads 1–4; all four are connected. Geometry artifact. |
| `PM-001` — J3/J4 courtyard overlap | 0.02 mm intentional terminal-block adjacency. |
| `PM-002` — J2 0.79 mm from edge | The terminal block's wire entry faces the edge; copper is inside the fab clearance. |
| `DFM-001/002` — 0.075 mm annular ring | Within JLCPCB's capability; accepted in the prior review. |
| `FD-001` — no fiducials | Hand assembly, not pick-and-place. |
| `TE-001` — no test points | Hand assembly; J5/J7 are the accessible points. |
| `OR-001` — passive orientation | Hand assembly, not pick-and-place. |
| `SS-001` — <50% MPN coverage | Production-sourcing issue, not a functional issue. You are hand-assembling by value/package. |
| `DS-003` — 31/34 parts lack MPN | Same; the values and footprints are specified. |

---

## Prior review delta (2026-10-26 → 2026-11-01)

| Status | Items |
|--------|-------|
| **Fixed / confirmed** | SW1/SW2 pad-pair short; Y1 crystal pin swap; SWCLK reaching PA14; refdes desync; TLV75533 OUT pin type; MCP2562 pin 5 naming (library source); J1–J4 second terminal pads on the same net. |
| **Still open (non-functional)** | ERC `power_pin_not_driven` on +5 V/GND (no PWR_FLAG); `lib_symbol_mismatch` for IC2/Y1/U1/U2 (run *Update Symbols from Library*); stray wire endpoint; J3/J4 0.02 mm courtyard adjacency; silkscreen overlaps. |
| **New in this pass** | CAN_TX/CAN_RX traces run under Y1 on B.Cu; U1 VIO has no nearby +3V3 bypass; terminal blocks have no silkscreen labels; no mounting holes; no reverse-polarity protection; DIP switch relies on firmware pull-downs. |
| **Unchanged** | 3/17 MPN coverage; 0.075 mm via annular ring; 2-layer EMC profile; no fiducials/test points. |

---

## Not performed / review limits

- **Datasheets:** There is no `datasheets/` directory and no extraction cache. I manually checked the STM32L433 (DS11449, NRST internal pull-up, LQFP-48 pinout), MCP2562 (pinout, VIO/VDD, STBY), TLV75533 (SOT-23-5 pinout, 1 µF in/out), and Würth 830059082 (8 MHz, 10 pF load, 80 Ω ESR). The LEDs, switches, terminal blocks, and headers are **inference-only** (standard parts; no manufacturer PDF on file). Their pinouts are simple and match the footprints, but they are not datasheet-verified in this pass.
- **SPICE:** No `ngspice`, `ltspice`, or `xyce` installed. There are no analog subcircuits to simulate (no filters, dividers, op-amps). The crystal and LDO margins were checked analytically instead.
- **Thermal:** `analyze_thermal.py` ran but skipped because no MPNs/datasheet extraction were available. Manual estimate: U2 ~85 mW at 50 mA → ~5 °C rise; U1 ~350 mW worst-case continuous dominant → ~52 °C rise (average is much lower). No thermal issue.
- **Gerbers:** No fabrication outputs present. The `STM32L433CCT6-Prototype-Board.zip` contains only the schematic and the prior review, not Gerbers. Run `kicad-cli pcb export gerbers` before ordering.
- **Lifecycle:** Not audited — no distributor API keys and only 3/17 MPNs. The three ICs and the crystal are active parts; the rest are generic passives/connectors.
- **kicad-cli ERC/DRC:** Not installed on this machine, so I did not rerun native ERC/DRC. The prior review ran them (0 unconnected, 24 DRC violations, mostly cosmetic). The current analyzer confirms 0 unconnected and all pad-to-net assignments match.

---

## If you have not ordered the PCB yet

These are the changes I would make before spending money on fabrication. None are required for the board to function, but they make hand assembly much cleaner:

1. **Add silkscreen labels:** `+5V` and `GND` at J1/J2; `CANH` and `CANL` at J3/J4; `+3V3` at J5 pin 1; `TERM` at J6; `PA0`–`PA3` at S1. This is the single highest-value change for hand assembly.
2. **Add four 10 kΩ pull-downs** from PA0–PA3 to GND, so S1 works without relying on firmware.
3. **Add a 100 nF cap** from U1 pin 5 (VIO) to GND, close to the pin.
4. **Move the CAN_TX/CAN_RX B.Cu traces** out from under Y1, or add a ground island under the crystal. This is the main layout weakness.
5. **Add mounting holes** (e.g., 4× M3 at the corners) if you want to mount the board.
6. **Add reverse-polarity protection** (series Schottky or P-FET) on the 5 V input, and a CAN TVS/common-mode choke if the bus leaves the bench.
7. **Add NRST to J7** (make it a 5-pin header) for easier programming.
8. **Add a revision marking** (`V3`, date code) to the silkscreen.

If the boards are already made, skip these and follow the assembly checklist at the top.

---

## Bottom line

**Build it.** The schematic is electrically correct, the PCB is fully routed with zero unconnected nets, and every footprint's pad-to-net assignment matches the schematic. The board will power up, run the MCU, talk CAN, read the DIP switches, and drive the LEDs.

The only things that will stop it from working are **assembly/wiring mistakes**, not the design:

- Do not plug a 2-pin power cable into J1 — J1 is +5 V on both pins.
- Do not assume S1 works until PA0–PA3 are configured as inputs with pull-downs.
- Install or omit the J6 termination jumper deliberately.
- Check J7's pin order against your ST-Link cable.
- Inspect the LQFP-48 for solder bridges.

If you want the cleanest possible build, add the silkscreen labels and the four DIP-switch pull-downs before ordering. Everything else is refinement.
