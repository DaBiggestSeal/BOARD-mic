# Design Review — `mic` SMD board, v1

USB-C microphone with a 3.5 mm headphone-monitor jack and a mute button.

- **Branch:** `smd`, working copy as of 2026-10-27.
- **Files reviewed:** `mic.kicad_sch` and `mic.kicad_pcb` (both saved 15:55). Schematic and PCB are in sync.
- **Tools:** KiCad 10.0.6.
- **Replaces:** the 2026-10-11 review.

---

## Before ordering ✅ nothing left

Every item from earlier passes is done:
- **R1 LCSC:** C4190 (2.2 kΩ).
- **R14 LCSC:** C23179 (470 Ω).
- **PCB:** updated from the schematic, with RV1 re-routed.

All 30 LCSC codes were re-checked against their schematic values on 2026-10-27 15:55, and every one matches. Only **U3** (PCM2902CDBR) and **MK2** (AOM-5024L) are out of stock at LCSC, and both are being sourced elsewhere.

---

## Tool results

- **ERC:** 0 errors, 3 warnings. The AOM-5024L, INA826 and PCM2902C symbol libraries aren't registered. This is harmless because the symbols are embedded.
- **DRC:** 0 errors, **0 unconnected**, and no GND-pour islands (checked piece by piece).
  - **Excluded on purpose:** 3 courtyard overlaps with MK2. The capsule is mounted off-board on its leads.
- **Schematic ↔ PCB:** in sync. The only note is J2 A8 having no pad, which is expected because the `…_NoA8` footprint removes it.

## How it was checked

- **Netlist:** exported with `kicad-cli`, with every pin of every part traced.
- **Datasheets:**
  - TI PCM2900C/2902C (SBFS039, Fig. 36)
  - TI INA826 (SBOS562G, Fig. 11–12)
  - TI TS5A3159A (SCDS200F)
  - TI TLV700xx
  - TI SN74LVC1G74 and SN74LVC1G17
  - Nexperia 74LVC1G74-Q100 (rev 6)
  - ST USBLC6-2 (Doc 11265 rev 5, Fig. 7/17/18)
  - PUI AOM-5024L-HD-F-R
  - Hroparts PJ-3920
  - Alps RK097 catalogue
  - SHOU HAN SH-1102AQ-55ZJ
- **LCSC:** every code checked against LCSC's live product data on 2026-10-27.

---

## Status of all findings

| # | Finding | Status |
|---|---|---|
| 1 | USB-C B6/B7 (D+/D−) weren't routed, so USB worked with the plug one way round only | ✅ Fixed |
| 2 | Headphone jack driven straight from the codec DAC (needs ≥ 10 kΩ load) | ⏭️ v2 |
| 3 | Mute flip-flop clocked by a slow RC edge, which could double-toggle | ✅ Fixed (U7 Schmitt buffer) |
| 4 | U3, MK2, U6 and SW1 out of stock at LCSC | ✅ Resolved |
| 5 | Gain pot direction vs. taper | ✅ Decided: anticlockwise = more gain; PCB re-routed |
| 6 | Mic capsule load and bias | ✅ Fixed (R1 2k2 / C4190, R14 470 Ω / C23179) |
| 7 | 4.4 mA drawn in USB suspend (limit 2.5 mA) | ✅ Accepted for v1 |
| 8 | Mute switch signal goes slightly below 0 V | ✅ No change needed |
| 9 | Minor items | Optional |

---

## 1. USB-C D+/D− on both rows ✅

**What was wrong:**
- A USB-C plug has one D+/D− pair. When the plug is flipped, it lands on B6/B7, so the receptacle must join A6↔B6 and A7↔B7.
- After the Oct 23 re-layout, only A6/A7 were routed.

**Now:**
- **Routing:** both rows are routed.
- **Schematic:** the lines are split at U5 into `_USB_DP`/`_USB_DM` (connector → U5 pins 1/3) and `USB_DP`/`USB_DM` (U5 pins 6/4 → R7/R8).
  - **Why that's correct:** in the USBLC6-2SC6, pins 1↔6 and 3↔4 are joined internally (Fig. 17: 110 mΩ + 750 pH per side).
  - **Why it's the right layout:** this is ST's recommended flow-through layout (Fig. 7, "optimized layout"; Fig. 18). ESD current passes the clamp before it reaches the codec.

## 2. Headphone output ⏭️ v2

U3's VOUTL/VOUTR go through C11/C12 (470 µF) straight to J1.

