<p align="center">
  <img src="BitForge SDR_logo_invert.png" alt="BitForge SDR Logo" width="280">
</p>

<h1 align="center">BitForge SDR</h1>

<p align="center">
  <b>Pure Signal. Full Control.</b><br>
  Open-Source 16-bit Direct-Sampling HF SDR Transceiver | 9 kHz -- 72 MHz | >100 dB Dynamic Range
</p>

A high-performance Software Defined Radio transceiver built around the LTC2209 16-bit 153.6 MSPS ADC and MAX5885 16-bit DAC. Designed for demanding HF reception from VLF through the 4-meter band, with full transmit capability and instrumentation-grade dynamic range.

> **Note:** This project is under active development. The hardware design (schematics + PCB) is complete. FPGA firmware is in progress.

---

## Key Specifications

| Parameter | Value |
|---|---|
| **ADC** | LTC2209 -- 16-bit, 153.6 MSPS |
| **DAC** | MAX5885 -- 16-bit, 200 MSPS (clocked at 153.6 MSPS) |
| **Frequency range** | 9 kHz -- 72 MHz (VLF through 4m band) |
| **Nyquist bandwidth** | 76.8 MHz |
| **Dynamic range** | >100 dB (dual DSA cascade) |
| **SFDR** | 100 dB (typical) |
| **Noise figure** | 3.0 dB (LTC6432 dominated) |
| **Output IP3** | +33 dBm (at ADC driver output) |
| **Clock jitter** | <50 fs RMS (Si5391B, integer mode) |
| **Gain range** | -63.5 dB to +15.2 dB (0.25 dB steps) |
| **Filter bands** | 7 switched BPF + wideband master LPF |
| **ADC interface** | LVDS differential (17 pairs, DDR @ 153.6 MHz) |
| **RX/TX switching** | Atomic, single-pin, <100 ns |
| **FPGA platform** | ALINX AXU2CGB (Xilinx Zynq UltraScale+) |

---

## Why This Project?

### 16-bit at 153.6 MSPS -- wider Nyquist than LTC2208 designs

Most open-source direct-sampling SDRs use the LTC2208 (130 MSPS). The LTC2209 pushes the sample rate to 153.6 MSPS, extending the Nyquist bandwidth to 76.8 MHz and comfortably covering the entire 9 kHz -- 72 MHz range without aliasing concerns at the band edges.

### Dual Digital Step Attenuators with 0.25 dB resolution

Two PE43711A DSAs in cascade provide 63.5 dB of digitally controlled attenuation in 0.25 dB steps. This is finer than most SDR designs (which use 0.5 dB or 1 dB steps) and enables precise AGC control from the FPGA without sacrificing signal quality.

### LTC6432 combined LNA/ADC driver -- a unique topology

Instead of the traditional separate-LNA-plus-ADC-driver chain (PGA-103 + ADA4937 or similar), this design uses the LTC6432-15 as a combined low-noise amplifier and differential ADC driver. The result is a simpler signal path with +15.2 dB fixed gain, 3.0 dB noise figure, and +33 dBm OIP3 -- all in a single IC with dual-balun impedance matching.

### 7-band switched filter bank + wideband master LPF

Two PE42582A SP8T switches route the signal through one of seven optimized bandpass filters (90 kHz -- 73 MHz) or a wideband master LPF covering the full 9 kHz -- 72 MHz range. The master LPF provides a panoramic view of the entire HF spectrum for digital waterfall display, while the BPFs deliver sharp selectivity when zoomed into a specific band.

### Si5391B integer-mode clock -- zero fractional spurs

The clock synthesizer operates in pure integer mode: 38.4 MHz TCXO x4 = 153.6 MHz. No fractional dividers means zero fractional spurs on the SDR waterfall -- a real advantage over designs using fractional-N PLLs with 25 MHz references. Clock jitter is below 50 fs RMS, preserving the full 16-bit ADC performance.

