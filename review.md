# Design Review — `mic` (USB-C microphone + 3.5 mm headphone monitor)

Reviewed: `mic.kicad_sch` / `mic.kicad_pcb` on branch `v2_design`, after the schematic changes for the bias filter, codec input resistor and USB protection. KiCad 10.0.6. Last updated 2026-10-09.
Only one KiCad project exists in this folder. `main` has its own `review.md`, which doesn't include the v2 jack change (J1 = PJ-3920-3A, footprint in the project library `mic.pretty`).

**Method:** exported the netlist with `kicad-cli` and traced every net by hand. Ran ERC, DRC and the DRC schematic-parity check. Checked pinouts, reference circuits and limits against the TI datasheets (PCM2900C/2902C SBFS039, TS5A3159A SCDS200F, INA826 SBOS562), the PUI AOM-5024L-HD-R specs, and the LCSC listings for every part that has an LCSC number.

**Tool results:**
- **ERC:** 0 errors, 6 warnings, all about the AOM-5024L / INA826 / PCM2902C symbols not being registered in the library tables (harmless).
- **DRC:** ✅ clean apart from the U1 ↔ MK2 overlap, which is ignored because the capsule is mounted off-board. 0 unconnected.
- **Schematic ↔ PCB:** ✅ in sync (0 footprint/symbol mismatches).

---

## Status

| # | Severity | Issue | Status |
|---|---|---|---|
| 1 | 🟢 Minor | Part-fit checks for school parts, silkscreen, library warnings | ❌ Open |

---

## What changed on the PCB (for reference)

The bias filter, codec input resistor, VBUS filter and USB ESD protection are placed and routed. **All passives are through-hole** and use the footprints already on the board: `R_Axial_DIN0204_L3.6mm_D1.6mm_P2.54mm_Vertical` for resistors and `C_Disc_D3.4mm_W2.1mm_P2.50mm` for capacitors. Only the chips are SMD; U5 is a SOT-23-6.

| Part | Where | Notes |
|---|---|---|
| **U5** USBLC6-2SC6 (SMD) | Bottom side, above J2 at (81.0, 89.75) mm | D− runs up the bottom layer and D+ up the top layer (via near U5) from R8/R7's connector-side pads. VBUS comes from C17's +5V pad, and GND goes through a via under the chip. |
| **R16** 2.2 Ω | At (81.0, 106.0–108.5) mm, next to J2's +5V entry | Feeds a dedicated filtered-VBUS track. That track runs on the bottom layer beside R12, goes to the top layer through a via above C16, and ends at C19 and U3 pin 3. |
| **C19** 1 µF | At (93.33–95.83, 102.35) mm, just below U3 | Connected to U3 pin 3's via. VCCCI (U3 pin 10 → C4) now runs on the bottom layer between C19's pads. |
| **R15** 1 kΩ | At (85.0, 83.5–86.0) mm, above C9 | C10 → R15 runs on the top layer between C9's pads. R15 → VINL runs on the top layer past C17 to a via onto U3 pins 12/13. |
| **R14** 1 kΩ | At (98.5–101.0, 106.0) mm, inline in the +3.3 V track | Everything after R14 (the long track up to R1) is now the mic-bias node (`Net-(C18-Pad1)`). |
| **C18** 10 µF | **C3's old spot**, next to R1 | Changed from 47 µF electrolytic to a 10 µF ceramic disc. Filter corner ≈ 16 Hz, still below the audio band. |
| **C3** 1 µF | **Moved next to U4** (the LDO), fed from U4's output via | It was ~25 mm from the LDO before. An LDO output cap belongs at the LDO, so this is an improvement. |