**The limits:**
- **Load:** the DAC needs a load of **≥ 10 kΩ**.
- **Level:** full scale is about 2 Vpp.
- **Headphones:** 16–32 Ω phones would need about 30 mA peak. Expect low volume and distortion.
- **Other loads:** 250–600 Ω phones are quiet but usable, and line inputs work properly.

**Monitoring:** the PCM2902C has no analog path from mic to output. You hear yourself only if the PC plays the mic back, with about 20–50 ms of delay.

**For v2:**
- **Amp:** add a TI **TPA6111A2DR** (SOIC-8, **LCSC C497472**) between the codec and C11/C12.
  - **Extra parts:** input caps, gain resistors and a bypass cap, per its datasheet.
  - **SHUTDOWN:** tie it to GND or to U3 SSPND̅ (pin 28).
- **Zero-latency monitoring:** mix the INA826 output into the amp too.
- **Plug-in click:** add 10–47 kΩ from each jack-side cap terminal to GND.

## 3. Mute toggle clock ✅

**What was wrong:**
- **Slow edge:** the release edge on U6 CP decays through C21/R2 (τ = 10 ms), about 5 ms/V.
- **Datasheet limit:** the 74LVC1G74 allows at most **10 ns/V** (Nexperia -Q100 datasheet, 2.7–5.5 V) and has no specified Schmitt input.
- **Risk:** supply noise could make the input chatter and toggle the mute twice.

**Now:**
- **Buffer:** U7, **SN74LVC1G17DBVR** (Schmitt buffer, **LCSC C7836**), sits between `BTN_RC` (SW1/R2/C21) and `BTN_CLK` (U6 pin 1). Its hysteresis is 0.6–1.0 V at 5 V.
- **Decoupling:** C23 100 nF.
- **Placement:** U7 is 3.0 mm from U6, and C23 is 1.8 mm from U7 VCC.

The slow CLR̅ edge (R18/C22, τ ≈ 100 ms) is harmless, because a level-sensitive clear just clears again.

## 4. Sourcing ✅

| Ref | Part | LCSC | Stock 2026-10-27 | Notes |
|---|---|---|---|---|
| U3 | PCM2902CDBR | C2651869 | 0 | bought elsewhere |
| MK2 | AOM-5024L-HD-F-R | C3273730 | 0 | bought elsewhere |
| U6 | 74LVC1G74DP-Q100H | **C548409** | 21 189 | same die, package and pinout as the DP,125; "-Q100" is only automotive qualification |
| SW1 | SH-1102AQ-55ZJ | **C51018255** | 600 | fits the PTS645 footprint (see below) |

**SW1 (SH-1102AQ) fit:**
- **Holes:** the switch pins are Ø1.0 mm at 4.5 mm, and the bracket legs Ø1.3 mm, 7.0 mm apart and 2.5 mm back. Both match `SW_Tactile_SPST_Angled_PTS645Vx58-2LFS`.
- **Body and force:** the body is 7.4 mm wide (the PTS645 is 6.2 mm), and the button is stiffer (2.5 N vs 1.3 N). Keep the area beside it clear.
- **LCSC field:** `alr C51018255`. The `alr` prefix is intentional: a friend's BOM script uses it to mark the part as already sourced.
- **Bracket legs:** these are the metal cover, which the drawing shows grounded. They're unconnected here, which is fine. GND would be a small ESD bonus.

Everything else is in stock. The lowest count is the INA826AID with 321.

## 5. Gain pot ✅

**Current wiring:**
- **Connections:** R4 → RV1 pin 1, and pins 2+3 → U2 RG_3. So RG = 470 Ω + R(1–2).
- **Pot:** Alps RK097111080R, 10 kΩ, **15A (log)** taper.
- **Taper and wiring:** the log taper suits this wiring.

| Knob (from fully anticlockwise) | 0 % | 25 % | 50 % | 75 % | 100 % |
|---|---|---|---|---|---|
| Gain | 40.5 dB | ~35 dB | ~28 dB | ~21 dB | 15 dB |

**Turning anticlockwise raises the gain.** That's accepted for v1.

**For v2, if you want clockwise = more gain:**
- **Wiring:** move R4 to pin 3, with pins 1+2 tied.
- **Pot:** use a **C (reverse-log) 10 kΩ** pot, e.g. the Taiwan Alpha **RD901F-40-15R1-C10K** (not on LCSC).
  - **Pins:** 3 at 2.5 mm, the same as now.
  - **Support lugs:** it adds 2 lugs, 1.8 × 1.1 mm, 11.4 mm apart and 7.5 mm toward the shaft. Add slots or snip them off.
  - **Shaft:** 6.35 mm, so it needs a ¼" knob.

  Alps only makes the 15C taper as a dual-gang RK097.