### Full transmit path with 16-bit DAC

The MAX5885 16-bit DAC shares the same signal chain (filter bank, LTC6432, and RF switches) in reverse for TX. Atomic RX/TX switching is controlled by a single FPGA pin with <100 ns transition time -- no sequencing logic required.

### Extended range: 9 kHz -- 72 MHz

Coverage extends from VLF (9 kHz) through the entire HF spectrum and up to the 4-meter band (70 MHz). The 3-stage RF choke network in the LNA power supply is specifically designed for continuous performance from 9 kHz to 72 MHz without resonant dips.

---

## Comparison with Existing SDR Platforms

| Parameter | **BitForge SDR** | Hermes-Lite 2 | HPSDR Angelia | RX-888 MKII | Red Pitaya SDRlab 122-16 | Elad FDM-S3 |
|---|---|---|---|---|---|---|
| **ADC resolution** | 16-bit | 12-bit | 16-bit | 16-bit | 16-bit | 16-bit |
| **Sample rate** | 153.6 MSPS | 76.8 MSPS | 122.88 MSPS | 64 MSPS | 122.88 MSPS | 192 MSPS |
| **SFDR** | 100 dB | ~70 dB | 100 dB | 78 dB | 90 dB | 100 dB |
| **Dynamic range** | >100 dB | ~80 dB | >100 dB | ~85 dB | ~90 dB | >100 dB |
| **Frequency range** | 9 kHz -- 72 MHz | 0 -- 38.4 MHz | 0 -- 61.44 MHz | 0 -- 32 MHz | 0 -- 60 MHz | 9 kHz -- 108 MHz |
| **TX capability** | Yes (16-bit DAC) | Yes (12-bit) | Yes | No | Yes (14-bit) | No |
| **Filter bands** | 7 BPF + master LPF | 5 LPF | 7 BPF | None | None | 6 BPF |
| **Clock jitter** | <50 fs | Not specified | Not specified | Not specified | Not specified | Not specified |
| **Gain control step** | 0.25 dB | 1 dB | 0.5 dB | 0.5 dB | Analog | 0.5 dB |
| **ADC driver** | LTC6432 (LNA+driver) | ADC direct | ADA4937 | LTC6401 | ADC direct | Proprietary |
| **License** | CERN-OHL-S v2 | GPL | GPL | Closed | Closed | Closed |
| **Approx. price** | ~$150 (PCB+BOM) | ~$300 | ~$800+ | ~$200 | ~$500 | ~$1,200 |

---

## Architecture

```
                            ┌─────────────────────────────────┐
                            │        FPGA (Zynq US+)          │
                            │   ALINX AXU2CGB via J701/J901  │
                            │                                 │
                            │  J701: ADC LVDS (17 diff pairs) │
                            │  J901: DAC data + control (3.3V)│
                            └──┬──────────────┬───────────────┘
                               │              │
                          ADC LVDS        DAC data + /RX_TX_MODE
                               │              │
┌─────────┐   ┌─────────┐   ┌─┴──────┐   ┌───┴──────┐
│ SMA RX  │   │Block 1  │   │Block 7 │   │ Block 9  │
│  Input  ├──►│DSA1     │   │LTC2209 │   │ MAX5885  │
│  (J101) │   │0-31.75dB│   │ADC     │   │ DAC      │
└─────────┘   └────┬────┘   └───▲────┘   └────┬─────┘
                   │             │              │
              ┌────▼─────────────┴──────────────┴────┐
              │          Block 4: RF Switches         │
              │       3x F2912 (atomic RX/TX)         │
              │  RX: Antenna→ADC  TX: DAC→Antenna     │
              └────────────────┬──────────────────────┘
                               │
                   ┌───────────▼───────────┐
                   │  Block 6: LTC6432     │
                   │  LNA/ADC Driver       │
                   │  +15.2 dB, NF=3.0 dB  │
                   │  Dual balun topology   │
                   └───────────▲───────────┘
                               │
                   ┌───────────┴───────────┐
                   │  Block 5: DSA2        │
                   │  0 -- 31.75 dB        │
                   │  0.25 dB steps        │
                   └───────────▲───────────┘
                               │
              ┌────────────────┴────────────────┐
              │  Blocks 2-3: Switched Filter Bank│
              │  7x BPF + Master LPF (0-72 MHz) │
              │  2x PE42582A SP8T switches       │
              └─────────────────────────────────┘

              ┌─────────────────────────────────┐
              │  Block 8: Clock Generation       │
              │  Si5391B (<50 fs jitter)         │
              │  38.4 MHz TCXO x4 = 153.6 MHz   │
              │  Integer mode, zero spurs        │
              └─────────────────────────────────┘
```

