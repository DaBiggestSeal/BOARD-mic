# Design Review — `mic` SMD board (USB-C microphone + 3.5 mm headphone monitor + mute)

Reviewed: `mic.kicad_sch` / `mic.kicad_pcb` on branch `smd` (working copy, 2026-10-11, re-checked after the 01:07 save). KiCad 10.0.6.
This replaces the THT-board review that was copied over from `v2_design`.

**Method:**
- Exported the netlist with `kicad-cli` and traced every pin of every part.
- Ran ERC, DRC and the schematic-parity check.
- Measured placement, decoupling distances and routing with KiCad's Python API.
- Checked every chip against its current datasheet:
  - TI PCM2900C/2902C (SBFS039), including the typical-circuit figures 36–39
  - TI INA826 (SBOS562G)
  - TI TS5A3159A (SCDS200F)
  - TI TLV700xx
  - Nexperia 74LVC1G74 (rev. 18)
- Checked every LCSC code against LCSC's live product data.

**Tool results (latest run):**
- **ERC:** 0 errors, 3 warnings. The AOM-5024L, INA826 and PCM2902C symbol libraries aren't registered. This is harmless because the symbols are embedded.
- **DRC:** 0 errors, 0 unconnected. The MK2 courtyard overlaps are excluded on purpose, because the capsule is wired off-board.
- **Schematic ↔ PCB:** in sync. One expected note: J2 pin A8 has no pad, because the custom `…_NoA8` footprint removes it on purpose (SBU1 is unused).
- **Netlist:** identical to the previous review apart from C20, so every pin trace and datasheet check below still holds.

> **Applies to `v2_design` too:** the THT board uses the same schematic, so issues 1 and 2 apply there as well. Issue 1 is fixed only on `smd` so far.

---

## Status

| # | Severity | Issue | Status |
|---|---|---|---|
| 1 | 🔴 Critical | No 1.5 kΩ pull-up on D+, so the PC would never detect the device | ✅ Fixed (R9) |
| 2 | 🟠 High | Headphone jack is driven straight from the codec's DAC, which needs a load of ≥ 10 kΩ | ⏸️ Deferred |
| 3 | 🟢 Low | U6 (mute flip-flop) had no local decoupling cap | ✅ Fixed (C20) |
| 4 | 🟢 Low | Crystal under the preamp and long gain-pot tracks could add a little noise | 👂 Listen on first build |
| 5 | 🟢 Minor | Small items (decoupling distance, crystal caps, stock, assembly) | ✅ Accepted |

---

## ✅ 1. D+ pull-up: fixed

A full-speed USB device signals that it's connected with a 1.5 kΩ pull-up on D+. The PCM2902C has no internal pull-up: TI's typical circuits all show one external 1.5 kΩ from VDDI (pin 27) to D+. The earlier THT review was wrong to call it internal.

**Fix in place:**
- **Part:** **R9**, 1.5 kΩ 0603, **LCSC C22843**, on the bottom side near U3 at (150.25, 102.0).
- **Wiring:** from `Net-(U3-VDDI)` (U3 pin 27 / C8) to `Net-(U3-D+)` (U3 pin 1 / R7).
- **Placement:** TI draws it on the connector side of the 22 Ω resistor, and here it's on the chip side. That makes no practical difference, since 22 Ω is negligible next to 1.5 kΩ.
- **Checks:** routed, with DRC clean and 0 unconnected.

## ⏸️ 2. Headphone output: deferred

U3's VOUTL/VOUTR go through C11/C12 (470 µF) straight to J1.

- The datasheet's DAC spec is **load impedance ≥ 10 kΩ (AC coupled)**, and TI's reference circuits all put an "LPF, Amp" block after the outputs.
- **Headphones (16–300 Ω):** volume will be very low with heavy distortion on peaks.
- **What still works:**
  - High-impedance phones (250–600 Ω) will be quiet but listenable.
  - A line input (powered speakers, a mixer) will work properly.

**If you come back to it:**
- **a) Headphone amp.**
  - **Part:** **TI TPA6111A2DR** (SOIC-8, hand-solderable, **LCSC C497472**).
  - **Why it fits:** it runs from 5 V and drives 16/32 Ω loads single-ended, so your existing C11/C12 become its output caps.
  - **Extra parts:** input caps, the Ri/Rf gain resistors and a bypass cap, from its datasheet's typical circuit. Tie SHUTDOWN to GND, or drive it from U3's SSPND̅ (pin 28) so it mutes during USB suspend.
- **b) Line out.** Relabel J1 as "line out". C11/C12 can then be much smaller (1–10 µF).

## ✅ 3. U6 decoupling: fixed (C20)

On the THT board, C2 sat next to U6. In the SMD layout, it's about 18 mm away, next to U4.

