# Design Review — `mic` (USB-C microphone + 3.5 mm headphone monitor)

Reviewed: `mic.kicad_sch` / `mic.kicad_pcb` on branch `v2_design` (uncommitted working copy, 2026-10-09), KiCad 10.0.6.
Only one KiCad project exists in this folder, so this is the only review file.

**Method:** exported the netlist with `kicad-cli` and traced every net by hand. Ran ERC, DRC and the DRC schematic-parity check. Checked pinouts, reference circuits and limits against the TI datasheets (PCM2900C/2902C SBFS039, TS5A3159A SCDS200F, INA826 SBOS562), the PUI AOM-5024L-HD-R specs, and the LCSC listings for every part that has an LCSC number.

**Tool results:** ERC: 0 errors, 7 warnings (6 about missing library-table entries, 1 about the PWR_FLAG on VBUS, all harmless). DRC: **2 errors** (courtyard overlaps), 0 unconnected. Schematic and PCB are in sync, apart from missing `LCSC` fields on the footprints.

---

## Summary

The signal chain is a good choice and the IC pinouts are all correct. That chain is: electret, then the TS5A3159A switch, then the INA826 preamp, then the PCM2902C codec over USB, with the DAC output going to the 3.5 mm jack. There are **three problems that will stop the headphone output from working properly**, and several smaller issues:

| # | Severity | Issue |
|---|---|---|
| 1 | 🔴 Critical | **C11 is in backwards**: its + pad is on the jack and its − pad is on VOUTR (≈1.65 V DC) |
| 2 | 🔴 Critical | **J1 footprint is a mono Thonkiconn (PJ398SM)**. The "R" pad is really the tip-switch contact, so the right channel never reaches the plug. The LCSC part (SOFNG PJ-358) also doesn't match the footprint. |
| 3 | 🟠 Major | **The PCM2902C VOUT drives the headphones directly.** It is rated for a load of at least 10 kΩ, and TI's reference circuits all put an "LPF, Amp" stage after it. |
| 4 | 🟠 Major | **The INA826 (U2) and TS5A3159A (U1) have no supply decoupling.** They are fed from raw, unfiltered USB VBUS. |
| 5 | 🟠 Major | **The crystal BOM part and footprint don't match.** LCSC C326537 is an HC-49/US can but the footprint is a C26-LF cylinder. The part's CL is 18 pF, so 22 pF load caps are a bit low. |
| 6 | 🟡 Medium | Mic bias comes straight from the LDO with no RC filter. LDO noise will likely set the noise floor. |
| 7 | 🟡 Medium | The INA826 can drive VINL/VINR beyond the PCM2902C absolute maximum ratings at high gain. |
| 8 | 🟡 Medium | SW1 is a **momentary** tactile switch, so this is push-to-talk, not a mute toggle. Also expect a pop each time it switches. |
| 9 | 🟡 Medium | PCM2902C VBUS pin has no local 2.2 Ω + 1 µF filter as in TI's reference, and the USB data lines have no ESD protection |
| 10 | 🟢 Minor | LED current is 20 mA, the analog switch signal goes slightly below GND, BOM fields are missing, and there are 2 DRC courtyard errors |

---

## 🔴 1. C11 polarity is reversed

The netlist (and the PCB pads) show:

```
C12: pad 1 (+) = VOUTL (U3.16)   pad 2 (−) = J1.T      ✅ correct
C11: pad 1 (+) = J1.R            pad 2 (−) = VOUTR (U3.15)   ❌ reversed
```

VOUTL and VOUTR sit at VCOM, about 0.5 × VCCCI ≈ 1.65 V DC. The jack side is at 0 V through the headphone. As a result, C11 is reverse-biased by about 1.65 V all the time. An aluminium electrolytic will leak, slowly degrade, and can eventually fail.

Part of the cause is that C11 and C12 use the **non-polarized `Device:C` symbol** with a polarized `CP_Radial` footprint. Because the schematic shows no polarity marking, the mistake is easy to miss.

**Fix:** Switch C11 and C12 to `Device:C_Polarized` (CP). Wire both caps with + toward the codec, then update the PCB.

## 🔴 2. The headphone jack is mono

`J1` uses `Connector_Audio:Jack_3.5mm_QingPu_WQP-PJ398SM_Vertical_CircularHoles`, which is a Thonkiconn eurorack jack. It is a **mono (TS) jack** with three contacts: **S, T and TN**. TN is the normalling switch that touches the tip when no plug is inserted. In the stock KiCad library the pads are `S`, `T` and `TN`. In this board's copy, `TN` was renamed to `R`, which is also why DRC used to report `lib_footprint_mismatch` on J1.