## 6. Mic capsule load and bias ✅

**Bias network:** +3.3V → R14 470 Ω → C18 10 µF to GND → R1 2.2 kΩ → MK2 (+).

**How the values work out:**
- **Sensitivity:** at audio frequencies C18 grounds the R14/R1 junction, so the capsule's load is R1 ‖ R11 ≈ 1.8 kΩ. That's about 1.7 dB below the datasheet's 2.2 kΩ rating, which is fine. With the old R1 = 1.2 kΩ it was about 6 dB.
- **DC:** the capsule is fed through R14 + R1 = 2.67 kΩ.
  - **Worst case:** at its maximum current of 0.5 mA, the capsule gets about 2.0 V.
  - **Typical:** at 0.3 mA it gets about 2.5 V.
  - **Rating:** the capsule's standard voltage is 3 V and its max 10 V. ✅
- **Supply filter:** R14/C18 low-pass at 34 Hz.
- **Levels:** the ADC reaches full scale at about **76 dB SPL** at max gain and **102 dB SPL** at min gain. That's a good range for voice.

**LCSC codes:** R1 → **C4190** (UNI-ROYAL 0603WAF2201T5E, 2.2 kΩ) and R14 → **C23179** (0603WAF4700T5E, 470 Ω), in both the schematic and the PCB. ✅

## 7. USB suspend current ✅ accepted for v1

SSPND̅ is unused, so everything stays powered in suspend:

| Load | Current |
|---|---|
| LED | ~3.2 mA (when live) |
| R5/R6 divider | 0.25 mA |
| Mic capsule | ≤ 0.5 mA |
| U3 suspend | 0.25 mA |
| INA826 | 0.2 mA |
| LDO + logic | ~0.05 mA |
| **Total** | **≈ 4.4 mA** |

The USB 2.0 suspend limit is 2.5 mA.
- **In practice:** PCs and laptops don't enforce this. The board draws about 22 mW, nothing is stressed, and it works normally on wake.
- **What you'll see:** the LED stays lit while the PC sleeps.
- **For v2:** add an AO3400A (**C20917**) in the LED cathode, with its gate on U3 pin 28.

## 8. Mute switch below 0 V ✅ no change

The TS5A3159A is specified for 0 V to V+ (absolute min −0.5 V). R11 holds the mic signal at 0 V DC, so negative half-waves dip below ground.

**Why it's fine:** with the gain set so the ADC doesn't clip, the switch sees at most about ±9 mV (max gain) to ±175 mV (min gain). The internal diodes don't conduct until about −0.5 V.

## 9. Minor items (optional)

- **U5 GND path:**
  - **Current path:** pin 2 connects only through the B.Cu pour. There's no via at the pin, because the 0.5 mm F.Cu +5V track runs directly above it.
  - **Why it's worth improving:** ST wants this path as short as possible.
  - **v2:** route +5V around U5 and add a GND via at pin 2.
- **U5 VBUS cap:** ST's Fig. 18 shows 100 nF at pin 5. The nearest +5V cap is C20, 15.6 mm away. A 0603 100 nF (C149620) next to the pin is cheap insurance.
- **INA826 REF impedance:**
  - **The spec:** REF is fed from the R5/R6 divider (5 kΩ) with C15 4.7 µF. TI asks for < 5 Ω.
  - **Why it's acceptable:** the input is single-ended, so the CMRR loss doesn't matter. USB ripple is attenuated about 23 dB at 100 Hz and 43 dB at 1 kHz.
  - **Option:** a 10 µF C15 would help the low end.
- **Crystal load caps:** Y1 (X503212MSB2GI) is CL = 20 pF. 22 pF caps give about 15 pF, so it runs a few tens of ppm fast, well inside ±500 ppm. The ideal value is 33 pF C0G.
- **Naming:** R3 is `1kR`, while R15 is `1k`, on the same C22548. Rename R3 to `1k` so the BOM merges them.
- **Untracked footprints:** in `mic.pretty`, `MIC_AOM-5024L…`, `SOIC127P599X175-8N`, `SOP65P780X200-28N` and `USB_C_…_NoA8` aren't committed. Commit them.
- **Jack T/R:** J1 sleeve (pin 1) is GND, which is correct. The PJ-3920 drawing doesn't say which of pins 2/3 is tip, so L/R might be swapped. Check with a meter.
- **Library warnings (ERC):** register or embed the three symbol libraries if you want ERC fully clean.

