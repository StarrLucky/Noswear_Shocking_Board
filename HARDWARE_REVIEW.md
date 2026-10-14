# Noswear Shocking Board — Hardware Review

Review of `Schematic_Noswear-project-(shocking-watch-profanity-tracking).pdf` against
`main.c`, `custom_board.h` and `README.md`.

**Method.** No PDF renderer was available in the review environment, so the schematic was read
by parsing the PDF content stream directly: stroked line segments, net-label flag polygons and
text baselines were extracted and the connectivity was reconstructed by union-find over segment
endpoints with T-junction splitting. Every claim below about a net is traced, not inferred from
label proximity. Component *values* are taken from the silkscreen text in the PDF.

---

## 1. Verified pinout

The schematic agrees with `README.md`. It did **not** agree with the firmware, which has since
been corrected.

| Function | Net / schematic | nRF51822 | Was in firmware |
|---|---|---|---|
| Shock output | `MCU_PIN` → NE555 pin 4 (RST) | **P0.06** (U1 pin 10) | `LED_SHOCK 12` |
| Status LED | schematic LED2 | **P0.12** (U1 pin 18) | `LED_1 = 1` |
| Shock-indicator LED | schematic LED1 | **P0.13** (U1 pin 19) | `LED_2 = 2` |

P0.06 is the only GPIO on the sheet carrying a named net. P0.00–P0.05, P0.07–P0.11, P0.14–P0.16
and P0.30 terminate in unconnected pin stubs.

**There are no buttons on this board.** P0.03 / P0.04 are unconnected; the `app_button` code is
dead weight inherited from the SDK blinky example.

### Polarities

- **Shock output is active HIGH.** `MCU_PIN` drives NE555 RST, which is active low, and R24
  (10k) pulls that net to ground. Low or hi-Z on P0.06 holds the timer in reset (output off);
  high releases it. The 10k pull-down also means the circuit is safe while the pin is still an
  input at reset.
- **Both LEDs are active HIGH.** Anode on the GPIO, cathode through a 200R (R1/R2) to ground.
  The stock `LEDS_INV_MASK = LEDS_MASK` inherited from PCA10028 inverted `LEDS_ON`/`LEDS_OFF`.

---

## 2. Output stage as drawn

```
VBATT ──┬── TR1 primary ── R32 (10R) ──┬── Q3 drain (D1, pin 1)
        │                              │
        └────── D31 (1N4148W) ─────────┘      ZD30 (MM3Z3V3, 3V3) ── GND
                cathode↑   anode↓             cathode at drain

555 OUT (pin 3) ── R30 (68R) ── Q3 gate (G1, pin 4),  R31 (5k) pull-down
Q3 source (S1, pins 2-3) ── GND
Q3 MOSFET 2 disabled: G2 (pin 5) and S2 (pin 6) both tied to GND

TR1 secondary ──── R33 (100k) ──── back to the other end of the secondary
```

### 2.1 D31 kills the flyback — highest impact

D31 sits across the primary with its anode on the switched node and its cathode on VBATT: a
freewheel diode. When Q3 turns off it clamps the drain at VBATT + 0.7 V and recirculates the
primary energy back into the battery. The reflected secondary voltage is therefore limited to
roughly 0.7 V × N instead of the hundreds of volts the topology exists to produce.

Per `README.md` the MOSFET was meant to drive either the transformer *or* a vibro motor. A
freewheel diode is correct for a motor and fatal for an HV flyback.

**Action: do not populate D31 when driving the transformer.**

### 2.2 ZD30 is a 3.3 V zener on the drain node

MM3Z3V3 is a 3.3 V zener, cathode on `Q3_DRAIN`, anode on GND. With VBATT specified as
3.9–13.2 V it is in breakdown continuously, shunting the primary. It is almost certainly a
copy-paste of ZD2 (the legitimate clamp on the 3.3 V rail). In SOD-323 at 300 mW it should
already have failed — worth checking whether the fitted part is open, since that may be why the
board functions at all.

**Action: remove, or replace with a high-voltage TVS sized for the intended drain excursion.**

### 2.3 555 duty cycle is inverted for a flyback

Traced timing network: Ra = R21 = 100k (VBATT → pin 7), Rb = R22 = 10k (pin 7 → pins 6 and 2),
C = C21 = 0.1 µF. Standard astable:

```
t_on  = 0.693 × (100k + 10k) × 0.1u = 7.62 ms
t_off = 0.693 ×  10k         × 0.1u = 0.69 ms
f     = 1.44 / ((R21 + 2*R22) * C)  = 120 Hz
duty  = 91.7 % high
```