What this does in practice:
- **Plug inserted:** TN lifts off the tip, so the right channel (C11) connects to nothing. A stereo plug only gets the left channel on its tip, and its ring touches the sleeve or floats.
- **No plug:** TN touches T, which shorts VOUTL and VOUTR together through C11 and C12.

The LCSC number `C22355828` is a **SOFNG PJ-358**. That is a stereo TRS jack, but its pin layout is not the Thonkiconn footprint, so the board would not fit the part you are actually ordering.

**Fix:** Choose a stereo TRS jack, then use **its** footprint. For example, use the PJ-358's own footprint/datasheet drawing, or a common PJ-320/PJ-313 style part with a matching KiCad footprint. Map T to left, R to right and S to GND. If that jack has switch contacts, leave them unconnected.

## 🟠 3. No headphone driver after the DAC

Datasheet (SBFS039, DAC characteristics): *VO = 0.6 VCCCI Vpp, load impedance **10 kΩ min**, AC coupling*. Every TI typical circuit (Fig. 36–39) shows **"LPF, Amp"** after VOUTL/VOUTR.

The PCM2902C output is a line output. Into 16–32 Ω headphones it will be very quiet and distorted, and it is outside the specified operating conditions. The 470 µF caps solve the bass roll-off but not the drive problem.

**Fix options, best first:**
- **TPA6132A2** or a similar ground-referenced (DirectPath) headphone amplifier. It runs on 3.3/5 V and needs **no** output coupling caps, so C11 and C12 could be removed. It also has a shutdown pin for pop-free start-up.
- A small dual op-amp buffer that can drive 32 Ω (e.g. NJM4556A / TPA6111A2), followed by output caps.
- At minimum, if you keep the direct drive: add about 33–47 Ω in series with each output. This protects VOUT against a short from the plug while it is being inserted, and you accept low volume.

## 🟠 4. Missing decoupling on U2 (INA826) and U1 (TS5A3159A)

On the `+5V` net, the only capacitor is **C2** (1 µF), which is the LDO input cap. Pin 8 (+VS) of U2 and pin 5 (V+) of U1 have no local bypass capacitor. The INA826 datasheet layout section requires bypass caps "as close to the device pins as possible". Also, USB VBUS carries noise from the host's switching supplies and SOF-rate bursts.

**Fix:**
- Add **100 nF** right at U2 pin 8 and **100 nF** at U1 pin 5.
- Better: give the analog section its own filtered 5 V. Use a ferrite bead or 10 Ω from VBUS to an "+5VA" rail, with 10–47 µF plus 100 nF on that rail. Feed U2, U1 and the R5/R6 REF divider from +5VA.

## 🟠 5. Crystal: footprint does not match the part, and load caps are low

- `Y1` footprint: `Crystal_C26-LF_D2.1mm_L6.5mm_Vertical`, a 2 mm cylindrical watch-crystal can.
- `Y1` LCSC `C326537` is a **Citizen HC-49/U-S12000000ABJB**: an **HC-49/US** can with 4.88 mm lead spacing and **CL = 18 pF**.

These will not fit each other. Choose one and make the footprint and BOM agree. For HC-49/US use `Crystal:Crystal_HC49-U_Vertical`. Watch-crystal cylinders at 12 MHz are rare anyway.

Load caps: C = 2 × (CL − C_stray) ≈ 2 × (18 − 4) ≈ **27–33 pF**. TI allows 10–33 pF depending on the crystal. With 22 pF the crystal will run a little fast. USB full speed tolerates ±2500 ppm, so it will still enumerate, but **27 pF** is the correct value for this crystal. R10 = 1 MΩ matches TI's circuit. ✅

## 🟡 6. Mic bias noise

The bias path is +3.3 V (TLV70033) → R1 2.2 kΩ → capsule. The electret's internal JFET behaves like a current source, so noise on the bias rail appears at the mic output almost 1:1. A TLV700 has tens of µVrms of output noise. The capsule gives -24 dB re 1 V/Pa (≈63 mV/Pa) with 80 dB SNR, so its own noise is only about 14 dB(A) SPL equivalent. LDO noise of around 30 µV corresponds to about 28 dB SPL, more than ten times louder than the capsule's own noise.

**Fix:** Add an RC filter between the LDO and the bias resistor. For example: +3.3 V → **1 kΩ** → node with **47–100 µF** to GND → **1.2 kΩ** → mic. That keeps about 2.2 kΩ total and gives a corner of about 2–3 Hz. Better still, run the bias from the filtered 5 V rail with the same RC (the capsule is rated 2–10 V), which gives slightly more headroom.