**How big a problem is this?** It's a low risk:
- U6 only switches when the button is pressed.
- Its loads are tiny: the LED at about 3 mA and U1's control input.
- The clock input has a Schmitt trigger plus the R2/C21 debounce.

It would very likely work without the cap. Adding it is cheap insurance against a rare random mute toggle when the USB load changes.

**Why placement matters, despite the GND pour:**
- The cap's current flows in a loop through both +5V and GND. The pour makes the GND half short wherever the cap sits.
- The **+5V half** is still the 18 mm track from C2, and only a cap close to pin 8 shortens it.
- C21 (debounce) and C22 (reset timing) can't do this job, because neither is connected between VCC and GND.

**Fix in place:**
- **Part:** **C20**, 100 nF 0603, **LCSC C149620**, the same part as C16/C17/C21.
- **Wiring:** between +5V and GND.
- **Placement:** top side at (131.25, 113.25), next to U6. Its +5V pad is **3.1 mm from U6 pin 8**, and a GND via sits within 2 mm of its ground pad. ✅

## 👂 4. Noise risks around the preamp: listen on the first build

These won't stop the board from working. At worst you'd get faint hiss or whine at very high gain.

- **Crystal:** Y1 (12 MHz, bottom side) sits directly under U2, the INA826 running at up to 106× gain on the top side. 12 MHz is far above the audio band, and the INA826 has a built-in input RFI filter, so trouble would need that signal to leak in and be turned into audible noise. That's unlikely here.
- **Gain-pot tracks:** `Net-(U2-RG_3)` is about 23 mm and `Net-(R4-Pad1)` about 12 mm, running from U2 to RV1. Noise picked up there gets amplified. The nearby `MIC_ON` track only changes when the button is pressed, so the worst case is a faint click on toggle.
- **If you do hear noise:** run with a little less gain, or add GND stitching along both sides of the RG tracks. Neither needs a new board.

## ✅ 5. Minor items: accepted

- **Decoupling distance:** C6 (VCCP2I, about 5.8 mm of track) and C7 (VCCXI, about 6.8 mm) are farther from U3 than ideal. C4/C9/C5/C8 are all within 2–3 mm.
- **Crystal load caps:** Y1 has CL = 20 pF. With 22 pF caps the effective load is about 15 pF, so it runs a few tens of ppm fast, well inside the PCM2902C's ±500 ppm. The ideal value is 33 pF C0G. R10 = 1 MΩ matches TI.
- **Stock (LCSC, 2026-10-11):** U3 PCM2902CDBR (C2651869) has only 3 in stock, and MK2 (C3273730) has none. Buy both elsewhere, as planned.
- **Double-sided SMD:** U3, U5, Y1, J2 and most of the codec parts are on the bottom. That's fine for hand soldering, but JLCPCB assembly charges extra for two-sided placement.
- **J2 parity note:** this comes from the modified footprint and is expected. Don't route anything under the physical A8 pin.
- **Ceramic caps in the audio path:** C1 and C10 are X5R. They add a little distortion, which is fine for a voice mic.
- **0603 10 µF caps (C4/C9/C18) are rated 10 V:** they see ≤ 3.7 V. ✅
- **MK2:** use a short shielded cable, with signal on the core and GND on the shield.
- **Library warnings:** register or embed the three symbol libraries if you want ERC fully clean.

---

## Verified correct ✅

- **PCM2902C (U3)**:
  - **Pinout:** matches the SSOP-28 pinout, pins 1–28. The footprint is `mic:SOP65P780X200-28N`, on the bottom side.
  - **SEL0/SEL1:** tied to +3.3 V. They must be high, and they're 5 V tolerant with VIH 2 V.
  - **TEST0 (DIN):** grounded. TEST1 (DOUT) and SSPND̅ are left open. HID0–2 are grounded; they have internal pull-downs, so that's fine.
  - **Internal supply pins:** VCCCI, VCCP1I, VCCP2I, VCCXI and VDDI are only decoupled, never driven.
  - **Decoupling values:** VCCCI and VCOM 10 µF. VCCP1I/VCCP2I/VCCXI/VDDI 1 µF each, inside TI's "< 2 µF" limit.
  - **VBUS pin:** +5V → R16 2.2 Ω → pin 3 with C19 1 µF, exactly as TI shows.
  - **USB lines:** 22 Ω series resistors (R7/R8) within about 2–3 mm of the pins, and the D+ pull-up R9 1.5 kΩ to VDDI.
  - **Crystal:** 12 MHz between XTI (21) and XTO (20) with R10 1 MΩ and two load caps.
  - **Mic input:** the mono mic feeds VINL and VINR together through C10 and R15 1 kΩ. The input impedance is 30 kΩ, so the high-pass corner is about 1 Hz.