So the measured frequency is ~120 Hz, not the ~60 Hz stated in `README.md`, and the MOSFET is on
for 7.6 ms of every 8.3 ms. Any small-core transformer saturates within microseconds; for the
remaining ~7.6 ms the primary is just its DC resistance. A flyback wants a short on-time
(enough to ramp primary current to target) and the majority of the cycle for energy transfer.

### 2.4 R32 = 10R in series with the primary

Once the core saturates, primary current is set by R32: 0.74 A at 7.4 V, 0.37 A at 3.7 V.
Dissipation in a single chip resistor is ~5.0 W average at 7.4 V and ~1.25 W at 3.7 V, at 92 %
duty. That is both a thermal problem in a watch case and a battery-life problem, and it is energy
that never reaches the secondary.

**Action: ≤0.5 R, or remove entirely and bound the energy by controlling on-time.**

### 2.5 No output electrodes are drawn

The secondary is a closed loop through R33 (100k) — there is no connector, pad or test point on
the sheet. Either the electrodes are omitted from the schematic or they attach somewhere on the
PCB not represented here. **This needs resolving before any current figure can be trusted.**

Note that R33 is not a defect: 100k in series with an electrode is a sensible current limiter
(350 V / 100k = 3.5 mA, see §5). It should stay in the path.

---

## 3. Why it feels weak even without the four faults above

Perceived intensity is governed by **charge per pulse** and pulse width, not by open-circuit
voltage. Nociceptors respond to pulses in the 100–300 µs range carrying roughly **1–10 µC**.
Dry skin is tens of kΩ, so delivering that charge needs hundreds of volts.

The circuit as designed is a continuous flyback with no energy storage. Even with D31 and ZD30
removed, each cycle delivers only ½·L·I² into the secondary, and the secondary conduction lasts
single-digit microseconds — one to two orders of magnitude shorter than the nerve's integration
window. The result is a sensation that is detectable but not aversive.

### Recommended architecture: storage capacitor

```
flyback 50-150 kHz ──> HV diode ──> C_store 10-22 nF / 630 V
                                         |
                                    R_ser 100k ──> electrode A
                                         +--------> electrode B
```

For C = 22 nF charged to 350 V:

| Quantity | Value |
|---|---|
| Stored energy | 1.35 mJ |
| Charge | 7.7 µC |

That lands in the target band and is ~100× the per-cycle energy of the present design. The
capacitor charges over several milliseconds and is discharged at a 50–100 Hz repetition rate,
which is also the frequency range that feels strongest.

---

## 4. Transformer selection

Requirements: 1:100 turns ratio, Isat ≥ 1.5 A, winding-to-winding isolation ≥ 600 V preferred,
height ≤ 4 mm.

Verified candidate — **Coilcraft LPR6235-752SMR**:

| Parameter | Value |
|---|---|
| Turns ratio | 1:100 |
| Primary inductance | 7.5 µH, DCR 0.085 R |
| Secondary DCR | 340 R |
| Isat | 1.6 A |
| Coupling | K > 0.95 |
| Package | **6 × 6 × 3.5 mm** SMD |
| Isolation | 300 Vrms, 1 min |

Energy per cycle: ½ × 7.5 µH × 1.6² = **9.6 µJ**. From 3.7 V, primary current ramps to 1.6 A in
3.2 µs, so a practical switching frequency is ~140 kHz, giving ~1.3 W of available charging
power — enough to charge the 1.35 mJ storage capacitor in a few milliseconds.

**Caveat: 300 Vrms is ~424 V peak, leaving almost no margin over a 350 V rail plus switching
overshoot.** Either hold the output at 250–300 V, or source an equivalent-size part with ~1 kV
isolation. The relevant category is photoflash and ionizer transformers; specific part numbers
in that category were not verified during this review.

### Drive chain

- **Q3 (8205A) is the wrong part**: a 20 V dual battery-protection MOSFET. Replace with a
  logic-level device rated 60–100 V. With a 1:100 ratio the reflected voltage keeps the
  primary-side swing modest, so a high-voltage FET is not required.
- Reduce the 68 R gate resistor (R30) to ~10 R; at 140 kHz the present value is too slow.
- Consider dropping the NE555 entirely and driving the gate from the nRF51822 via PWM on
  PPI/GPIOTE. On-time then becomes a firmware parameter, which makes intensity programmable and
  removes a part from a space-constrained board.

