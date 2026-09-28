# Charger + Protection Fix — Checklist (IP2312U)

## Context

Your requirements: switching charger (low heat), charge current **1.3–1.6 A** (goal 1.5 A), charge status known to the firmware, a hand-reworkable, cheap package, and **as few unique BOM parts as possible** (standard assembly charges per unique part).

Choice: **IP2312U_VSET** (LCSC C605432, ESOP-8).

- With ICHG = **93.1 kΩ 0.1%**, the current is 1.45 A nominal and **1.30–1.60 A** worst case (chip ±10% plus resistor 0.1%).
- Charge status comes from the **MAX17048 fuel gauge** over I²C (CRATE / state of charge), plus **VBUS sense on module pad 27** (already wired) to tell "full" from "unplugged". ALRT is not needed (the firmware polls).
- Unique parts vs today: **+5** (IP2312U, 93.1 kΩ, 1 µH, 0.5 Ω, 22 µF, 1 kΩ added; BL8573 removed). 680 Ω, 75.3 kΩ, 10 kΩ and 100 nF are reused from the existing BOM.
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
- [x] Check LCSC stock for IP2312U_VSET (C605432). **Not C605433** (that one is 4.35 V). - **C605432**
- [x] **93.1 kΩ 0.1%** 0402 resistor: Vishay TNPW040293K1BEED, LCSC [C2088785](https://www.lcsc.com/product-detail/C2088785.html) (±25 ppm/°C, in stock)
  - [x] Confirm JLC can assemble it (no basic 0.1% option exists; extended is fine)
- [x] Pick a 1 kΩ 0402 1% (e.g. JLC basic C11702) - **C11702**
- [x] Pick an inductor: 1 µH, saturation ≥ 3 A, DCR ≤ 30 mΩ, shielded, ~4×4 mm - **C2921163**
- [x] Pick a 22 µF 0805 X5R ≥ 10 V - **C296720**
- [x] Pick a 0.5 Ω resistor (0402 or 0603) - **C423160**
- [x] TSV, before polyfuse, to the gnd - **C2990427**
- [ ] Import the IP2312U footprint and symbol with easyeda2kicad (thermal pad 2.09 × 2.09 mm)

## 2. Right schematic (`kicad/right.kicad_sch`)

### Remove

- [ ] U5 (BL8573)
- [ ] R11 (680 Ω) — part removed here, but the 680 Ω value is reused below (no new unique part)
- [ ] R15, R18 (10 kΩ CHRG/STDBY pull-ups)
- [ ] CHRG and STDBY nets (module pads 41 and 43 become unconnected)
- [ ] R4 (10 kΩ ALRT pull-up) and the `BAT_ALTR` net; leave U8 (MAX17048) pin 5 unconnected (open-drain, safe to float); module pad 18 becomes free

### Add charger (IP2312U_VSET, datasheet p.9)

- [ ] Pin 8 VIN → VBUS, with 2 × 22 µF to GND (10 V, X5R, 0805)
- [ ] Input snubber: **0.5 Ω + 22 µF in series**, VBUS → GND (datasheet R2/C6: absorbs the plug-in voltage spike). Same parts as the output damping branch, so no new unique part.
- [ ] Pin 7 SW → 1 µH inductor → BAT+
- [ ] Pin 5 BAT → BAT+, with 22 µF to GND, plus 0.5 Ω + 22 µF in series to GND (damping)
- [ ] Pin 6 ICHG → **93.1 kΩ 0.1%** → GND (1.30–1.60 A)
- [ ] Pin 4 NTC → **(2 × 75.3 kΩ in parallel) + 10 kΩ in series** → GND = 47.6 kΩ → 0.95 V at 20 µA (datasheet uses 51 kΩ = 1.02 V). All existing parts. Never leave NTC floating. Battery temperature check is off (no thermistor).
- [ ] Pin 2 TEST → **1 kΩ** → BAT+ (as datasheet)
- [ ] Pin 1 D1 → LED + **680 Ω** (existing part) from VIN (LED = one of the hand-soldered key LEDs, not a JLC part). **No resistor to GND on D1** (any resistor there raises the charge voltage).
- [ ] Pin 3 D2 → **1 kΩ** → BAT+ (as datasheet)
- [ ] EPAD → GND
- [ ] F1 → 2 A-hold polyfuse
- [ ] Keep VBUS → module pad 27 as is (firmware uses it)

### Fix protection (U6 DW01A, Q2 FS8205A)

- [ ] Create net `BAT-`
- [ ] Move U4 pin 2 (battery −) from GND to `BAT-`
- [ ] Move U6 pin 6 (VSS) from GND to `BAT-`
- [ ] Move Q2 pin 3 (S2) from GND to `BAT-`
- [ ] Keep Q2 pin 1 (S1) on GND
- [ ] Take U6 pin 2 (CS) off the drain net and connect it to GND through **680 Ω** (existing part)
- [ ] Add **680 Ω** (existing part) between BAT+ and U6 pin 5 (VCC)
- [ ] Add 100 nF (existing CL05B104 part) from U6 pin 5 to `BAT-`
- [ ] Leave Q2 pins 2 + 5 connected only to each other (rename the net to `Q2_DRAIN`)

### Check

- [ ] Assign footprints and LCSC numbers to all new parts
- [ ] Run ERC, no errors

## 3. Right PCB — remove old, place new (`kicad/right.kicad_pcb`)

### Remove old

- [ ] Update PCB from schematic (U5, R11, R15, R18, R4 disappear; new parts appear next to the board)
- [ ] Delete the leftover traces and vias of removed nets: CHRG, STDBY, `BAT_ALTR` (to module pad 18), U5 PROG, and the old U5 pads area
- [ ] Delete the old U4 pin 2 → GND and Q2 pin 3 → GND connections (they are now `BAT-`), and the old CS → drain trace at U6 pin 2

### Place new

- [ ] Keep the whole charger group away from the antenna end of the module (x ≈ 42–46, y ≈ 78–91) and from X1
- [ ] IP2312U first, then around it:
  - [ ] VIN: 2 × 22 µF right at pin 8, snubber (0.5 Ω + 22 µF) next to them
  - [ ] 1 µH inductor right next to SW (pin 7)
  - [ ] BAT: 22 µF and the damping 0.5 Ω + 22 µF right at pin 5 / the inductor output
  - [ ] ICHG 93.1 kΩ, NTC resistors (2 × 75.3 kΩ + 10 kΩ), TEST 1 kΩ, D2 1 kΩ close to their pins
  - [ ] D1 LED + 680 Ω where the LED is visible (hand-soldered later)
- [ ] F1 (2 A polyfuse) between USB-C and VIN
- [ ] Protection extras near U6: 680 Ω (CS), 680 Ω (VCC), 100 nF (VCC → `BAT-`)
- [ ] Leave room under the IP2312U for 6+ thermal-pad vias

## 4. Right PCB — route

- [ ] Power first: VBUS_RAW → F1 → VIN caps → pin 8 (wide)
- [ ] SW → inductor → BAT+: short and wide; solid GND under the charger
- [ ] 6+ GND vias in the IP2312U thermal pad; short GND loop from the VIN caps to the pad
- [ ] `BAT-`: U4 pin 2 → Q2 pin 3, short, ≥ 0.9 mm, **not joined to the GND pour**
- [ ] Q2 pin 1 → GND with 2–3 vias
- [ ] Small signals: ICHG, NTC, TEST, D1, D2, U6 CS/VCC
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
- [ ] Check the JLC BOM: IP2312U, 93.1 kΩ 0.1%, 1 kΩ, 1 µH, 22 µF, 0.5 Ω, polyfuse all matched and in stock
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

- [ ] Battery current (multimeter in series with the battery lead, 10 A range): **1.30–1.60 A**
- [ ] If battery current is ~2.1 A: the ICHG resistor is open/unsoldered (chip falls back to 2.1 A); fix before continuing
- [ ] USB-C meter on a 3 A-capable USB-C charger: input about 0.9 A (battery at 3.0 V) up to 1.4 A (battery at 4.1 V)
- [ ] Charger IC and inductor stay below about 60 °C
- [ ] Charge LED blinks while charging and is solid when full
- [ ] Battery reaches about 4.2 V (not 4.35 V) when full
- [ ] Run the battery flat, plug in USB, confirm charging restarts
- [ ] Optional (oscilloscope): VIN peak at plug-in stays below 6.5 V, with your longest cable and a short thick cable
- [ ] Measure LP5024 current with LEDs on (datasheet 5 mA is at 10 mA/LED; yours may be lower)
