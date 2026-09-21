# PDB_V3 Design Review

**Project:** Power Distribution Board V3 (KiCad 8, 6 schematic files / 8 hierarchical sheet instances, 2-layer PCB)
**Date:** 2026-09-17
**Analyzers run:** `analyze_schematic.py`, `analyze_pcb.py --full`, `cross_analysis.py`, `analyze_thermal.py` (40°C ambient), `analyze_emc.py` (emc skill), deep-review pass (15 gated findings, `analysis/deep_review.json`), LCSC lifecycle spot-check
**Analyzers not run:** `analyze_gerbers.py` (no fabrication outputs exist), SPICE simulation (no simulator installed — user declined install), DigiKey/Mouser/element14 lifecycle (no API keys)
**Design intent (user-provided):** Tattu 8000mAh 6S 25C LiPo (18.0–25.2V) via Daly BMS and **XT60 plug**, UAV application. 2-layer stackup intentional. **Rail loads defined 2026-09-17: 12V ch1/ch2 @ 20A each, 24V @ 10A, 5V @ 20A (818W total).** FETs mixed CSD17575Q3 / IAUCN04S7L019.

---

## Critical Findings

| Severity | Issue | Detail |
|----------|-------|--------|
| CRITICAL | **30V FETs on the 24V boost leg** (Q3_3, Q3_2_3, Q4_3) | CSD17575Q3 is 30V-rated (datasheet p.1). SW2 swings to VOut=23.84V → 1.26× DC margin, zero margin for ringing. **Preferred fix — consolidate all four switch positions on IAUCN04S7L019 (already used on the buck legs):** boost leg sees only VOUT=23.84V → 40V/23.84V = **1.68× margin (16.2V ring budget)**, which is *more* margin than the buck leg (25.2V → 1.59×) where the part is already in use; Rds 2.41mΩ@4.5V beats the CSD's 2.6mΩ, Qg 29nC, logic-level, avalanche-tested, 175°C. Bonus: one part number + one footprint for all 23 FETs, eliminating the CSD 5-pad issue. **Gate this on sourcing:** LCSC stock for IAUCN04S7L019ATMA1 is 1 unit (family-wide 52–102/variant) — confirm DigiKey/Mouser reel availability; only PG-TDSON-8-**33** siblings share the footprint. **Fallback (if consolidation sourcing fails):** CSD18540Q5B (60V, 2.2mΩ@10V, VSON-CLIP 5×6, new footprint) — DigiKey mid-Oct lead time, LCSC 15k. In-stock 60V alternates: CSD18563Q5A (TI, 60V, 6.8mΩ, LCSC 2,319), AON6262E (AOS, 60V, 6.2mΩ, LCSC 1,878). Rejected: SIR158DP (Vishay) — 30V-rated, not 60V. |
| CRITICAL | **Shunt values wrong for the defined loads** (Rsense1–4) | Targets at 12V/5V @20A and 24V @10A: **3.2mΩ (12V ch), 3.2mΩ (5V), 6.4mΩ (24V)** in 2W+ wide-terminal 4-terminal packages. Installed 1.8/3.6/5mΩ 2-terminal 1206s: limits sit 60–100% above the loads (no protection) and dissipation at load is 0.7–0.9W (over 1206 class). |
| CRITICAL | **Input side must carry 50A: 39.6A @22.2V, 48.9A @18V** | 818W total. Daly BMS continuous rating must be ≥50A; user's XT60 runs at 82% of rating (PCB already has an XT90 footprint — prefer XT90/XT90-S anti-spark); VCC copper must be plane-class. |
| CRITICAL | **No ceramic capacitors in the buck hot loops** | Per-channel input bulk is 410µF + 100µF electrolytic only. At 20A out, input caps see ~10A RMS ripple — only ceramics can serve this. *(Newer layout revision with <25mm² loops reported by designer but not pushed to this repo — the layout half is reportedly fixed; the capacitor-type half is still open.)* |
| WARNING | 1µH inductors wrong for loads — targets **3.6µH (12V ch), 1.5µH (5V), 2.4µH (24V)** for 40% ripple; installed values give 57–125% ripple; verify WE-HCM 1411 Isat ≥ Ipk (24.4/25.7/16.5A) | |
| WARNING | No reverse-polarity device / main fuse after XT90 input; Df1–4 TVS are valueless generic "D_Zener" | |
| WARNING | UVLO at 15.35V vs 6S empty 18V → 2.65V margin; load sag can nuisance-trip all rails | |
| WARNING | 22 of 23 FETs have **zero thermal vias** on 2-layer 1oz copper | |
| WARNING | VCC input pour split into 9 islands; VCC net contains 0.45mm segments (~1.4A class); 5mm spine ≈ 11A @10°C rise — inconsistent with shunt-implied currents | |
| WARNING | 0/178 nets have test points; no fiducials; J2 courtyard overhangs board edge 0.25mm; annular ring 0.075mm below advanced fab minimum | |