**Signal flow (RX):** Antenna → ESD protection → Balun → DSA1 → Filter Bank → DSA2 → LTC6432 → RF switches → LTC2209 ADC → LVDS → FPGA

**Signal flow (TX):** FPGA → MAX5885 DAC → RF switches → Filter Bank → DSA2 → LTC6432 → RF switches → Antenna

---

## Repository Contents

| Path | Description |
|---|---|
| `README.md` | This file |
| `LICENSE` | CERN Open Hardware Licence v2 -- Strongly Reciprocal |
| `BitForge SDR_logo_invert.png` | Project logo |
| `PROJECT_OVERVIEW_v2.0.md` | Detailed technical overview of all 9 functional blocks |
| `ALINX_Interface_Pinout_Specification.md` | FPGA carrier board connector pinout mapping |
| `FPGA_Interface_Voltage_Analysis.md` | Voltage level and bank compatibility analysis |
| `pcb_design/PCB_STACKUP_IMPEDANCE.md` | PCB stackup and controlled impedance specifications |
| `schematics/All_Bloks_of_project.kicad_sch` | Top-level KiCad schematic (all blocks) |
| `schematics/All_Bloks_of_project.kicad_pcb` | PCB layout |
| `schematics/All_Bloks_of_project.kicad_pro` | KiCad project file |
| `schematics/All_Bloks_of_project.kicad_dru` | Design rules |
| `schematics/production/LTC2209_SDR.zip` | Production Gerber files |
| `schematics/production/LTC2209_SDR_bom.csv` | Bill of Materials |
| `schematics/bom/ibom.html` | Interactive BOM (open in browser) |

---

## Getting Started

### Prerequisites

- **KiCad 8.0** or later -- [https://www.kicad.org/download/](https://www.kicad.org/download/)
- A web browser (for the interactive BOM)

### Viewing the Design

1. **Schematics:** Open `schematics/All_Bloks_of_project.kicad_pro` in KiCad. The top-level schematic contains all 9 functional blocks with hierarchical sheets.

2. **PCB Layout:** Open the same project file and switch to the PCB editor, or open `schematics/All_Bloks_of_project.kicad_pcb` directly.

3. **Interactive BOM:** Open `schematics/bom/ibom.html` in any web browser. This provides a visual, searchable component placement overlay on the PCB -- no KiCad installation required.

4. **Production files:** The `schematics/production/` directory contains ready-to-order Gerber files and BOM.

### FPGA Platform

This board connects to the **ALINX AXU2CGB** carrier board (Xilinx Zynq UltraScale+ XCZU2CG) via two 2x20 pin headers:
- **J701** (mapped to ALINX J12): ADC LVDS interface -- 16 data pairs + 1 clock pair
- **J901** (mapped to ALINX J15): DAC data bus + control signals (3.3V LVCMOS)

See `ALINX_Interface_Pinout_Specification.md` for the complete pin mapping.

---

## License

This project is licensed under the **CERN Open Hardware Licence Version 2 -- Strongly Reciprocal** ([CERN-OHL-S v2](https://ohwr.org/cern_ohl_s_v2.txt)).

See the [LICENSE](LICENSE) file for the full text.
