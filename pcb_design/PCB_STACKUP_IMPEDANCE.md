# PCB Stackup & Impedance Control

**Project:** SDR Receiver/Transceiver LTC2209
**PCB Manufacturer:** JLCPCB
**Stackup ID:** JLC06161H-7628

---

## Board Specifications

| Parameter       | Value                            |
| --------------- | -------------------------------- |
| Layers          | 6                                |
| Total Thickness | 1.6 mm                           |
| Material        | FR4 TG155 (NP-155F)              |
| Outer Copper    | 1 oz = 0.035 mm (35 µm)          |
| Inner Copper    | 0.5 oz = **0.0152 mm** (15.2 µm) |
| Surface Finish  | HASL / ENIG                      |

**Important:** Inner copper thickness per JLCPCB is 0.0152 mm, not theoretical 17.5 µm (0.0175 mm).

---

## Layer Stackup (JLC06161H-7628 based, 1.6mm)

**Source:** [JLCPCB Impedance Calculator](https://jlcpcb.com/impedance)

```
┌─────────────────────────────────────────────────────────────────┐
│  Solder Mask (Top)                              ~0.02 mm        │
├─────────────────────────────────────────────────────────────────┤
│  L1: Signal (RF/Analog)         Cu: 0.035 mm   ← RF, LNA, DSA   │
├─────────────────────────────────────────────────────────────────┤
│  Prepreg 7628                   H1: 0.2104 mm   εr = 4.4        │
├─────────────────────────────────────────────────────────────────┤
│  L2: GND (Solid)                Cu: 0.0152 mm  ← Reference L1   │
├─────────────────────────────────────────────────────────────────┤
│  Core                           H2: 0.4 mm     εr = 4.6        │
├─────────────────────────────────────────────────────────────────┤
│  L3: Signal/Power               Cu: 0.0152 mm  ← +5V, Control   │
├─────────────────────────────────────────────────────────────────┤
│  Prepreg 7628                   H3: 0.2028 mm   εr = 4.4        │
├─────────────────────────────────────────────────────────────────┤
│  L4: Signal/Power               Cu: 0.0152 mm  ← Digital aux    │
├─────────────────────────────────────────────────────────────────┤
│  Core                           H4: 0.4 mm     εr = 4.6        │
├─────────────────────────────────────────────────────────────────┤
│  L5: GND (Solid)                Cu: 0.0152 mm  ← Reference L6   │
├─────────────────────────────────────────────────────────────────┤
│  Prepreg 7628                   H5: 0.2104 mm   εr = 4.4        │
├─────────────────────────────────────────────────────────────────┤
│  L6: Signal (Digital)           Cu: 0.035 mm   ← ADC, DAC, LVDS │
├─────────────────────────────────────────────────────────────────┤
│  Solder Mask (Bottom)                           ~0.02 mm        │
└─────────────────────────────────────────────────────────────────┘
```

**Note:** Copper thicknesses from JLCPCB specs:

- Outer layers (L1, L6): 1oz = 0.035 mm (35 µm)
- Inner layers (L2-L5): 0.5oz = **0.0152 mm** (15.2 µm) — JLCPCB actual value

---

## Layer Assignment

| Layer | Type   | Content                                           | Reference      | Line Type  |
| ----- | ------ | ------------------------------------------------- | -------------- | ---------- |
| L1    | Signal | RF filters, DSA1, DSA2, LTC6432 (LNA), LDOs       | L2 (GND)       | Microstrip |
| L2    | GND    | Solid ground plane (analog shield)                | —              | —          |
| L3    | Signal | Digital signals, RF, ADC_OF differential pair     | L2 (GND) + L4 (+5V) | **Stripline** |
| L4    | Power  | +5V power plane (AC ground)                       | —              | —          |
| L5    | GND    | Solid ground plane (digital shield)               | —              | —          |
| L6    | Signal | LTC2209 (ADC), MAX5885 (DAC), Si5391B, LVDS pairs | L5 (GND)       | Microstrip |

**Note:** L3 is an **Offset Stripline** — traces between two reference planes (L2=GND, L4=+5V).
L4 (+5V) acts as AC ground when properly decoupled with bypass capacitors.

---

## Dielectric Properties

| Material              | Thickness | Dk (εr) | Df (tan δ) |
| --------------------- | --------- | ------- | ---------- |
| Prepreg 7628 (H1, H5) | 0.2104 mm | 4.4     | 0.02       |
| Prepreg 7628 (H3)     | 0.2028 mm | 4.4     | 0.02       |
| Core FR4 (H2, H4)     | ~0.4 mm   | 4.6     | 0.02       |
| Solder Mask           | ~0.02 mm  | 3.8     | —          |

**Note:** Core thickness adjusted for 1.6 mm total board thickness.

---

## Impedance Targets

| Signal Type              | Target Z | Location              | Line Type      |
| ------------------------ | -------- | --------------------- | -------------- |
| RF Single-ended          | 50Ω      | L1 (RF path)          | CPWG Microstrip |
| LVDS Differential        | 100Ω     | L6 (ADC data)         | Edge-coupled CPWG |
| Clock Differential       | 100Ω     | L6 (Si5391B → ADC)    | Edge-coupled CPWG |
| CMOS Digital             | 50Ω      | L6 (DAC data)         | CPWG Microstrip |
| **Digital/RF on L3**     | **50Ω**  | **L3 (control, RF)**  | **Stripline**  |
| **Differential on L3**   | **100Ω** | **L3 (ADC_OF pair)**  | **Coupled Stripline** |

---

## Track Geometry Calculations

### Reference Parameters (L1/L6 to GND)

- **H** (dielectric height): 0.2104 mm (prepreg 7628)
- **T** (outer copper): 0.035 mm (1oz)
- **T** (inner copper): 0.0152 mm (0.5oz per JLCPCB)
- **εr** (prepreg): 4.4
- **εr** (core): 4.6

---

### 50Ω Single-Ended (Coplanar Waveguide with Ground - CPWG)

For RF traces on L1 with GND pour and L2 as reference:

**Calculated with KiCad PCB Calculator:**

| W (trace)   | Sg (gap to GND pour) | Z0        |
| ----------- | -------------------- | --------- |
| 0.30 mm     | 0.15 mm              | ~48Ω      |
| **0.36 mm** | **0.20 mm**          | **50Ω** ✓ |
| 0.40 mm     | 0.25 mm              | ~52Ω      |

**Recommended:** W = 0.36 mm, Sg = 0.20 mm

```
Cross-section (CPWG Single-Ended):

      Sg        W        Sg
    ┌────┐  ┌──────┐  ┌────┐
    │GND │  │Signal│  │GND │   ← L1 (Top)
════╧════╧══╧══════╧══╧════╧════
─────────────────────────────── ← L2 (GND plane, H = 0.2104 mm)
```

**Nets:**

- RF_IN (J101 → T101)
- DSA1_IN, DSA1_OUT
- RF_FILTR_IN_x, RF_FILTR_OUT_x
- DSA2_IN, DSA2_OUT
- RF_1 (to LTC6432)
- F2912 switch paths

---

### 100Ω Differential (Edge-Coupled CPWG)

For LVDS pairs on L6 with L5 as reference:

**Parameters:**

- **W** — trace width (ширина доріжки)
- **S** — gap between differential pair traces (gap між парою)
- **Sg** — gap from traces to GND pour (gap до заливки землі)

**Calculated values:**

| W (trace)   | S (pair gap) | Sg (to GND pour) | Zdiff       |
| ----------- | ------------ | ---------------- | ----------- |
| 0.15 mm     | 0.15 mm      | 0.15 mm          | ~95Ω        |
| 0.18 mm     | 0.15 mm      | 0.20 mm          | ~98Ω        |
| **0.20 mm** | **0.15 mm**  | **0.20 mm**      | **~100Ω** ✓ |
| 0.20 mm     | 0.20 mm      | 0.20 mm          | ~105Ω       |

**Recommended:** W = 0.20 mm, S = 0.15 mm, Sg = 0.20 mm

```
Cross-section (Edge-Coupled CPWG Differential):

     Sg       W     S      W      Sg
   ┌────┐ ┌─────┐     ┌─────┐ ┌────┐
   │GND │ │  +  │     │  -  │ │GND │   ← L6 (Bottom)
═══╧════╧═╧═════╧═════╧═════╧═╧════╧═══
─────────────────────────────────────── ← L5 (GND plane, H = 0.2104 mm)

W  = 0.20 mm  (trace width)
S  = 0.15 mm  (gap between + and -)
Sg = 0.20 mm  (gap to GND pour on sides)
```

**Nets:**

- DA0±...DA15± (ADC Bus A)
- DB0±...DB15± (ADC Bus B)
- CLKOUTA/CLKOUTB (ADC clock output)
- CLK_0/CLK_B (Si5391B → ADC clock input)
- ADC_P_OUT/ADC_N_OUT (LTC6432 → ADC)

---

### 50Ω Single-Ended Digital (CMOS on L6)

For DAC data bus (TX_D0-TX_D15):

| W (trace)   | Sg (gap to GND pour) | Z0        |
| ----------- | -------------------- | --------- |
| 0.30 mm     | 0.15 mm              | ~48Ω      |
| **0.36 mm** | **0.20 mm**          | **50Ω** ✓ |

**Recommended:** W = 0.36 mm, Sg = 0.20 mm (same as RF)

**Nets:**

- TX_D0...TX_D15
- TX_SEL0, TX_XOR
- DSA_SCLK, DSA_DATA, DSA_LE1, DSA_LE2
- FLT_V1...FLT_V4
- RX_TX_MODE

---

## L3 Stripline Calculations

### Why L3 is Different

L3 is located **between two reference planes**:
- **L2 (GND)** — above, distance H1 = 0.4 mm (core)
- **L4 (+5V)** — below, distance H2 = 0.2028 mm (prepreg)

This creates an **Offset Stripline** geometry, which has different impedance characteristics than Microstrip (L1/L6).

```
L2: GND (solid)     ════════════════════════════════════
                           ↑ H1 = 0.4 mm (core, εr = 4.6)
L3: Signal          ───────────────────────────────────
                           ↓ H2 = 0.2028 mm (prepreg, εr = 4.4)
L4: +5V (solid)     ════════════════════════════════════
                              (AC ground with bypass caps)
```

### Reference Parameters (L3 Stripline)

**Calculated with KiCad PCB Calculator (Stripline mode):**

| Parameter | Value | Description |
|-----------|-------|-------------|
| εr | 4.5 | Average dielectric constant |
| tan δ | 0.02 | Loss tangent |
| H | 0.618 mm | Total height (H1 + H2 + T) |
| a | 0.4 mm | Offset (distance to L2) |
| T | 0.0152 mm | Copper thickness (inner) |

---

### 50Ω Single-Ended (Stripline on L3)

**Calculated with KiCad PCB Calculator:**

| W (trace) | Z0 (stripline) |
|-----------|----------------|
| 0.18 mm   | ~55Ω           |
| **0.22 mm** | **50Ω** ✓    |
| 0.25 mm   | ~45Ω           |

**Recommended:** W = 0.22 mm

```
Cross-section (Offset Stripline Single-Ended):

L2: GND ════════════════════════════════
              ↑ H1 = 0.4 mm

L3:         ┌─────┐
            │ SIG │  W = 0.22 mm
            └─────┘

              ↓ H2 = 0.2028 mm
L4: +5V ════════════════════════════════
```

**Nets on L3 (Single-Ended):**

- Control signals routed on L3
- RF signals between switches (if any)
- Digital signals that didn't fit on L1/L6

---

### 100Ω Differential (Coupled Stripline on L3)

For ADC_OF differential pair on L3:

**Estimated values (coupled stripline):**

| W (trace) | S (pair gap) | Zdiff |
|-----------|--------------|-------|
| 0.12 mm   | 0.12 mm      | ~105Ω |
| **0.15 mm** | **0.12 mm** | **~100Ω** ✓ |
| 0.15 mm   | 0.15 mm      | ~95Ω  |

**Recommended:** W = 0.15 mm, S = 0.12 mm

```
Cross-section (Coupled Stripline Differential):

L2: GND ════════════════════════════════════════
                 ↑ H1 = 0.4 mm

L3:           ┌─────┐     ┌─────┐
              │  +  │     │  -  │   W = 0.15 mm, S = 0.12 mm
              └─────┘     └─────┘

                 ↓ H2 = 0.2028 mm
L4: +5V ════════════════════════════════════════
```

**Nets on L3 (Differential):**

- ADC_OF_P / ADC_OF_N (overflow pair routed partially on L3)

---

### L3 Pour Recommendation

**Option: No Pour (Recommended for simplicity)**

Since L3 is already sandwiched between two solid planes (L2=GND, L4=+5V), additional pour is not strictly necessary for impedance control.

**If Pour is desired:**
- Use **GND pour** connected to L2 via stitching vias
- Clearance (Sg) = 0.15-0.20 mm from traces
- Via stitching every 3-5 mm

---

## Design Rules Summary

### Minimum Specifications (JLCPCB)

| Parameter     | Min Value | Recommended |
| ------------- | --------- | ----------- |
| Trace width   | 0.09 mm   | 0.15 mm     |
| Trace spacing | 0.09 mm   | 0.15 mm     |
| Via drill     | 0.2 mm    | 0.3 mm      |
| Via pad       | 0.45 mm   | 0.6 mm      |
| Annular ring  | 0.13 mm   | 0.15 mm     |

### Impedance-Controlled Traces

| Type                | W       | S (pair) | Sg (to GND) | Layer | Notes              |
| ------------------- | ------- | -------- | ----------- | ----- | ------------------ |
| 50Ω SE (RF)         | 0.36 mm | —        | 0.20 mm     | L1    | CPWG Microstrip    |
| 100Ω Diff (LVDS)    | 0.20 mm | 0.15 mm  | 0.20 mm     | L6    | Edge-coupled CPWG  |
| 50Ω SE (Digital)    | 0.36 mm | —        | 0.20 mm     | L6    | CPWG Microstrip    |
| **50Ω SE (L3)**     | **0.22 mm** | —    | N/A         | **L3** | **Stripline**     |
| **100Ω Diff (L3)**  | **0.15 mm** | **0.12 mm** | N/A   | **L3** | **Coupled Stripline** |

---

## Net Classes (KiCad)

```
Net Class: RF_50OHM (L1)
  Track Width: 0.36 mm
  Clearance: 0.20 mm         ← Sg (gap to GND pour)
  Nets: RF_IN, DSA1_*, DSA2_*, RF_FILTR_*, RF_1, F2912_*

Net Class: LVDS_100OHM (L6)
  Track Width: 0.20 mm
  Clearance: 0.20 mm         ← Sg (gap to GND pour)
  Diff Pair Gap: 0.15 mm     ← S (gap between pair)
  Nets: DA*_P, DA*_N, DB*_P, DB*_N, CLKOUTA, CLKOUTB, CLK_0, CLK_B

Net Class: DIGITAL_50OHM (L6)
  Track Width: 0.36 mm
  Clearance: 0.20 mm         ← Sg (gap to GND pour)
  Nets: TX_D*, DSA_*, FLT_V*, RX_TX_MODE

Net Class: L3_50OHM (L3 Stripline)
  Track Width: 0.22 mm       ← Narrower than L1/L6!
  Clearance: 0.15 mm         ← If using pour
  Nets: Control signals routed on L3

Net Class: L3_DIFF_100OHM (L3 Coupled Stripline)
  Track Width: 0.15 mm
  Diff Pair Gap: 0.12 mm     ← S (gap between pair)
  Clearance: 0.15 mm         ← If using pour
  Nets: ADC_OF_P, ADC_OF_N

Net Class: POWER
  Track Width: 0.5 mm (min), 1.0 mm (preferred)
  Clearance: 0.25 mm
  Nets: +5V, +3.3V*, +1.8V*, GND
```

---

## Routing Guidelines

### L1 (Analog/RF) — Microstrip

1. Keep RF traces short and direct
2. Maintain GND pour on both sides of traces (Sg = 0.20 mm)
3. Use GND vias (via stitching) every 2-3 mm along RF paths
4. Avoid 90° corners (use 45° or arcs)
5. **W = 0.36 mm** for 50Ω

### L3 (Signal) — Stripline

1. **Use narrower traces:** W = 0.22 mm for 50Ω (NOT 0.36 mm!)
2. **Differential pairs:** W = 0.15 mm, S = 0.12 mm for 100Ω
3. L3 is between L2 (GND) and L4 (+5V = AC ground)
4. Pour is optional — impedance controlled by two reference planes
5. If using GND pour: Sg = 0.15 mm, via stitch to L2
6. Keep high-speed signals away from vias to L4 (+5V)
7. ADC_OF differential pair: maintain length matching ±0.5 mm

### L4 (+5V Power Plane)

1. Keep as solid as possible (acts as AC ground for L3)
2. Add bypass capacitors (100nF + 10µF) near IC power pins
3. Connect bypass cap GND to L2 and L5 via short vias
4. Minimize cuts/splits in +5V plane under L3 signal traces

### L6 (Digital/LVDS) — Microstrip

1. Route LVDS pairs with matched lengths (±0.5 mm within pair)
2. Match all ADC data pairs to clock pair (±5 mm)
3. Keep LVDS pairs at least 3× spacing from other signals
4. Use GND vias between differential pairs
5. **W = 0.20 mm, S = 0.15 mm** for 100Ω differential
6. **W = 0.36 mm** for 50Ω single-ended

### Via Stitching

- Spacing: 2-3 mm around RF traces (L1)
- Spacing: 3-5 mm for L3 (if using pour)
- Via size: 0.3 mm drill / 0.6 mm pad
- Connect L2 and L5 GND planes with vias at board edges
- Vias from L3 GND pour → L2 (NOT to L4!)

---

## Length Matching Requirements

| Signal Group            | Layer | Matching Tolerance  |
| ----------------------- | ----- | ------------------- |
| LVDS data pairs (DA/DB) | L6    | ±0.5 mm within pair |
| All LVDS to CLKOUT      | L6    | ±5 mm               |
| CLK_0/CLK_B pair        | L6    | ±0.5 mm             |
| TX_D0-TX_D15            | L6    | ±3 mm               |
| **ADC_OF_P/ADC_OF_N**   | **L3** | **±0.5 mm within pair** |

---

## Verification Checklist

### L1/L6 (Microstrip)
- [ ] L2 and L5 are solid GND planes (no splits under RF/LVDS)
- [ ] All RF traces on L1 have GND pour with Sg = 0.20 mm
- [ ] LVDS pairs on L6 are length-matched (±0.5 mm)
- [ ] Via stitching around RF sections on L1

### L3 (Stripline)
- [ ] L3 traces use W = 0.22 mm (NOT 0.36 mm like L1/L6!)
- [ ] ADC_OF differential pair: W = 0.15 mm, S = 0.12 mm
- [ ] ADC_OF pair is length-matched (±0.5 mm)
- [ ] No cuts in L2 (GND) or L4 (+5V) under L3 signal traces

### L4 (+5V Power)
- [ ] L4 is mostly solid (AC ground for L3)
- [ ] Bypass caps (100nF + 10µF) placed near each IC
- [ ] Bypass cap GND connected to L2/L5 with short vias

### General
- [ ] GND planes (L2, L5) connected with vias at board edges
- [ ] L3 GND pour (if used) connected to L2, NOT L4

---

## References

- [JLCPCB Impedance Calculator](https://jlcpcb.com/impedance)
- [JLCPCB 6-Layer PCB](https://jlcpcb.com/6-layer-pcb)
- [JLCPCB Stackup Guidelines](https://jlcpcb.com/blog/six-layer-pcb-stackup-and-buildup-guidelines)
- [GitHub JLCPCB Stackups](https://github.com/gsuberland/jlcpcb_autogenerated_stackups)

---

**Document Version:** 1.3
**Date:** 2026-02-07

### Changelog

**v1.3 (2026-02-07):**

- **NEW:** Added L3 Stripline section with calculations
- L3 50Ω Single-Ended: W = 0.22 mm (KiCad Calculator verified)
- L3 100Ω Differential: W = 0.15 mm, S = 0.12 mm
- Updated Layer Assignment table with Line Type column
- Added L3_50OHM and L3_DIFF_100OHM net classes
- Added L3 and L4 routing guidelines
- Updated Impedance-Controlled Traces table with L3 parameters
- Updated Verification Checklist for L3/L4

**v1.2 (2026-02-07):**

- **CRITICAL FIX:** Corrected inner copper thickness to 0.0152 mm per JLCPCB specs
- Updated core thickness to ~0.42 mm for 1.6 mm total board thickness
- Added note about JLCPCB actual vs theoretical copper values
- Added source reference to JLCPCB Impedance Calculator

**v1.1 (2026-02-06):**

- Updated 50Ω SE trace: W = 0.36 mm, Sg = 0.20 mm (KiCad Calculator verified)
- Updated 100Ω Diff pair: W = 0.20 mm, S = 0.15 mm, Sg = 0.20 mm
- Added cross-section diagrams for CPWG structures
- Clarified terminology: S (pair gap) vs Sg (gap to GND pour)