**Verdict: NOT ready for fabrication.** Schematic is close to TI's recipe (pin maps verified 1:1), but four load-bearing corrections are needed: 60V FETs on the 24V boost leg, corrected Rsense/L values for the defined loads, ceramic hot-loop input caps, and 50A-class input path (BMS/connector/copper). Push the newer layout revision and re-review.

---

## Overview

Battery (6S via Daly BMS, XT60 plug) enters at J2 (XT90PW-M footprint — **connector mismatch with the XT60 plug, verify intent**) onto a rail named `VCC`, feeding four LM5176-Q1 4-switch buck-boost channels: 2× 12V (U1, U4, 220.8kHz), 1× 24V (U3, 23.84V, 340.2kHz), 1× 5V (U2, 340.2kHz). Each channel has 4 switch positions (buck leg = IAUCN04S7L019 40V; boost leg = CSD17575Q3 30V; parallel pairs on high-side positions except the 5V channel's buck HS). Each rail distributes through 8× (blade fuse → SPST switch) branches to 16-pin terminal blocks (32 fused/switched outputs total). 207.5×137mm, 2-layer, 1oz/1oz, 1.6mm.

## Component Summary

| Type | Count | | Type | Count |
|---|---|---|---|---|
| Resistors | 52 | | Inductors | 4 |
| Capacitors | 60 | | Diodes | 12 |
| ICs (LM5176) | 4 | | Transistors (NMOS) | 23 (11 IAUCN + 12 CSD) |
| Fuses | 32 | | Switches | 32 |
| Connectors | 5 | | Mounting holes | 4 |

228 schematic components, 180 nets, 947 wires. PCB: 230 footprints (228 + 2 logo decals `G***` — benign), 489 vias, 26 zones, routing complete, 82 THT / 143 SMD.

**Sourcing (SS-001, 0% MPN coverage):** no component carries an MPN property. Part identity is only inferable from footprints. Fuse values ("Fuse"), zener/boot diode values ("D_Zener") are unspecified — **unfabricatable BOM lines**.

## Power Tree

```
6S LiPo (18.0–25.2V) -> Daly BMS -> XT60 plug -> [J2: XT90PW-M footprint]
  |  (no reverse-polarity device, no main fuse, Df1-4 TVS valueless)
  +-- VCC rail (9 pour islands)
  |-- U1  LM5176 12.0V  220.8kHz  Rsense 1.8m  L1 1uH  -> 8x fuse->SW branches (J1)
  |-- U2  LM5176  5.0V  340.2kHz  Rsense 3.6m  L2 1uH  -> 8x branches (J3)
  |-- U3  LM5176 23.84V 340.2kHz  Rsense 5m    L3 1uH  -> 8x branches (J4)
  +-- U4  LM5176 12.0V  220.8kHz  Rsense 1.8m  L4 1uH  -> 8x branches (J5)
       UVLO all channels: 15.35V rising (249k/21.5k, EN=1.22V)
       MODE: 93.1k -> CCM + hiccup (matches datasheet exactly)
       BIAS: U1/U4 to VOut; U2/U3 floating w/ 100nF only
```

FB dividers (raw-file verified): Rfbt/Rfbb = 280k/20k → 12.00V, 105k/20k → 5.00V, 576k/20k → 23.84V (Vref=0.8V; Rfbb=20k matches TI's own example). Full computations in `analysis/helpers/check_lm5176_design.py`.

## Analyzer Verification

### Component Count — 228/228 match
Schematic 228 (excl. power symbols) vs PCB 230 = 228 + 2 `G***` logo footprints. No missing/extra components.

### Component Pinout Verification

| Ref | Value | Verification | Status |
|---|---|---|---|
| U1–U4 | LM5176QPWPRQ1 | HTSSOP-28 pin map vs TI SNVSB46B Table 5-1 (p.3–4) — all 29 pins, all 4 ICs, pin-for-pin | **Verified (datasheet)** |
| Q1_*/Q2_* (11) | IAUCN04S7L019 | 40V/1.92mΩ confirmed (datasheet p.1); symbol 1=D,2=G,3=S vs footprint pads — electrically consistent (PCB pads 1=VCC/SW, 2=gate-drive, 3=SW/RCS) but **physical G/S pad position unverified** (package drawing is image-only; datasheet Rev 1.0 "in development") | **Unverified — manual check required** |
| Q3_*/Q4_* (12) | CSD17575Q3 | Pin functions from datasheet p.1/p.4 (1,2,3=S; 4=G; 5–8+tab=D); **5-pad structure verified by designer 2026-09-17** (4 top pins merged into big pad + 4 individual bottom pads). Residual: footprint pads 4/5 carry no net — two source lands unbonded | **Verified (user)** — action: net pads 4/5 to Source |
| J1/J3/J4/J5 | "1862262" | Symbol has 8 power+8 GND pin pairs (16-pos block); Phoenix 1862262 is a 2-position part per its own numbering — **footprint/MPN mismatch likely** (Dependences contains 1862550 — verify actual ordering MPN) | **Unverified** |
| J2 | XT90PW-M | Pad 1=GND, 2=VCC — correct polarity; user's plug is XT60 (physically incompatible — adapter or connector change needed) | Verified (raw file); intent mismatch flagged |
| F1–32, SW1–32 | "Fuse"/"SW_SPST" | Series order rail→fuse→switch→terminal is correct practice; ratings unspecified | Skipped (values undefined) |
| Rsense1–4 | 1.8m/3.6m/5m | Pads: RCS_x / CSG_x — series path Q2+Q4 sources → Rsense → CSG→GND; topology correct, part class wrong (see Critical) | Verified (topology); part wrong class |
| L1–L4 | 1uH | WE-HCM 1411 footprint; datasheet unobtainable — Isat unknown | Unverified |
| Passives (R/C, 2-pin) | — | — | Skipped |

### Net Tracing
- `VCC`: J2.2 → all Cin/Cbulk/Ruvt/Rvisns/Q1-drains/Df — traced end-to-end, no missing members. Named "VCC" — generic name for a 25V battery rail confuses every automated tool (rail voltage unparseable); rename to e.g. `VBATT`.
- Output rails named `/uuid/VOut` and `VOSNS_2/3` — UUID-path net names from unlabeled hierarchical connections; rename to `+12V_CH1` etc.
- Sheet names contain spaces ("lm5176 12v Channel 1") → net names with spaces — tooling-hostile; prefer underscores.
- PP-001 ×26 / RS-001 ×7 triage: switch-node/BOOT/VCC-LDO pins have no DC rail path by topology (expected for a controller); the "VOut has no declared source" and "VCC has no declared source" findings are ERC-hygiene (add PWR_FLAGs), not functional defects.

### PCB Verification
Footprint count, positions, and pad-nets spot-verified for U1, Q1_1, Q2_1, Q3_1, Q4_1, Rsense1, J1, J2 — all match schematic pin-to-net assignments (no symbol↔footprint pad numbering errors found at the logical level; the residual risk is physical package geometry, above).

## Deep Review (gated: 15 findings, 0 quarantined)

Full evidence with datasheet quotes and computations: `analysis/deep_review.json`. Highlights beyond the Critical table:

- **Slope compensation**: 12V channels exact (220pF installed vs 222pF derived); 5V/24V over-sloped (150pF vs 111/80pF) — stable but reduced current-limit accuracy (info).
- **Soft-start**: 22nF → 3.5ms charging 2mF output banks → ~6.8A precharge per 12V rail; four channels energize together against the BMS (info; consider 100nF / staggered EN).
- **Boot diodes**: Dboot1_x/Dboot2_x correctly oriented VCC→BOOT per U1 pad nets; datasheet requires *Schottky* (p.19) — generic "D_Zener" values must become a real Schottky part (warning).
- **BIAS inconsistency**: U2/U3 floating (U3 = 24V rail loses the >8V BIAS efficiency benefit); U1/U4 connected (info).
- **Buck-leg 40V FETs**: 1.59× DC margin at 25.2V; avalanche-rated (EAS 55mJ) but tight for SW ringing at 340kHz (warning; snubber or 60V part).
- **IAUCN04S7L019ATMA1 sourcing**: LCSC stock = 1 unit; 12 used (warning; DigiKey/Mouser unverified — no keys).

## Signal Analysis Review

- **Power regulators detector: zero detections.** The LM5176 is a controller (no power_out pin); the analyzer's pattern library doesn't classify it. All regulator analysis above is manual, datasheet-driven (deep review) — treat the analyzer's silence as a coverage gap, not a pass.
- Voltage dividers (UVLO, FB, VOSNS): all values recomputed and consistent (see Power Tree).
- RC filters: Rcsp/Rcsg = 100Ω/100Ω CS-line filter sits exactly at the datasheet's "filter resistance should not exceed 100 Ω" limit (p.24) — acceptable, note the CSG pin description says "connect directly"; the matched-pair filtering chosen here is a defensible variant.
- Current sense: shunt topology correct (single RCS in the Q2+Q4 common source); ISNS± tied to GND = "short together to disable" per datasheet — valid.
- Crystal/opamp/LED/bus detections: none applicable (pure power board).

## Power Analysis

- **Power budget (loads defined 2026-09-17)**: 818W out → 880W in → **39.6A @22.2V / 48.9A @18V input**. Per-channel input: 11.6/11.6/11.5/4.8A nominal. Input-side checklist: BMS ≥50A, connector choice, VCC plane-class copper, J2 joints, input fuse sizing. Output side is within each LM5176's capability at the corrected Rsense/L values below.
- **Corrected design points** (from `analysis/helpers/check_lm5176_design.py`):

| Channel | Load | L target | Rsense target | Shunt P | Ipk (L+ΔI/2) |
|---|---|---|---|---|---|
| 12V ch1/ch2 | 20A | 3.6µH (1µH installed) | 3.2mΩ (1.8mΩ installed) | 1.28W → 2W+ part | 24.4A |
| 5V | 20A | 1.5µH (1µH) | 3.2mΩ (3.6mΩ) | 1.28W → 2W+ | 25.7A |
| 24V | 10A | 2.4µH (1µH) | 6.4mΩ (5mΩ) | 1.12W → 2W+ | 16.5A (IL 13.2A @18V in) |

- **Inrush**: 4× (410µF+100µF) input bulk + hot-plug (no anti-spark; XT90-S variant has it) → hard inrush at plug-in; Daly BMS OCP behavior unverified. Output side: 2mF×2 banks at 3.5ms SS → ~6.8A each.
- **PDN**: VCC islands + B.Cu nearly pour-free (1 GND zone) — at 50A-class input this is mandatory to fix: unify VCC pour, pour B.Cu GND, stitch.

## Correction Guide (2026-09-17 addendum — concrete fixes for the shunt/inductor blocker)

### 1. Rsense (all four channels)
Use **standard-value 4-terminal / wide-terminal shunts (2512-class, ≥2W)** — not 2-terminal 1206 chip resistors:

| Channel | Rsense | Buck valley limit | Boost peak limit | P at load | Notes |
|---|---|---|---|---|---|
| 12V ch1/ch2 (20A) | **3mΩ** | 26.7A (1.33× load) | n/a (buck-only rail) | 1.2W | — |
| 5V (20A) | **3mΩ** | 26.7A | n/a | 1.2W | — |
| 24V (10A) | **6mΩ** | 13.3A | 20A (Ipk 16.5A → 1.21×) | 1.05W | — |

Part class examples: Vishay WSLP / Stackpole CSR / Bourns CRA 4-terminal wide-terminal series — verify the exact part's rating and use **its** footprint (4 pads: 2 current + 2 Kelvin sense). **Kelvin routing:** the Rcsp/Rcsg (100Ω) sense lines must land on the shunt's *sense* pads, never on the current-carrying pads; route them as a pair away from the SW nodes.

### 2. Inductors (L1–L4)
| Channel | L (std value) | ΔI ripple | Ipk | Isat requirement |
|---|---|---|---|---|
| 12V ch1/ch2 | **3.3µH** | 8.6App (43%) | 24.3A | ≥25A |
| 5V | **1.5µH** | 7.9App (39%) | 23.9A | ≥24A |
| 24V | **2.2µH** | 5.9App (45%) | 16.1A | ≥17A |

Select the WE-HCM 1411 variant whose datasheet Isat/Ir meets the requirement **at the chosen inductance** (Isat drops as L rises — the current 1µH value's Isat cannot be assumed for 3.3µH). Fetch the Würth datasheet for the exact 744-part-number before ordering.

### 3. Retune after the value changes
- **Cslope** = 0.4×1000×L(µH)/Rsense(mΩ): 12V ch → **440pF**; 5V → **200pF**; 24V → **150pF** (current 24V value coincidentally correct).
- **Css**: consider 100nF class (~16ms) on the 12V channels to soften the 2mF output-bank charge (currently ~6.8A in 3.5ms).
- Recompute FB/UVLO: unchanged (correct as-is).

### 4. Ceramic hot-loop input caps (all four channels) — placement
The buck commutation loop is `VCC copper → Q1 → SW1 → Q2 → Rsense → PGND copper`. Place the ceramics **across the ends of that loop, next to the FETs**:

```
        ┌──────────── hot loop (<25mm²) ────────────┐
 VCC ═══╪══ Q1(drain)                       Q2(source) ══╗
        │      \\ SW1 ── L ── SW2 //                      ║ PGND
        ╚══════════════ C_ceramic bank HERE ═════════════╝
              (2–4× 4.7–10µF 50V X7R + 1× 100nF 50V)
```

- 2–4× **4.7–10µF, 50V, X7R** (1206/1210) + 1× **100nF, 50V**, on F.Cu, immediately adjacent to the Q1 drain pad and Q2 source pad — same <25mm² loop discipline applied to the main loop. The 100nF goes closest.
- **50V rating is mandatory** (not 25V): VCC reaches 25.2V charged, and MLCC capacitance collapses ~80% at DC bias near rating.
- The 410µF/100µF electrolytics stay back at the channel input as bulk — their ~15nH ESL is invisible at 220–340kHz edges, which is exactly why they cannot serve this loop.
- **24V channel extra**: in boost mode the hot loop moves to the output side (Cout → Q3 → SW2 → Q4 → PGND). Ensure the Coutx 70µF ceramic (or an added 10µF 50V) sits next to the **Q3 drain pad on VOut**, not just near the output terminal.

## PCB Layout Analysis

- **Stackup**: 2-layer, 1oz/1oz, 1.6mm — intentional per user. Consequences below are accepted trade-offs, but several exceed even 2-layer norms.
- **Thermal**: LM5176 DAPs: **34 vias each vs 16 min — good**. FETs: 0 vias on 22/23 (Q4_2: 1) — the dominant thermal risk on this board. Thermal analyzer @40°C: Rsense1/4 → 596°C, Rsense2 → 318°C, Rsense3 → 240°C est. (at assumed limit currents — real numbers depend on defined loads, but no plausible load makes a 1206 shunt viable at these limits).
- **Current capacity (IPC-2221, 1oz external)**: 0.45mm ≈ 1.4A, 5.0mm ≈ 11A @10°C rise. VCC has 30×0.45mm + 8×5mm segments across 9 pour islands. GND return likewise fragmented (1 B.Cu zone; 12 F.Cu GND zones).
- **DFM**: tier "challenging": min annular ring 0.075mm < 0.1mm advanced minimum (error); board 207.5×137mm (pricing tier, info); J2 courtyard overhangs edge 0.25mm (fix); no fiducials on 143 SMD (add 3); via edge clearance and tenting clean.
- **Testability**: 0 test points / 178 nets. For bring-up of a 4-channel power board, minimum set: each VOut, VCC, 4× SW1/SW2, 4× RCS, PGOODs.
- **Placement**: courtyard overlaps only from auto-placement rule areas (dismissed, see False Positives); CP-002 ×88 same-layer GND-zone-under-component = normal pour behavior.

## EMC Pre-Compliance (CISPR 32 baseline)

133 findings, trust level *low* (heuristic-heavy) — read as a risk map, not violations:
- GP-001 ×68 (reference-plane gaps) and GP-004 ×3: direct consequence of 2-layer/no-B.Cu-pour + fragmented VCC/GND islands — **pour-fill B.Cu with GND and stitch**; this single change addresses the largest EMC risk class.
- SW-001/002/003 (switching): large hot loops (no ceramic input caps) — fix the hot-loop decoupling above; SW-003 (hot loop) resolves with it.
- DC-002 ×4 (no decoupling near U4): check Cvcc/Cf placement distances on U4's sheet instance.
- IO-001 ×4 (unfiltered I/O at terminals): power terminals of this class normally don't need filtering — triaged as low-priority for a UAV PDB, revisit if radio interference observed.

## Thermal Hotspots
Covered above (Rsense criticals; FET via absence; 40°C ambient assumption stated; total est. dissipation 15.9W at assumed operating points — invalid until loads defined).

## Standards Compliance
- All working voltages ≤25.2V — IPC-2221 spacing satisfied by 0.2mm min clearance; creepage/clearance N/A (no mains).
- Conductor current capacity: see PCB section — deficient at implied currents, fine at ≤5–10A/rail with loads defined accordingly.
- Annular ring below IPC Class 2 min (0.125mm) and fab advanced min — fix via pad sizes or larger drill.

## Schematic ↔ PCB Cross-Reference
- Component count: 228↔230 (2 logo decals) — reconciled.
- Pin-net verification: logical level verified for all ICs/transistors/connectors (see Analyzer Verification).
- Net name sync: PCB nets carry sheet-path prefixes with spaces; same design uses three naming styles across channels (path-prefixed, global-suffixed, UUID) — rename for sanity before revision 4.
- DNP: none; no routing on logos.

## Interface Summary

| Interface | Connector | Protection | Notes |
|---|---|---|---|
| Battery in | J2 XT90PW-M (user has XT60) | None (BMS upstream); Df TVS valueless | No main fuse / reverse block |
| 12V#1 out ×8 | J1 (16-pos terminal) | Per-branch blade fuse + switch | Fuse ratings unspecified |
| 5V out ×8 | J3 | same | |
| 24V out ×8 | J4 | same | 30V FETs on this rail's boost leg |
| 12V#2 out ×8 | J5 | same | |

## Quality & Manufacturing

- **Sourcing audit**: 0% MPN coverage (SS-001) — every component. Minimum pre-order action: MPN + ratings for Fuses, D_Zener positions (Df TVS + Dboot Schottky), terminal blocks (verify 16-pos part number vs "1862262"), shunts (4-terminal 2W+), inductors (exact WE-HCM MPN + Isat).
- **Lifecycle (partial — LCSC only)**: LM5176QPWPRQ1 (309), CSD17575Q3 (7655) healthy; IAUCN04S7L019ATMA1 stock 1 — sourcing risk.
- **Ordering notes**: 2-layer, 1.6mm, 1oz — consider 2oz copper given power intent (cheap upgrade that directly helps every thermal/current finding); DFM tier "challenging" → fix annular ring before quoting; stencil required (143 SMD); JLCPCB standard max size exceeded (207×137mm fine, pricing tier note).
- **BOM optimization**: good consistency across channels (shared values); 5V/24V slope-cap mismatch is the only per-channel drift.

## All Issues & Suggestions (full list)

| Severity | Issue | Detail |
|---|---|---|
| CRITICAL | 30V FETs on 24V boost leg | Q3_3/Q3_2_3/Q4_3 → ≥60V part |
| CRITICAL | Shunts wrong for loads + 1206 class | 3.2mΩ (12V/5V ch), 6.4mΩ (24V), 2W+ 4-terminal wide-terminal shunts; 1.28/1.12W at load |
| CRITICAL | FET footprint: CSD pads 4/5 carry no net (orientation designer-verified) | Map pads 4/5 to Source in symbol or merge source pads; re-run DRC |
| CRITICAL | No ceramic hot-loop input caps | Add 2–4× 1–10µF X7R + 100nF at each Q1/Q2 loop (~10A RMS input ripple per 12V channel). Newer off-repo layout reportedly fixed loop *area*; capacitor *type* still required |
| CRITICAL | Input side must carry 50A (39.6A nom / 48.9A @18V) | BMS ≥50A; prefer XT90 over XT60; VCC plane copper; 60A-class input fuse if added |
| WARNING | 1µH inductors wrong for loads (targets 3.6/1.5/2.4µH) | Retarget per Eq.13; verify WE-HCM 1411 Isat ≥ 24.4/25.7/16.5A |
| WARNING | No reverse polarity / main input fuse / TVS valueless | Add protection; specify real TVS (SMBJ-class for 6S) |
| WARNING | UVLO 15.35V, 2.65V margin at empty 6S | Lower threshold or add hysteresis |
| WARNING | 22/23 FETs zero thermal vias | Via arrays into opposite-layer pour; consider 2oz Cu |
| WARNING | VCC 9 islands, thin segments | Unify pour, plane-class spine |
| WARNING | Dboot "D_Zener" must be Schottky | Specify part (20–40V, ≥500mA) |
| WARNING | IAUCN single-point sourcing (stock 1) | Qualify second source |
| WARNING | BIAS floating on U2/U3 | Connect U3 BIAS to 24V rail |
| WARNING | Buck-leg 40V margin 1.59× | Snubber or 60V FET |
| WARNING | 0 test points, no fiducials, J2 edge overhang, annular ring 0.075mm | Bring-up + fab checklist |
| SUGGESTION | 0% MPN coverage; fuse/zener values blank | Populate BOM properties |
| SUGGESTION | Net names: UUID paths, spaces, generic "VCC" | Rename rails (`VBATT`, `+12V_CH1`, …) |
| SUGGESTION | XT90 footprint vs XT60 plug | Reconcile connector choice (XT90-S anti-spark?) |
| SUGGESTION | Slope caps 5V/24V over-spec'd; SS 22nF with 2mF banks | Recompute after Rsense fix |
| SUGGESTION | 5V channel has single buck-HS FET (pair everywhere else) | Confirm intent |
| SUGGESTION | lm5176-converter.kicad_sch orphan template | Archive/delete to avoid future confusion |

## Positive Findings

1. **LM5176 symbol pin map verified 1:1 against the TI datasheet for all four ICs** (29 pins × 4) — the custom symbol is correct.
2. FB dividers produce exact rail voltages (12.00/5.00/23.84V) with Rfbb = TI's own recommended 20k.
3. MODE (93.1k = datasheet CCM+hiccup value), DITH, SLOPE (12V channels exact to within 1%), RT, SS, CVCC (1µF per datasheet 1–4.7µF), boot caps (100nF per 0.1–0.22µF) all match the datasheet recipe.
4. Boot diode orientation VCC→BOOT correct (anode/cathode verified at pad level).
5. LM5176 thermal design: 34 DAP vias per IC (2× the minimum).
6. Branch protection order rail→fuse→switch→terminal is correct practice on all 32 branches.
7. Consistent per-channel design (copy discipline) — deviations found are localized and enumerable.

## False Positives / Reviewer Overrides

1. **KO-001 ×50 (keepout violations)** — all 50 are footprints inside `auto-placement-area-<sheet>` rule areas (KiCad auto-placement confinements, no copper keepout rules set). Dismissed.
2. **PP-001 ×26** — switch nodes/BOOT/VCC-LDO pins have no DC rail path by 4-switch-buck-boost topology. Dismissed (except as PWR_FLAG hygiene below).
3. **RS-001 ×7** — controller-sourced rails have no power_out pin: add PWR_FLAGs on VCC and VOut rails (ERC hygiene, not functional).
4. **CP-002 ×88** — components over GND pour with GND pads: expected. Dismissed.
5. **EMC IO-001 ×4** — unfiltered "I/O" at power terminals: low relevance for a UAV PDB. Downgraded.
6. **GP-001 (majority)** — inherent to the intentional 2-layer choice; the actionable subset is B.Cu pour + island stitching, already listed.

## Analyzer Gaps

1. Regulator detector does not recognize LM5176-class controllers — zero detections; all regulator findings here are manual.
2. Thermal analyzer lacks FET conduction-loss modeling (only 4 shunt components assessed); FET/inductor temps must be re-estimated once loads are defined.
3. Current-capacity analyzer emitted info-level only — no load model to compare against.
4. Analyzer could not parse rail voltages from names like `VCC`/`VOut`/UUID paths — net-name hygiene would improve automated review quality.
5. Image-only datasheets (IAUCN Rev 1.0, package pages) blocked automated pinout verification; this model cannot read images in this environment.

## Not Performed / Review Limits

- **Newer layout revision**: the designer reports a hot-loop-reduced layout (loops <25mm²) on another machine, not pushed to this repository. This review covers the pushed revision only — the newer layout must be pushed and re-reviewed before fab. The ceramic-capacitor half of the hot-loop finding is schematic-level and open regardless of layout.
- **Datasheets**: LM5176-Q1, CSD17575Q3, IAUCN04S7L019 on disk (`datasheets/`); **missing**: WE-HCM 1411 (exact MPN unknown), Phoenix 1862262/1862550, E-Switch 1101A4CQEA, XT90PW-M, 3557-2 holders — all connector/mechanical; ratings cited from generic knowledge are marked unverified.
- **SPICE simulation** not run — no simulator installed; user declined install. Divider/filter/compensation values were hand-computed instead (helper script).
- **Gerber analysis** — no fabrication outputs exist.
- **Lifecycle** — LCSC only (no DigiKey/Mouser/element14 keys); `--lifecycle` full audit not run.
- **Prior review delta** — no prior review files or analyzer runs exist in the project.
- **FET physical pinout** (both custom footprints) — manual footprint-editor verification required (see Critical #3); this is the single highest-consequence unverified item.
- **Zone-fill currency**: KiCad lock files present (board may be open in editor); analysis reflects the last saved state — re-run `B` (fill all zones) and re-export before fab.
