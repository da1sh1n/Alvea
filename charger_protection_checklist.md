# Charger + Protection Fix — Checklist (IP2312U)

## Context

Your requirements: switching charger (low heat), charge current **1.3–1.6 A** (goal 1.5 A), charge status known to the firmware, a hand-reworkable, cheap package, and **as few unique BOM parts as possible** (standard assembly charges per unique part).

Choice: **IP2312U_VSET** (LCSC C605432, ESOP-8).

- ICHG uses **two footprints on one net, gap-selectable** (only one bridged at a time): **96.5 kΩ 0.1%** (primary, JLC-assembled) gives 1.40 A nominal, 1.54 A worst case; **102 kΩ 0.1%** (backup) gives 1.32 A nominal, 1.46 A worst case (chip ±10% plus resistor 0.1%). Bridge exactly one gap before first power-up — ICHG open/>170 kΩ falls back to the chip's 2.1 A default. Default to 96.5 kΩ; move to the 102 kΩ jumper on any board whose measured charge current runs high enough to risk the fuse.
- A **TVS diode (SMF5.0A, 5 V unidirectional, SOD-123FL)** sits across VBUS to GND at the USB-C connector, before the polyfuse, for ESD/static protection. Cathode to VBUS, anode to GND with a short direct trace. Does not replace the input snubber (that's for plug-in ringing, not ESD).
- Charge status comes from the **MAX17048 fuel gauge** over I²C (CRATE / state of charge), plus **VBUS sense on module pad 27** (already wired) to tell "full" from "unplugged". The firmware polls, so ALRT isn't needed for now — but R4 (10 kΩ pull-up) and the `BAT_ALTR` net to module pad 18 are being **kept wired, just unused**, in case an interrupt-driven approach is added later.
- Polyfuse: already ordered, **1.5 A hold**. The dual ICHG jumper exists specifically to keep worst-case charge current under this fuse's hold rating on boards that measure high.
- Unique parts vs today: **+7** (IP2312U, 96.5 kΩ, 102 kΩ, 1 µH, 0.5 Ω, 22 µF, TVS added; BL8573 and R11/680 Ω removed; 1 kΩ TEST/D2/U6-VCC unchanged from earlier plan). 10 kΩ and 100 nF are reused from the existing BOM; 75.3 kΩ is no longer used in this circuit (check the rest of the board before dropping it from stock). 680 Ω is no longer used anywhere in this circuit — check the rest of the board before dropping it from stock.
- TEST and D2 use **1 kΩ exactly as the datasheet shows**. The datasheet gives no reason, so it isn't safe to change.
- IP2312U draws 30–50 µA from the battery when USB is unplugged (about 1% of 3500 mAh per month). Accepted.

Order per board: **schematic → place (remove old, drop in new) → route.** Right half first, then the mirrored left half. Time (your estimate): about 30 min of schematic + about 5 h of PCB.

---

## 1. Before you start (parts)

- [x] Confirmed in the IP2312U datasheet ([C605432](https://wmsc.lcsc.com/wmsc/upload/file/pdf/v2/lcsc/2009081935_INJOINIC-IP2312U-VSET_C605432.pdf)):
  - [x] no resistor on D1 = 4.2 V charge voltage (p.7)
  - [x] ICHG = 135000 / R, ±10%; R above 170 kΩ or open = 2.1 A default (p.7)
  - [x] one-LED mode wiring: D1 LED + 100 Ω, D2 via 1 kΩ to BAT+ (p.9); TEST via 1 kΩ to BAT+ (p.2)
  - [x] NTC: 20 µA source, normal window 0.60–1.32 V, must not float (p.8)
  - [x] VIN absolute max 9 V (< 10 µs) / 6.5 V (> 10 µs) (p.3)
- [x] Check LCSC stock for IP2312U_VSET (C605432). **Not C605433** (that one is 4.35 V). - C605432
- [x] **96.5 kΩ 0.1%** ICHG primary, same TNPW-series style as before — **C852962**
- [x] **102 kΩ 0.1%** ICHG backup, same net as primary via gap-selectable jumper — **C852473**
- [x] Pick a 1 kΩ 0402 1% (e.g. JLC basic C11702) - C11702
- [x] Pick an inductor: **Bao Cheng (BC) 1 µH, 7 A saturation / 4.5 A rated, 27 mΩ DCR, ±20%, 4.2×4.4 mm — LCSC C2921163, JLC-assembled**
- [x] Pick a 22 µF 0805 X5R ≥ 10 V - **C296720**
- [x] Pick a 0.5 Ω resistor 0402 — **C423160**. Used singly as R1 at BAT (matches datasheet exactly, no deviation) and as **two in series (~1 Ω total)** for the VIN snubber — avoids stocking a separate 1 Ω part
- [x] Polyfuse: **already ordered, 1.5 A hold**, 8 V, 0805/1206 — place near USB-C connector, physically separated from the charger/inductor cluster (copper gap + a few GND vias on the charger side)
- [x] Pick a TVS diode: **SMF5.0A, 5 V unidirectional, SOD-123FL** — LCSC C2990427 (check stock before ordering; C336129 and C2761010 are alternates of the same part)
- [x] Import the IP2312U footprint and symbol with easyeda2kicad (thermal pad 2.09 × 2.09 mm)
- [x] Import/confirm footprint for the SMF5.0A TVS (datasheet pad layout, not generic)

## 2. Right schematic (`kicad/right.kicad_sch`)

### Remove

- [x] U5 (BL8573)
- [x] R11 (680 Ω) — removed; not reused anywhere in this charger circuit anymore (no LED on D1/D2; U6 VDD and VM now use 1 kΩ). Check whether 680 Ω is used elsewhere on the full board before dropping it from your parts stock entirely
- [x] R15, R18 (10 kΩ CHRG/STDBY pull-ups)
- [x] CHRG and STDBY nets (module pads 41 and 43 become unconnected)

### Add charger (IP2312U_VSET, datasheet p.9)

- [x] Pin 8 VIN → VBUS, with 2 × 22 µF to GND (10 V, X5R, 0805)
- [x] Input snubber: **two 0.5 Ω in series (~1 Ω) + 22 µF in series**, VBUS → GND (damps the plug-in voltage spike; RC ≈ 22 µs vs. the datasheet's own R2/C6 = 2 Ω/10 µF = 20 µs — close enough, and reuses the same 0.5 Ω part as R1 instead of adding a new value)
- [x] Pin 7 SW → 1 µH inductor → BAT+
- [x] Pin 5 BAT → the node between a single **0.5 Ω** (R1 in the datasheet) and **22 µF** to GND (C3); the other end of the 0.5 Ω goes to BAT+. Pin 5 is **not** tied directly to BAT+. Plus **22 µF** from BAT+ to GND (C4)
- [x] Pin 6 ICHG → gap-selectable jumper → **96.5 kΩ 0.1%** (primary) or **102 kΩ 0.1%** (backup) → GND. Only one gap bridged at a time; both open = chip falls back to 2.1 A default (do not leave unbridged). Silkscreen-label each branch (e.g. "1.40A" / "1.32A")
- [x] Pin 4 NTC → **2 × 102 kΩ in parallel = 51.0 kΩ** → GND → 1.02 V at 20 µA (datasheet p.8: no NTC needed → 51 kΩ to GND; the 82 kΩ ∥ 100 kΩ figure there is only the example for a real 100 kΩ thermistor at 25 °C). Existing 102 kΩ part, no new value. Never leave NTC floating. Battery temperature check is off (no thermistor).
- [x] Pin 2 TEST → **1 kΩ** → BAT+ (as datasheet)
- [x] Pin 1 D1 → **leave unconnected** (no LED, no resistor). D1 is multiplexed with battery-type select, so open = 4.2 V; **no resistor from D1 to GND** (43 kΩ = 4.3 V, 75 kΩ = 4.35 V, 100 kΩ = 4.4 V).
- [x] Pin 3 D2 → **1 kΩ** → BAT+ (datasheet one-LED wiring; no LED on either pin. The datasheet does not cover a no-LED setup, so this keeps a documented state; leaving D2 open is the alternative)
- [x] EPAD → GND
- [x] F1 → 1.5 A-hold polyfuse (already ordered part; unchanged from earlier plan)
- [x] Add TVS (SMF5.0A): cathode → VBUS at the USB-C connector (before F1), anode → GND, short direct trace
- [x] Keep VBUS → module pad 27 as is (firmware uses it)

### Fix protection (U6 DW01A, Q2 FS8205A)

- [x] Create net `BAT-`
- [x] Move U4 pin 2 (battery −) from GND to `BAT-`
- [x] Move U6 pin 6 (VSS) from GND to `BAT-`
- [x] Move Q2 pin 3 (S2) from GND to `BAT-`
- [x] Keep Q2 pin 1 (S1) on GND
- [x] U6 pin 2 (VM) → **1 kΩ** (reused; the Slkor datasheet circuit shows R2 = 1 kΩ) → GND (pack negative P−, the Q2 pin 1 side). **Not** to the drain net: VM has to see P− for overcurrent, short-circuit and charger detection
- [x] Add **1 kΩ** (reused, same as TEST/D2) between BAT+ and U6 pin 5 (VDD) — datasheet circuit shows 100 Ω, but 1 kΩ shifts VDD by at most about 6 mV at max supply current, far inside the ±50 mV thresholds
- [x] Add 100 nF (existing CL05B104 part) from U6 pin 5 to `BAT-`
- [x] Q2 pins 2 + 5 (drains) are connected internally. no need to connect.

### Check

- [x] Assign footprints and LCSC numbers to all new parts
- [x] Run ERC, no errors

## 3. Right PCB — remove old, place new (`kicad/right.kicad_pcb`)

### Remove old

- [x] Update PCB from schematic (U5, R11, R15, R18 disappear; R4 and its net stay, just unused; new parts appear next to the board)
- [x] Delete the leftover traces and vias of removed nets: CHRG, STDBY, U5 PROG, and the old U5 pads area (`BAT_ALTR`/pad 18 stays — R4 and the net are kept, just unused)
- [x] Delete the old U4 pin 2 → GND and Q2 pin 3 → GND connections (they are now `BAT-`), and the old U6 pin 2 (VM) → drain trace

### Place new

- [ ] Keep the whole charger group away from the antenna end of the module (x ≈ 42–46, y ≈ 78–91) and from X1
- [ ] IP2312U first, then around it:
  - [ ] VIN: 2 × 22 µF right at pin 8, snubber (two 0.5 Ω in series + 22 µF) next to them — needs slightly more room than a single resistor would
  - [ ] 1 µH inductor right next to SW (pin 7)
  - [ ] BAT: 22 µF and the damping 0.5 Ω + 22 µF right at pin 5 / the inductor output
  - [ ] ICHG jumper (96.5 kΩ + 102 kΩ, gap-selectable, silkscreen-labeled), NTC network (2 × 102 kΩ in parallel), TEST 1 kΩ, D2 1 kΩ close to their pins
- [ ] TVS (SMF5.0A) right at the USB-C connector's VBUS/GND pins, before F1
- [ ] F1 (1.5 A polyfuse) between USB-C and VIN, physically separated from the charger group: leave a copper gap (no pour bridging) between the fuse's local copper and the charger/inductor cluster, with a few GND vias on the charger side tying the local pour to the internal ground plane
- [ ] Route the single VBUS trace from the merged USB-C VBUS pads to the fuse across that gap at constant width (~1.2 mm), not flared/teardropped into a pour at the crossing
- [ ] Make sure the two USB-C VBUS pads (0.6 mm each) merge into a properly sized junction (via stitch or small fill) before narrowing into the single 1.2 mm trace toward the fuse
- [ ] Protection extras near U6: 1 kΩ (VCC, from BAT+), 1 kΩ (VM → GND), 100 nF (VCC → `BAT-`)
- [ ] Leave room under the IP2312U for 6+ thermal-pad vias

## 4. Right PCB — route

- [ ] Power first: VBUS_RAW → F1 → VIN caps → pin 8 (wide)
- [ ] SW → inductor → BAT+: short and wide; solid GND under the charger
- [ ] 6+ GND vias in the IP2312U thermal pad; short GND loop from the VIN caps to the pad
- [ ] `BAT-`: U4 pin 2 → Q2 pin 3, short, ≥ 0.9 mm, **not joined to the GND pour**
- [ ] Q2 pin 1 → GND with 2–3 vias
- [ ] Fuse/charger thermal gap: confirm no pour (top layer) bridges across it; GND stays solid one layer down
- [ ] Small signals: ICHG (both jumper branches), NTC, TEST, D2, U6 VM (via 1 kΩ to GND) / VCC, TVS anode → GND
- [ ] Refill zones (B)
- [ ] Run DRC, no errors

## 5. Left half (mirrored)

- [ ] Section 2 in `kicad/left.kicad_sch` (ALRT goes to module pad 9 on the left board)
- [ ] Section 3 in `kicad/left.kicad_pcb`
- [ ] Section 4 in `kicad/left.kicad_pcb`
- [ ] ERC + DRC clean

## 6. Production

- [ ] Rebuild the panel (`kicad/panel.kicad_pcb`) from the updated boards
- [ ] Regenerate Gerbers, BOM and CPL (fabrication toolkit)
- [ ] Check the JLC BOM: IP2312U, 96.5 kΩ 0.1%, 102 kΩ 0.1%, 1 kΩ, 1 µH inductor, 22 µF, 0.5 Ω, polyfuse, TVS all matched and in stock (680 Ω no longer needed for this circuit)
- [ ] Optional: update `DESIGN.md` charger, status, ALRT and protection sections

## 7. Firmware (later, with ZMK)

Note: ZMK reports only the charge percentage, so the charge-status logic likely needs a small custom module. While charging, the MAX17048 reads about 50–75 mV high (its GND is on the far side of the protection FETs); this is harmless for the status logic.

- [ ] Poll MAX17048 over I²C (no ALRT)
- [ ] Charging: CRATE clearly positive (about +40 %/h at 1.45 A)
- [ ] Full: VBUS present (pad 27) + state of charge about 100% + CRATE ≈ 0
- [ ] Not charging: VBUS absent (or CRATE ≤ 0)
- [ ] LED_EN low when idle (LP5024 5 mA → 0.2 µA)
- [ ] LP5024 Power_Save_EN bit on (about 6 µA whenever all LEDs are off)

Expected battery life at 6 h/day (estimate): about 19–22 days with LEDs at 100%, about 6–9 months with LEDs at 0%.

## 8. Bring-up tests (per half)

- [ ] Battery current (multimeter in series with the battery lead, 10 A range), with the 96.5 kΩ jumper bridged: expect **~1.40 A nominal, up to ~1.54 A worst case**
- [ ] If measured current runs high (approaching 1.5 A+) or the board runs warm in a closed case during a full charge, desolder the 96.5 kΩ bridge and solder the 102 kΩ bridge instead (~1.32 A nominal, ~1.46 A worst case), then re-test
- [ ] If battery current is ~2.1 A: ICHG is open (neither jumper bridged, or a bad joint) — chip falls back to default; fix before continuing
- [ ] USB-C meter on a 3 A-capable USB-C charger: input about 0.9 A (battery at 3.0 V) up to 1.4 A (battery at 4.1 V)
- [ ] Charger IC and inductor stay below about 60 °C
- [ ] No status LED: confirm charging from battery current and MAX17048 CRATE instead
- [ ] Battery reaches about 4.2 V (not 4.35 V) when full
- [ ] Run the battery flat, plug in USB, confirm charging restarts
- [ ] Optional (oscilloscope): VIN peak at plug-in stays below 6.5 V, with your longest cable and a short thick cable
- [ ] Measure LP5024 current with LEDs on (datasheet 5 mA is at 10 mA/LED; yours may be lower)