## 🟡 7. Preamp can overdrive the codec input

- PCM2902C ADC full-scale is 0.6 × VCCCI ≈ **2 Vpp** (≈0.7 Vrms). VIN absolute maximum is **−0.3 V to VCCCI + 0.3 V**, and input current is limited to **±10 mA**.
- INA826 gain: G = 1 + 49.4 kΩ / (R4 + RV1) gives **5.7 to 106** (15 to 40.5 dB). ✅ The equation and pins are right.
- At G = 106 the ADC clips at about **74 dB SPL**, which is close-up loud speech. The INA826 output can swing about 0.1 V to 4.85 V (≈4.7 Vpp). AC-coupled through C10 around the ADC's mid-rail bias, that pushes VINL/VINR to roughly −0.7 V and +4 V, which exceeds the absolute maximum rating.

The gain range itself is reasonable for a speech mic. **Fix:** Add about **1–2.2 kΩ in series** between C10 and VINL/VINR. That limits clamp current to well under 1 mA and costs only about 0.6–1.2 dB, because the two 30 kΩ inputs in parallel form a 15 kΩ load. Optionally lower the maximum gain (raise R4 to about 1 kΩ, giving 34 dB max).

C10 (4.7 µF into the two 30 kΩ inputs in parallel, ≈15 kΩ) gives a high-pass corner of about 2.3 Hz. ✅

## 🟡 8. SW1 / mute behaviour and pop

- **Logic check ✅** (verified against the TS5A3159A datasheet: NO=1, GND=2, NC=3, COM=4, V+=5, IN=6; IN high connects COM to NO). Pressing SW1 pulls IN to 5 V, so the mic (via C1) connects to INA826 +IN and the LED lights. When released, R2 pulls IN low and +IN is held at GND by R11.
- **But** SW1 is a **C&K PTS645VL58 momentary** tactile switch. The mic is live only while the button is held, so this is **push-to-talk**. If you want a "mute toggle", use a latching switch (e.g. an SS12-type slide or a latching push switch). You could also wire a button to **PCM2902C HID0**, which sends a standard HID *mute* key to the host. That pin is currently tied to GND.
- **Pop:** NC (pin 3) is unconnected. While muted, C1's switch-side terminal floats, so pressing the button dumps whatever charge C1 holds into the preamp, causing a click at up to 40 dB gain. **Fix:** tie **U1 pin 3 (NC) to GND**, or move R11 to the COM side. That way C1 always sees a DC path and stays charged.

## 🟡 9. USB input protection / PCM2902C VBUS filter

- TI's PCM2902C reference circuits feed pin 3 (VBUS) through **2.2 Ω + 1 µF** to GND, placed at the pin. Here pin 3 connects straight to VBUS, and the nearest cap (C2) is about 15 mm away next to the LDO. Add that RC at U3.
- There is no ESD protection on D+/D− or VBUS, and the connector is user-facing. Add a **USBLC6-2SC6** (or TPD2E2U06) between J2 and R7/R8.
- Optional: a 500 mA PTC on VBUS.

## 🟢 10. Minor items

- **LED current:** (5 V − ~2 V) / 150 Ω ≈ **20 mA** while talking, which is at the absolute maximum for many 1.8 mm LEDs. It also uses a fifth of the 100 mA that the PCM2902C reports to the host (`bMaxPower` = 100 mA). The codec alone draws up to 67 mA. Use **1 kΩ** (≈3 mA), which is plenty bright.
- **Signal below GND at the switch/INA input:** +IN is referenced to 0 V by R11, so the audio swings a few mV either side of ground. The TS5A3159A analog range is 0…V+, and the INA826 input range includes V− only down to a little below it. At electret levels this is harmless. A cleaner option is to return R11 **and** −IN to the 2.5 V REF node instead of GND. Then both the switch and the INA sit at mid-supply, and REF noise becomes common-mode and is rejected.
- **DRC errors (2):**
  - **U1 ↔ MK2 courtyard overlap: acknowledged, not an issue.** The capsule is mounted off-board on wires, so the MK2 footprint only serves as two solder pads. Use a short **shielded** cable (signal on the core, GND on the shield), because the capsule output is a high-impedance, millivolt-level signal that picks up hum easily.
  - **U2 ↔ C10 courtyard overlap: still open.** Move them apart before fab.