- **INA826 (U2)**:
  - **Pinout:** −IN 1, RG 2/3, +IN 4, −VS 5, REF 6, VOUT 7, +VS 8.
  - **Gain:** G = 1 + 49.4 kΩ / RG. RG = R4 470 Ω + RV1 0–10 kΩ, so G = 5.7 to 106 (15–40.5 dB).
  - **Input range:** −IN = GND on a 5 V single supply works. The input range includes V−, and datasheet figure 12 (5 V, REF 2.5 V, G 100) shows the output can swing about 0.3–4.7 V with the inputs at ground.
  - **REF:** 2.5 V from R5/R6, bypassed by C15 4.7 µF.
  - **+IN:** has a DC return through R11 10 kΩ.
- **Signal levels:** the capsule is −24 dB re 1 V/Pa, about 63 mV/Pa.
  - Normal speech: max gain gives near full scale on the codec (about 2 Vpp).
  - Loud sources: turn the gain down. At minimum gain, about 106 dB SPL reaches full scale.
- **Mic bias:** +3.3 V → R14 1 kΩ → C18 10 µF → R1 1.2 kΩ → MK2. That's about 2.2 kΩ total, matching the capsule's load resistor. The capsule sees ≥ 2.2 V, and its range is 2–10 V. R14/C18 low-pass at about 16 Hz, and C1/R11 give a 16 Hz high-pass.
- **TS5A3159A (U1)**:
  - **Pinout:** NO 1, GND 2, NC 3, COM 4, V+ 5, IN 6.
  - **Logic:** IN high connects COM to NO (live). IN low connects COM to NC, which is GND (muted, and C1 stays charged, so there's no pop).
- **Mute toggle (U6, 74LVC1G74)**:
  - **Pinout:** CP 1, D 2, Q̅ 3, GND 4, Q 5, R̅D 6, S̅D 7, VCC 8.
  - **Decoupling:** C20 100 nF, 3.1 mm from VCC.
  - **Toggling:** Q̅ is fed back to D, so every rising clock edge flips the state.
  - **Debounce:** R2 100 kΩ and C21 100 nF (τ = 10 ms).
  - **Power-on:** R18 100 kΩ / C22 1 µF (τ ≈ 100 ms) mean it always starts **live**. S̅D is tied high.
  - **Output:** Q̅ drives U1's IN and the LED.
- **LED (D1):** R3 1 kΩ from a 5 V logic output gives about 3 mA. L-204SRD (red, 1.8 mm, THT).
- **TLV70033 (U4):**
  - **Pinout:** IN 1, GND 2, EN 3, NC 4, OUT 5. EN is tied to IN.
  - **Caps:** C2 1 µF on the input and C3 1 µF on the output, which meet the stability requirement. C3 is 1.6 mm from OUT.
- **USB-C (J2, GT-USB-7051A vertical, LCSC C2843970)**:
  - **CC pins:** CC1 and CC2 each have their own 5.1 kΩ to GND (R13/R12).
  - **Data pins:** A6/B6 and A7/B7 are paired.
  - **VBUS and shield:** all VBUS pins are on +5V, and the shield is on GND.
  - **Orientation:** the connector is vertical and USB-C is reversible, so its rotation doesn't matter.
- **USB ESD (U5, USBLC6-2SC6)**:
  - **Pins:** I/O1 (pins 1, 6) on `USB_DP`, I/O2 (pins 3, 4) on `USB_DM`, VBUS on pin 5, GND on pin 2.
  - **Placement:** between J2 and R7/R8.
  - **Routing:** D+ and D− are 9.7 / 9.6 mm long, which is matched. D− uses 3 vias, which is fine at 12 Mbit/s.
- **VBUS capacitance:** about 2.3 µF total, under the USB 10 µF inrush limit.
- **Output caps C11/C12 (470 µF 16 V):** the + side is on the codec (DC ≈ 1.65 V), which is the correct polarity.
- **Jack wiring (J1, PJ-3920-3A):** S → GND, T → left, R → right. Confirm with a continuity tester on a real part.
- **Power tracks:** +5V is 0.5 mm, and everything else is 0.2 mm.
- **GND:** both layers are poured, and stitched with GND vias on a 2.5 mm grid. One 0.5/0.3 via ties U3 pin 18 (AGNDP) to the main ground.
- **LCSC codes:** every placed part has one, checked against LCSC on 2026-10-10/11.

## Next steps

1. **Port R9 (and a U6 decoupling cap) to `v2_design`** if you'll ever build the THT board (issue 1).
2. **Build and listen** at max gain for noise (issue 4).
3. **Headphone output:** decide later (issue 2).