---

## Verified correct ✅

- **PCM2902C (U3)**, against TI Fig. 36:
  - **Pinout:** SSOP-28.
  - **Decoupling:**
    - **VCCCI and VCOM:** C4/C9, 10 µF.
    - **VDDI, VCCXI, VCCP2I, VCCP1I:** C8/C7/C6/C5, 1 µF, under the 2 µF limit.
    - **VBUS:** R16 2.2 Ω + C19 1 µF.
  - **USB:** R7/R8 22 Ω series resistors, and R9 1.5 kΩ D+ pull-up to VDDI.
  - **Crystal:** 12 MHz with R10 1 MΩ and two load caps.
  - **SEL0/1:** tied to +3.3 V (5 V tolerant).
  - **Unused pins:** DIN and HID0–2 are grounded. DOUT and SSPND̅ are left open.
  - **Mic input:** VINL+VINR are fed through R15 1 kΩ / C10 4.7 µF. The input is 15 kΩ, so the loss is 0.5 dB and the high-pass corner about 2 Hz.
- **INA826 (U2):**
  - **Pinout:** −IN 1, RG 2/3, +IN 4, −VS 5, REF 6, VOUT 7, +VS 8.
  - **Gain:** G = 1 + 49.4 kΩ / RG, giving 5.7 to 106.
  - **Input range:** with −IN = GND and REF = 2.5 V on 5 V, the output range is 1.1–3.9 V (Fig. 12). The ADC (2 Vpp) clips first.
  - **+IN:** has a DC return via R11 10 kΩ.
  - **Decoupling:** C16/C17 100 nF.
- **TS5A3159A (U1):**
  - **Pinout:** NO 1, GND 2, NC 3, COM 4, V+ 5, IN 6.
  - **Logic:** IN high connects COM to NO (live). IN low connects COM to NC, which is GND (muted). It has break-before-make.
- **74LVC1G74 (U6):**
  - **Pinout:** CP 1, D 2, Q̅ 3, GND 4, Q 5, R̅D 6, S̅D 7, VCC 8.
  - **Toggling:** Q̅ is fed back to D.
  - **Preset/clear:** S̅D is tied high. R18/C22 make it power up **live**.
  - **Decoupling:** C20.
  - **Symbol note:** it's an LVC part (1.65–5.5 V), even though the symbol was taken from 74AUC1G74.
- **SN74LVC1G17 (U7):**
  - **Pinout:** NC 1, A 2, GND 3, Y 4, VCC 5.
  - **Decoupling:** C23.
- **LED (D1):** R3 1 kΩ gives about 3.2 mA through the L-204SRD (red, Vf 1.8 V). It's lit when live.
- **TLV70033 (U4):**
  - **Pinout:** IN 1, GND 2, EN 3 (tied to IN), NC 4, OUT 5.
  - **Caps:** C2 and C3 are 1 µF, above the 0.1 µF minimum.
- **USB-C (J2, GT-USB-7051A vertical):**
  - **CC pins:** CC1 and CC2 each have their own 5.1 kΩ (R13/R12).
  - **VBUS and shield:** VBUS goes to +5V and the shield to GND.
  - **Data rows:** both A and B rows are routed.
  - **Inrush:** total VBUS capacitance is about 2.3 µF, under the 10 µF limit.
- **USBLC6-2SC6 (U5):** connector side on pins 1/3, codec side on 6/4, VBUS on 5, GND on 2.
- **C11/C12 (470 µF 16 V):** the + side is on the codec (about 1.65 V DC), which is the correct polarity.
- **MK2:** (+) goes to the bias/C1 node and (−) to GND, matching the PUI drawing.
- **LCSC codes:** all 30 are valid and match their value and footprint (re-checked 15:55). SW1's `alr` prefix is intentional.
  - **Capacitors:** X5R/X7R/C0G at ≥ 10 V.
  - **Resistors:** 1 % thick film.

## v2 list

- Headphone amp, zero-latency monitoring, jack bleed resistors (finding 2)
- Clockwise = more gain: C-taper pot with R4 on pin 3 (finding 5)
- LED switched off in suspend (finding 7)
- +5V routed around U5, with a GND via at pin 2 and 100 nF at U5 VBUS (finding 9)