Other routing changes made to fit these:
- **+3.3 V under U3** (U3 pins 8/9 → U4/R14) now runs straight down between C16's pads instead of diagonally.
- **The +5V track that used to pass through U3 pin 3's via** now goes around it to U2 pin 8. That via now carries only the filtered VBUS.
- **C16 moved 0.1 mm left** to make room for C19.
- **Ground stitching vias** (0.6 mm / 0.3 mm drill) tie the top and bottom ground pours together everywhere. There are 86 of mine: 38 on a ~3 mm grid across the board, plus 48 on a 2 mm grid under the pot (RV1), close around the headphone jack (J1) and around the USB +5V entry / C2 / U4 / R16. With your own 22 ground vias between U3 and J1, that makes 108. This includes the small patches that were previously joined only through one pad (around C16, R12/J2's shell and U5).
- Your own track tidy-ups around U5/C17 (21:27 save) are kept.

## 🟢 1. Minor items

- **Sourcing / BOM:** most resistors and capacitors will be borrowed from school, so they don't need LCSC codes. Chips and connectors, checked against LCSC on 2026-10-09:

  | Part | LCSC | Status |
  |---|---|---|
  | U1 TS5A3159ADBVR | C128405 | ✅ correct, in stock |
  | U2 INA826AID | C2868853 | ✅ correct, in stock |
  | U3 PCM2902C | — | ✅ no code (sourced elsewhere) |
  | U4 TLV70033 (LDO) | — | ✅ no code (sourced elsewhere) |
  | U5 USBLC6-2SC6 | C7519 | ✅ correct, in stock |
  | J1 PJ-3920-3A | C707259 | ✅ correct |
  | J2 GT-USB-7051A | C2843970 | ✅ correct, in stock |
  | MK2 capsule | — | sourced elsewhere, no code |
  | RV1 | C97444 | ⚠️ out of stock at LCSC |
- **School parts vs. footprints:**
  - **Resistors:** they mount standing, with 2.54 mm hole spacing. Check that 1/4 W bodies don't touch their neighbours around R14/R15/R16.
  - **10 µF caps (C4, C9, C18):** if you use electrolytics instead of ceramics, the footprint shows no "+". Fit + to C4 pad 2 (VCCCI), C9 pad 1 (VCOM) and C18 pad 1 (MIC_BIAS), and use 5 mm cans with 2.5 mm lead spacing.
  - **Y1:** ⏸️ accepted. The crystal's legs will be bent to fit the C26-LF footprint, because no crystal with the right specs and footprint could be found. For CL = 18 pF, 27 pF is the ideal value for C13/C14; 22 pF still works.
- **Capacitor parts (if ordered rather than borrowed):**
  - **C4, C9, C18 (10 µF):** C1620730, Murata RDEC71H106K3S1H03A, 10 µF 50 V X7S. It's a ceramic, so it isn't polarized, it uses 2.5 mm lead spacing, and it fits the C_Disc footprint. No electrolytic is needed.
  - **C11, C12 (470 µF):** C1579341, Nichicon UHE1A471MPD1TD, 10 V, 8 mm can, 3.5 mm lead spacing. These are the only polarized caps; + goes to the codec side.
  - **C19 (1 µF):** C1620326, the same Murata disc as C5–C8. Low stock (34) at the time of checking.
- **Silkscreen:** a few new reference labels sit close to or touch other labels or pads (a few more `silk_overlap`/`silk_over_copper` notes than before). Your project ignores silkscreen DRC, but nudge them in KiCad if you want the printed labels clean.
- **MK2:** the capsule is mounted off-board on wires ✅. Use a short **shielded** cable (signal on the core, GND on the shield), because the capsule signal is weak and high-impedance and picks up hum easily.
- **RV1 taper:** with a linear pot, most of the gain change happens in the first part of the rotation. That's acceptable. A reverse-log (C) taper would give more even control.
- **Library warnings:** `AOM-5024L-HD-F-R`, `INA826AID` and `PCM2902C` aren't registered in your KiCad libraries, which causes the 6 ERC warnings. It's harmless because the symbols and footprints are embedded in the project.
- **THT ceramics at 4.7–10 µF** on `C_Disc_D3.4mm` footprints: check that the parts you buy (C4, C9, C10, C15) actually fit a 3.4 mm disc at 2.5 mm pitch.

---

## Verified correct ✅

- **PCM2902C (U3)**:
  - Pinout matches SBFS039 (SSOP-28 DB, footprint `SOP65P780X200-28N`).
  - SEL0/SEL1 must be high. They are tied to +3.3 V: VIH min is 2 V and the pins are 5 V tolerant.
  - HID0–2 and DIN are grounded. DOUT and SSPND are left open.
  - The D+ pull-up is internal, so no external 1.5 kΩ is needed. R7/R8 = 22 Ω match TI.
  - Decoupling matches TI's notes: VCCCI 10 µF (C4), VCOM 10 µF (C9), and VCCP1I/VCCP2I/VCCXI/VDDI 1 µF each (C5–C8), all within TI's "must be less than 2 µF" limit.
  - The mono mic is fed into both VINL and VINR.
  - VOUTL goes to Tip (via C11) and VOUTR to Ring (via C12), which is the correct mapping.
- **INA826 (U2)**:
  - Pinout matches: −IN=1, RG=2/3, +IN=4, −VS=5, REF=6, VOUT=7, +VS=8.
  - G = 1 + 49.4 kΩ / (R4 + RV1) = 5.7 to 106 (15 to 40.5 dB), a good range for speech.
  - A single 5 V supply is allowed, and the input range includes V−, so −IN = GND works.
  - +IN has a DC return through R11.
  - REF = 2.5 V from R5/R6, bypassed by C15 = 4.7 µF.
- **TS5A3159A (U1)**: pinout and logic verified.
- **R3 / D1**: R3 = 1 kΩ gives about 3 mA through the LED. ✅
- **TLV70033 (U4)**: pinout IN=1, GND=2, EN=3, NC=4, OUT=5. EN is tied to VIN. Input and output caps are 1 µF each.
- **AOM-5024L-HD-R (MK2)**: rated 3 V (2–10 V range). C1 = 1 µF into 10 kΩ gives a 16 Hz corner.
- **Mic-bias filter**: +3.3 V → R14 1 kΩ → the mic-bias node (`Net-(C18-Pad1)`) (C18 10 µF to GND) → R1 1.2 kΩ → MK2. The total of about 2.2 kΩ matches the manufacturer's load resistor, so the operating point is unchanged. R14/C18 low-pass at about 16 Hz, which removes LDO noise from the audio band.
- **Switch pop fix**: U1 pin 3 (NC) is tied to GND, so C1 stays charged while muted.
- **USB-C (J2)**:
  - CC1 and CC2 each have their own 5.1 kΩ to GND.
  - A6/B6 and A7/B7 are paired correctly.
  - SBU pins are unconnected.
  - The footprint matches the GT-USB-7051A, a vertical receptacle.
- **J1 (Korean Hroparts PJ-3920-3A, LCSC C707259)**: stereo TRS jack with no switch contacts. The footprint is `mic:Jack_3.5mm_KoreanHropartsElec_PJ-3920-3A_Vertical`. Wiring: S → GND, T → C11 (left), R → C12 (right). Confirm the S/T/R pin assignment on a real part with a continuity tester before soldering a batch.
- **C10 + R15**: C10 (4.7 µF) → R15 (1 kΩ) → VINL/VINR. R15 limits any overdrive current into the codec's protection diodes to well under 1 mA, and the level loss is only about 0.6 dB. The high-pass corner is about 2.1 Hz.
- **PCM2902C VBUS**: +5V → R16 2.2 Ω → U3 pin 3, with C19 1 µF to GND, as in TI's reference circuit.
- **USB ESD (U5, USBLC6-2SC6)**: I/O1 (pins 1, 6) is on `USB_DP`, I/O2 (pins 3, 4) is on `USB_DM`, both on the connector side of R7/R8. VBUS (pin 5) goes to +5V and GND (pin 2) goes to GND. The CC1/CC2 nets are unaffected.

## Suggested fix order

1. **Silkscreen (optional):** tidy the reference labels around the new parts.