- **BOM gaps:** no LCSC numbers for C10, C15, all resistors, D1, MK2, and U4. U4's LCSC field holds the text `alr`, so give it a real TLV70033DBVR part number. RV1 (Alps RK097111080J, C97444) is currently out of stock at LCSC.
- **RV1 taper:** gain in dB is very non-linear with a linear rheostat, because most of the range sits at the low-resistance end. That is acceptable, or use a reverse-log (C) taper part if you want even control.
- **Library tables:** `AOM-5024L-HD-F-R`, `INA826AID` and `PCM2902C` aren't in the symbol/footprint library tables, which causes 6 ERC warnings. The board still works because the symbols and footprints are embedded, but register them to keep the project portable.
- **Ground symbol:** the design uses `power:Earth` for signal ground. That is cosmetic only, though `GND` is the normal convention.
- **THT ceramics at 4.7–10 µF** on `C_Disc_D3.4mm` footprints are unusual. Check that the parts you buy (C4, C9, C10, C15) actually fit a 3.4 mm disc at 2.5 mm pitch.

---

## Verified correct ✅

- **PCM2902C (U3)**:
  - Pinout matches SBFS039 (SSOP-28 DB, footprint `SOP65P780X200-28N` ✅).
  - SEL0/SEL1 must be high. They are tied to +3.3 V, which is above VIH = 2 V and the pins are 5 V tolerant. ✅
  - HID0–2 and DIN are grounded. DOUT and SSPND are left open. ✅
  - The D+ pull-up is internal, so no external 1.5 kΩ is needed. R7/R8 = 22 Ω match TI. ✅
  - Decoupling matches TI's notes: VCCCI 10 µF (C4), VCOM 10 µF (C9), and VCCP1I/VCCP2I/VCCXI/VDDI 1 µF each (C5–C8), all within TI's "must be less than 2 µF" limit. ✅
  - Mono mic duplicated into VINL and VINR. ✅
  - VOUTL goes to Tip and VOUTR to Ring in the intended mapping, though see items 1 and 2. ✅
- **INA826 (U2)**:
  - Pinout matches: −IN=1, RG=2/3, +IN=4, −VS=5, REF=6, VOUT=7, +VS=8. ✅
  - Single 5 V supply is allowed (the minimum is ±1.5 V, i.e. 3 V). The input range includes V−, so −IN = GND works. ✅
  - +IN has a DC return through R11 (10 kΩ). ✅
  - REF = 2.5 V from the R5/R6 10k/10k divider, bypassed by C15 = 4.7 µF (≈7 Hz corner, so low impedance at audio frequencies). ✅
- **TLV70033 (U4)**: SOT-23-5 pinout IN=1, GND=2, EN=3, NC=4, OUT=5. ✅ EN is tied to VIN and both input and output caps are 1 µF. ✅ The 5.5 V maximum input is within the USB VBUS range. ✅
- **AOM-5024L-HD-R (MK2)**: rated 3 V (2–10 V range), 500 µA. R1 = 2.2 kΩ is the manufacturer's load resistor. C1 = 1 µF into 10 kΩ gives a 16 Hz corner. ✅
- **USB-C (J2, GT-USB-7051A, vertical SMD)**:
  - CC1 and CC2 each have their own 5.1 kΩ to GND (R13/R12). ✅
  - A6/B6 and A7/B7 are paired correctly for either plug orientation. ✅
  - SBU pins are unconnected. ✅
  - The footprint matches the LCSC part. Confirm that a vertical (upright) receptacle on the bottom side is what you want mechanically.
- **C11/C12 value**: 470 µF, 10 V Nichicon UHE (C1579341, 8 mm, 3.5 mm pitch) matches the `CP_Radial_D8.0mm_P3.50mm` footprint. ✅ The value is fine: about 10 Hz corner into 32 Ω. Only the polarity of C11 is wrong.

## Suggested fix order

1. Fix C11 polarity and switch C11/C12 to the polarized symbol.
2. Replace J1 with a real stereo TRS jack and its matching footprint.
3. Add a headphone amp (TPA6132A2 recommended) or at least series resistors.
4. Add decoupling (100 nF at U1/U2) and a filtered analog 5 V rail.
5. Fix the Y1 footprint/BOM mismatch and change C13/C14 to 27 pF.
6. Add the mic-bias RC filter, the VIN series resistor, and the 2.2 Ω + 1 µF at PCM2902C VBUS. Add USB ESD protection.
7. Decide between push-to-talk and a mute toggle. Tie U1 NC to GND to remove the pop.
8. Clear up the minor items: LED resistor, the U2/C10 courtyard overlap, BOM/LCSC fields.