---

## 5. Safety envelope

A commercial wrist-worn device in the same category (Pavlok) operates at **50–450 V, 0–4.5 mA**.
Per IEC 60479-1: perception threshold ~0.5–1 mA; let-go threshold 5–10 mA; ventricular
fibrillation risk from ~40 mA for durations over 2 s.

So "clearly felt but safe" means hundreds of volts open-circuit at single-digit milliamps. R33
at 100k already produces exactly that.

Constraints to hold:

- **Both electrodes on the same wrist**, centimetres apart. A current path across the chest is
  the single change that moves this from painful to dangerous.
- Keep a series current-limiting resistor in the electrode path. It is the component that bounds
  current under any single fault.
- Firmware already caps on-time at `SHOCK_MAX_ON_TIME_MS = 2000`. 200–500 ms is ample for the
  intended effect and can be reduced.
- Do not exceed ~500 V output. Beyond that, electrode-site burns begin while perceived intensity
  barely increases.
- **Prefer electrode geometry over energy** for increasing intensity: smaller contact area and
  closer spacing raise current density. It is the cheapest and safest lever, and poor skin
  contact is a common reason such a device feels weak.
- Standard contraindications apply: pacemakers and ICDs, cardiac conditions, epilepsy,
  pregnancy, broken skin.

---

## 6. Form factor

Estimated from `pcb.jpg` (single photograph, dimensions eyeballed — not measured):

- Round PCB, roughly 32–34 mm diameter, in a 3D-printed case of roughly 40 mm.
- All visible parts are low-profile SMD; the tallest is a shielded inductor of about 4 × 4 mm.
- Height budget: case interior appears 8–10 mm, less 1.6 mm of PCB and 3–4 mm of LiPo, leaving
  roughly 3–4 mm.

A 6 × 6 mm footprint places easily on a board this size. **Height is the binding constraint** —
3.5 mm fits, but with little margin, and the transformer should go on the same side as the rest
of the low-profile parts.

Two observations from the photo: no transformer and no battery are visible on the pictured side,
so this is either the MCU side only or a revision without the output stage populated. And the
documented 3.9–13.2 V supply range implies a multi-cell pack, whereas a watch realistically
takes one 3.7 V cell — which is what the §4 calculations assume.

---

## 7. Action list, in order

1. Remove D31 and ZD30; change R32 to 0.1–0.5 R. Re-measure before anything else — this alone
   may give a large improvement.
2. Fix the drive waveform: swap R21/R22, or replace the NE555 with MCU-generated PWM.
3. Add the storage capacitor stage: LPR6235-752SMR + HV diode + 22 nF / 630 V.
4. Replace Q3 with a 60–100 V logic-level MOSFET; drop R30 to ~10 R.
5. Confirm where the electrodes actually attach and verify the current limit with the real load.

## 8. Open questions

- Where do the output electrodes connect? The secondary is drawn as a closed loop through R33.
- Is ZD30 fitted and intact on the assembled board?
- Single cell or multi-cell supply in the final build?

---

## Firmware status

The pinout and polarity errors found here have been corrected in `main.c` and `custom_board.h`:
`LED_SHOCK` is now P0.06 and active high, LEDs are P0.12/P0.13 with `LEDS_INV_MASK = 0`, the
output is de-asserted on BLE disconnect, and a single-shot `app_timer` bounds on-time at 2000 ms.
The project was not compiled during this review — no ARM toolchain was available.

One item remains open in firmware: the LED characteristic is writable without pairing
(`BLE_GAP_CONN_SEC_MODE_SET_OPEN` plus `PAIRING_NOT_SUPP`), so any central in range can trigger
the output. Closing it requires device_manager + pstorage + a whitelist, and corresponding
changes in the Android client.

---

## Sources

- [Coilcraft LPR6235 series](https://www.coilcraft.com/en-us/products/power/coupled-inductors/1-n-shielded-coupled/lpr/lpr6235/)
- [Farnell LPR6235-752SMRB](https://uk.farnell.com/coilcraft/lpr6235-752smrb/coupled-inductor-7-5uh-1-6a-20/dp/2434775)
- [Pavlok product specifications](https://helpdesk.pavlok.com/en/articles/6428701-product-specifications-1462)
- [IEC TS 60479-1, Effects of current on human beings and livestock](https://www.vde-verlag.de/iec-normen/preview-pdf/info_iects60479-1%7Bed4.1%7Den.pdf)
