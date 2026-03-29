# SDR Receiver/Transceiver Project - Technical Overview

**Version:** 2.9
**Date:** 2026-01-25
**Status:** RX/TX Complete (LVDS interface, 3.3V control, optimized Block 6)

---

## PROJECT SUMMARY

**Name:** Software Defined Radio (SDR) Receiver with LTC2209 ADC
**ADC:** LTC2209 (16-bit, 160 MSPS)
**DAC:** MAX5885 (16-bit, 200 MSPS)
**Frequency Range:** 9 kHz - 72 MHz
**Dynamic Range:** >100 dB (with dual DSA)
**Gain:** 15.2 dB (fixed, LTC6432 LNA/Driver)

---

## BLOCK STRUCTURE

The project consists of **9 functional blocks** with continuous component numbering:

| Block          | Prefix | Function             | Key ICs            | Purpose                                                    |
| -------------- | ------ | -------------------- | ------------------ | ---------------------------------------------------------- |
| **Block 1**    | 100s   | Input Stage          | PE43711A-Z, TC1-1T | Input protection, balancing, DSA1                          |
| **Block 2, 3** | 200s   | Switched Filter Bank | 2x PE42582A-X      | Multi-band filtering (7 bands) + Master LPF 0 - 72 MHz    |
| **Block 4**    | 400s   | RX/TX RF Switching   | 3x F2912NCGI8      | RF switching between RX and TX modes                       |
| **Block 5**    | 500s   | Second DSA           | PE43711A-Z         | Additional digital attenuation                             |
| **Block 6**    | 600s   | LNA/ADC Driver       | LTC6432AIUF-15     | LNA + differential driver (+15.2dB) for RX/TX              |
| **Block 7**    | 700s   | ADC + Interface      | LTC2209, TPS7A9101 | 16-bit ADC + FPGA interface (RX)                           |
| **Block 8**    | 800s   | Clock Generation     | Si5391B-A-GM       | Ultra-low jitter clock (38.4 MHz TCXO -> 153.6 MHz)       |
| **Block 9**    | 900s   | DAC & TX Interface   | MAX5885, LP5912    | 16-bit DAC + FPGA interface (TX)                           |

---

## DETAILED BLOCK DESCRIPTIONS

### **BLOCK 1: Input Stage & DSA1** (100s prefix)

#### **Key Components:**

- **IC101** - LP5907MFX-3.3: Ultra-low noise LDO (3.3V, 250mA)
- **IC104** - PE43711A-Z: 6-bit Digital Step Attenuator (0-31.75 dB, 0.25dB steps)
- **T101** - TC1-1T: 1:1 RF Transformer (50 ohm impedance matching)
- **TVS101** - CDSOD323-T05LC: Transient Voltage Suppressor
- **J101** - SMA RX: Input coaxial connector

#### **Functions:**

1. **ESD/Overvoltage Protection** (TVS101)
2. **Single-ended to Differential Conversion** (T101)
3. **Digital Control Gain** (IC104): 0 to -31.75 dB, FPGA controlled
4. **Clean Power Supply** (IC101) for DSA

#### **Control:**

- **DSA_SCLK, DSA_DATA, DSA_LE1**: SPI control for DSA1 (7-bit attenuation values)

#### **Specifications:**

- **Input impedance:** 50 ohm
- **IP3:** +42 dBm (PE43711A)
- **Insertion Loss:** ~3 dB @ 0 dB attenuation

---

### **BLOCK 2: Switched Filter Bank** (200s prefix)

#### **Key Components:**

- **IC201** - PE42582A-X: SP8T RF Switch (input)
- **IC202** - LP5907MFX-3.3: LDO for switch power supply
- **IC203** - PE42582A-X: SP8T RF Switch (output)
- **L301-L356, C301-C374**: 7x LC Band-Pass Filters
- **Master LPF:** Wideband filter covering the full dynamic range 9 kHz - 72 MHz. Designed to provide a complete passband overview with subsequent digital processing and filtering in the FPGA

#### **Architecture:**

```
IC201 (SP8T) -> [7x Bandpass Filters and Master LPF 9 kHz - 72 MHz] -> IC203 (SP8T)
     |                                        |
  FLT_V1-V4                               FLT_V1-V4
  (4-bit binary)                          (4-bit binary)
```

#### **Filters (7 bands):**

| Band | Filter Path       | Frequency Range | Components           |
| ---- | ----------------- | --------------- | -------------------- |
| 1    | RF_FILTR_IN/OUT_1 | 0.09 - 1.6 MHz  | L301-L305, C301-C305 |
| 2    | RF_FILTR_IN/OUT_2 | 1.5 - 4 MHz     | L306-L312, C306-C315 |
| 3    | RF_FILTR_IN/OUT_3 | 4 - 8 MHz       | L313-L319, C316-C327 |
| 4    | RF_FILTR_IN/OUT_4 | 8 - 15.5 MHz    | L320-L327, C328-C341 |
| 5    | RF_FILTR_IN/OUT_5 | 15 - 30 MHz     | L328-L333, C342-C351 |
| 6    | RF_FILTR_IN/OUT_6 | 30 - 50 MHz     | L334-L337, C352-C358 |
| 7    | RF_FILTR_IN/OUT_7 | 47 - 73 MHz     | L338-L347, C359-C374 |
| 8    | MASTER_LPF_IN/OUT | 0.09 - 72 MHz   | L338-L347, C359-C374 |

#### **Master LPF:**

- **MASTER_LPF_IN** -> L348-L356 -> **MASTER_LPF_OUT**
- Wideband filter covering the full dynamic range 9 kHz - 72 MHz.

#### **Control:**

- **FLT_V1, FLT_V2, FLT_V3, FLT_V4**: 4-bit binary control for both switches (synchronous switching)
  Band selection is performed programmatically. The user initially sees the full 9 kHz - 72 MHz band. This allows them to view the radio spectrum and choose software tools and filters for signal processing. When the user zooms into a specific range, the corresponding BPF filter is selected, providing fine signal filtering in that band.

#### **Functions:**

####

#### **Specifications:**

- **Isolation:** >55 dB (PE42582A)
- **Insertion Loss:** ~0.5 dB per switch
- **Switching Speed:** <500 ns

---

### **BLOCK 4: RX/TX RF Switching** (400s prefix)

#### **Key Components:**

- **IC401, IC402, IC403** - F2912NCGI8: SP2T RF switches (Renesas)
- **IC404, IC405** - LP5907MFX-3.3: LDO for switch power supply

#### **Functions:**

- **IC401:** TX signal injection from DAC into shared RX/TX signal path (Filter Bank)
- **IC402/IC403:** Routing of LTC6432 differential outputs between ADC (RX) and TX antenna
- **1-pin control:** All switches + ADC/DAC are controlled by a single `/RX_TX_MODE` signal

#### **Architecture:**

```
        RX Mode (/RX_TX_MODE = LOW):
Antenna -> Filter Bank -> LTC6432 -> IC402/IC403 (RF1) -> ADC (active)
                      (DAC shutdown)

        TX Mode (/RX_TX_MODE = HIGH):
DAC (active) -> IC401 (RF2) -> Filter Bank -> LTC6432 -> IC402/IC403 (RF2) -> TX Antenna
                                                    (ADC shutdown)
```

#### **Atomic RX/TX Control (single signal):**

**Switch Control Truth Table:**

| Mode   | /RX_TX_MODE | F2912 (IC401-403) | LTC2209 SHDN | MAX5885 PD   |
| ------ | ----------- | ----------------- | ------------ | ------------ |
| **RX** | LOW (0V)    | RF1 ON (RX path)  | LOW (active) | HIGH (off)   |
| **TX** | HIGH (3.3V) | RF2 ON (TX path)  | HIGH (off)   | LOW (active) |

**Control Signal Path:**

```
FPGA (J15, BANK25/26, 3.3V) -> J901.32 -> RX_TX_MODE
                                          +-- IC401.16 (CTL2) -> F2912 #1
                                          +-- IC402.16 (CTL2) -> F2912 #2
                                          +-- IC403.16 (CTL2) -> F2912 #3
                                          +-- IC701.19 (SHDN) -> LTC2209
```

**Control Current Budget (FPGA 3.3V LVCMOS output):**

| Component       | Pins          | Current per pin | Total       |
| --------------- | ------------- | --------------- | ----------- |
| F2912 CTL2 (x3) | IC401-403.16  | 500 nA max      | **1.5 uA**  |
| LTC2209 SHDN    | IC701.19      | ~1 uA           | **1 uA**    |
| U902 input      | Level shifter | ~1 uA           | **1 uA**    |
| **Total**       |               |                 | **~3.5 uA** |

**FPGA IOH/IOL capability: 8-12 mA -> margin >2000x**

#### **F2912 Configuration (3.3V Control Mode):**

- **Pin 18 (LOGICCTL)** = **GND** -> **3.3V logic mode enabled**
- **Pin 19 (ModeCTL)** = VCC -> 1-pin control mode
- **Pin 16 (CTL2)** = RX_TX_MODE -> Control input (3.3V logic)
- **Logic thresholds (3.3V):** VIH min = 2.0V, VIL max = 0.8V

**Advantage of LOGICCTL=GND:**

- Direct control via 3.3V signal from FPGA J15 (BANK25/26)
- Does not require level shifter IC704 (removed)
- Simplified circuit, fewer components

#### **Specifications:**

- **Switching time:** <100 ns (RF switches)
- **Isolation:** >74 dB (RF1/RF2 @ 1 GHz)
- **Insertion Loss:** <0.5 dB @ HF frequencies
- **Control:** Single FPGA pin `/RX_TX_MODE` (LVCMOS33, J901.32)

---

### **BLOCK 5: Second DSA** (500s prefix)

#### **Key Components:**

- **IC501** - LP5907MFX-3.3: LDO for DSA2
- **IC502** - PE43711A-Z: 6-bit Digital Step Attenuator

#### **Functions:**

- **Digital attenuation:** Additional digital attenuation (0 to -31.75 dB). Step size 0.25 dB
- **Dual DSA cascade** with Block 1 for wide dynamic range (0 to -63.5 dB total)

#### **Control:**

- **DSA_SCLK, DSA_DATA, DSA_LE2**: SPI control for DSA2 (7-bit attenuation values)

#### **Specifications:**

- Identical to IC104 (Block 1)
- **Total Gain Range:** 0 to -63 dB (with Block 1)

---

### **BLOCK 6: LNA/ADC Driver** (600s prefix) - **NEW DESIGN**

#### **Key Components:**

- **IC601** - **LTC6432AIUF-15#PBF**: Differential LNA/Driver (+15.2 dB gain)
- **IC602** - TPS7A9401DSCR: Ultra-low noise LDO
- **T601** - TTWB-2-A: 1:2 Input Balun (single-ended to differential), Bandwidth = 50 kHz to 200 MHz
- **T602** - CX2041NLT: 1:1 Output Balun (re-balancing + common-mode biasing), Bandwidth = 50 kHz to 200 MHz
- **L601, L611, L602, L603, L612, L604**: Power supply chokes + output filter inductors
- **C601-C622**: Coupling, bypass, and filter capacitors
- **R601-R608**: Input termination, biasing, output termination

#### **Architecture:**

```
RF_1 -> T601 (1:2 balun) -> IC601 (LTC6432) -> T602 (2:1 balun) -> ADC
       50 ohm single-ended     Diff. Amp         Output balancing   Differential
                           Gain = +15.2dB                           to LTC2209
                                  |
                            IC602 (5V LDO)
```

#### **Pinout LTC6432 (IC601):**

| Pin                      | Function | Connection                  |
| ------------------------ | -------- | --------------------------- |
| 5                        | NFILT2   | C605 (1uF) filter cap       |
| 6                        | NFILT1   | C606 (1uF) filter cap       |
| 7                        | -In      | From T601 via C602           |
| 8, 15, 17, 23, 25        | GND      | Ground plane                |
| 9, 22                    | +VS      | +5V (from IC602)            |
| 12                       | -FDBK    | C608 (1uF) feedback         |
| 13                       | -OUT     | To T602 via C614             |
| 18                       | +OUT     | To T602 via C613             |
| 19                       | +FDBK    | C607 (1uF) feedback         |
| 24                       | +In      | From T601 via C601           |
| 1-4, 10-11, 14-16, 20-21 | DNC      | Do Not Connect              |

#### **Input Network:**

```
RF_1 (from Block 5) -> T601 (TTWB-2-A, 1:2) -> +/-In (IC601)
                      50 ohm SE -> 200 ohm diff      Pins 24, 7
```

#### **Output Network:**

```
IC601 +/-OUT -> C613/C614 (1uF) --+-- R622/R623 (2.2k ohm shunt)
                               +-- T602 (CX2041NLT, 1:1) -> ADC differential input
                                   Center tap -> LTC2209 VCM
```

#### **Power Supply: 3-Stage RF Choke Network**

Three-stage RF choke network for continuous coverage of the 9 kHz - 72 MHz range:

```
+5V_B6 -+- L601 (470uH) -+- L611 (47uH) -+- L602 (560nH) -+- IC601 +VS
        |                |               |                |
        +- R603 (390 ohm) -+- R620 (82 ohm) -+- R604 (54 ohm) ---+
```

| Stage | Inductor     | De-Qing R    | Effective Range |
| ----- | ------------ | ------------ | --------------- |
| LF    | L601 (470uH) | R603 (390 ohm) | 9 kHz - 1 MHz   |
| MF    | L611 (47uH)  | R620 (82 ohm)  | 500 kHz - 6 MHz |
| HF    | L602 (560nH) | R604 (54 ohm)  | 5 MHz - 72 MHz  |

**Specifications:**

- Total DCR: 1.86 ohm -> DC drop: 130 mV -> VCC = 4.87V (within specification)
- Z_choke @ 72 MHz: 516 ohm
- Worst resonance: Q=72 @ 213 kHz (well damped)

#### **LTC6432 Specifications:**

- **Gain:** +15.2 dB (fixed)
- **Noise Figure:** 3.0 dB @ 100 MHz
- **Output IP3:** +33 dBm
- **Bandwidth:** 100 kHz to 1400 MHz (-3dB)
- **Output Impedance:** 50 ohm differential
- **Supply Current:** 165 mA @ 5V
- **DC Power:** 850mW

#### **Block Functions:**

1. **Low Noise Amplification:** +15.2 dB with NF = 3.0 dB
2. **Single-ended to Differential Conversion** (T601)
3. **Differential Output Balancing** (T602)
4. **ADC Common-Mode Biasing** (T602 center tap -> VCM)
5. **Output Filtering** (L605/L606 + C617/C618/C619)
6. **Clean Power Supply** (IC602 ultra-low noise LDO)

#### **Key Features:**

- **Dual Balun Architecture:** T601 (TTWB-2-A, 1:2) + T602 (CX2041NLT, 1:1), BW = 50 kHz - 200 MHz
- **3-Stage RF Choke:** Continuous coverage 9 kHz - 72 MHz without resonant dips
- **Optimized De-Qing:** R603=390 ohm, R620=82 ohm, R604=54 ohm for Q-factor control
- **Shunt Resistors:** R622/R623 (2.2k ohm) for additional LC resonance damping
- **T602 VCM Biasing:** Center tap connected to LTC2209 VCM for ADC DC biasing

---

### **BLOCK 7: ADC & FPGA Interface** (700s prefix)

#### **Key Components:**

- **IC701** - LTC2209IUPPBF: 16-bit ADC (153.6 MSPS)
- **IC702** - TPS7A9101DSKT: Precision LDO for analog supply (3.3V)
- **IC703** - LP5907MFX-1.8: LDO for digital outputs (1.8V)
- **J701** - Pin Header 2x20: FPGA interface connector (LVDS data)
- **J901** - Pin Header 2x20: Control signals + ADC_OF (3.3V)
- **C701** - 10uF: VCM bypass capacitor
- **C702, C711, C712** - Decoupling capacitors for VDD (3.3V)
- **C703, C718** - Decoupling capacitors for OVDD (1.8V)

#### **ADC Specifications:**

- **Resolution:** 16 bits
- **Sample Rate:** 153.6 MSPS (from Block 8)
- **Nyquist Bandwidth:** 76.8 MHz (covers 9 kHz - 72 MHz range)
- **Noise Floor:** 77.3 dBFS
- **SFDR:** 100 dB (typical)
- **Output Format:** 2's Complement (16-bit parallel)
- **Output Mode:** **LVDS Differential** (17 pairs via J701)
- **Power Dissipation:** ~1.0W @ 153.6 MSPS

#### **Output Mode: LVDS Differential**

**Why LVDS:**

- **High speed:** DDR at 153.6 MHz = 307.2 Mbps per lane
- **Noise immunity:** Differential signals with common-mode rejection
- **Compatibility:** FPGA HP banks (BANK65/66) have native LVDS support
- **Termination:** Built-in 100 ohm resistors in FPGA (DIFF_TERM_ADV TERM_100)

**LTC2209 LVDS Output Structure:**

- **DA0-DA15:** 8 differential pairs (Bus A)
- **DB0-DB15:** 8 differential pairs (Bus B)
- **CLKOUTA/CLKOUTB:** 1 differential pair (Data Clock Output)
- **OFA/OF-:** Overflow -> J901 pins 21/22 (single-ended 3.3V)
- **Total LVDS pairs on J701:** 17 (all J12 pairs utilized)

**LVDS Electrical Characteristics:**

| Parameter              | Value                      |
| ---------------------- | -------------------------- |
| Common Mode (VOS)      | 1.125V - 1.2V - 1.375V     |
| Differential (VOD)     | 247mV - 350mV - 454mV      |
| FPGA Input Common Mode | 0.3V - 1.5V (compatible)   |
| FPGA Min Differential  | >100mV (350mV >> 100mV OK) |
| Termination            | 100 ohm internal in FPGA   |

#### **Control Pins Configuration:**

| Pin | Name  | Connected  | Function                | Reasoning                             |
| --- | ----- | ---------- | ----------------------- | ------------------------------------- |
| 1   | SENSE | VDD (3.3V) | Internal 2.5V reference | Uses internal bandgap reference       |
| 19  | SHDN  | RX_TX_MODE | Shutdown control        | 3.3V from FPGA via J901.32            |
| 20  | DITH  | GND        | Dither disabled         | Better SNR for SDR (~3dB improvement) |
| 61  | LVDS  | VDD (3.3V) | LVDS mode enabled       | Differential output via J701          |
| 62  | MODE  | VDD (3.3V) | 2's Complement format   | Standard for DSP/SDR processing       |
| 63  | RAND  | GND        | Normal output           | No XOR randomization needed           |
| 64  | PGA   | GND        | Gain=1, 2.25Vpp range   | LTC6432 provides sufficient gain      |

#### **Dual Supply Architecture:**

- **VDD (3.3V):** Analog core supply (pins 5,6,15,16,17)
  - Source: IC702 (TPS7A9101) ultra-low noise LDO
  - Decoupling: C702 (1uF), C711 (47uF), C712 (10nF)

- **OVDD (1.8V):** Digital output drivers (pins 32,49)
  - Source: IC703 (LP5907-1.8) ultra-low noise LDO
  - Decoupling: C703 (1uF), C718 (2.2uF)
  - Powers LVDS output drivers

- **VCM (Pin 3):** Output voltage reference (0.95V typical)
  - Bypass: C701 (10uF, >=2.2uF per datasheet)
  - Connected to T602 center tap for DC bias

#### **Analog Inputs:**

- **IN+ (Pin 8), IN- (Pin 9):** Differential analog inputs
  - Connected to ADC_P_OUT, ADC_N_OUT (from Block 6 via T602)
  - Input range: 2.25Vp-p differential (PGA=0)
  - Series termination: R607/R608 (25 ohm each)

#### **Clock Input:**

- **ENC+ (Pin 12), ENC- (Pin 13):** Differential clock inputs
  - Connected to CLK_0, CLK_B (from Block 8)
  - Clock frequency: 153.6 MHz (from Si5391B OUT0/OUT0b)
  - AC coupled via C809, C818 (100nF)
  - Terminated: R701 (100 ohm differential)

#### **Digital Outputs (LVDS to FPGA via J701):**

- **DA0-DA15, DB0-DB15:** 16 differential pairs (32 lines)
  - Output standard: LVDS (VOS=1.2V, VOD=350mV)
  - Data format: 2's Complement (MODE pin = VDD)
  - DDR transfer: both edges of CLKOUT
  - Connected to J701 pins 3-36 -> FPGA J12 (BANK65/66)

- **CLKOUTA/CLKOUTB (Pins 41, 40):** Data clock output
  - LVDS differential pair
  - Connected to J701 pins 15/16 -> FPGA IO1_7 (GC-capable)
  - Frequency: 153.6 MHz (same as ENC clock)

- **OFA/OF- (Pins 60, 59):** Overflow indicator
  - Connected to J901 pins 21/22 -> FPGA BANK25/26 (3.3V)
  - Used as single-ended signals on HR banks

---

### **BLOCK 8: Clock Generation** (800s prefix)

#### **Key Components:**

- **IC801** - Si5391B-A-GM: Ultra-low jitter clock generator (QFN-65, 12 outputs)
- **X801** - 38.4 MHz TCXO: Temperature-compensated crystal oscillator
- **C801-C817** - Decoupling capacitors: Power supply filtering
- **R801, R802** - 2.2k ohm: I2C pull-up resistors

#### **Architecture:**

```
X801 (38.4 MHz TCXO) -> IC801 (Si5391B PLL) -> OUT0/OUT0b (153.6 MHz) -> LTC2209 CLK+/-
                         ^
                    I2C_SCL, I2C_SDA
                    (FPGA configuration)
```

#### **Clock Generation:**

```
38.4 MHz TCXO (X801)
    | XA/XB (pins 8/9)
Si5391B DSPLL (IC801)
    +- VCO: ~5 GHz (internal)
    +- Integer Mode (clean spectrum, no fractional spurs)
    +- MultiSynth dividers
         |
OUT0/OUT0b (pins 24/23) -> 153.6 MHz differential -> LTC2209 CLK+/- (IC701.12/13)
```

#### **I2C Interface:**

- **I2C_SEL (pin 39)** = +1.8V -> I2C mode
- **SCLK (pin 16)** -> FPGA I2C_SCL (with 2.2k ohm pull-up)
- **SDA (pin 18)** -> FPGA I2C_SDA (with 2.2k ohm pull-up)
- **A1/A0 (pins 17/19)** = GND -> I2C address **0x68**

#### **Input Selection:**

- **IN_SEL0/IN_SEL1 (pins 3/4)** = GND -> explicit XA/XB selection (crystal input)
- **XA/XB (pins 8/9)** -> connected to 38.4 MHz TCXO (X801)

#### **Si5391B Specifications for Project Requirements:**

- **Jitter:** <50 fs RMS (Integer Mode, Precision Calibration)
- **Output frequency:** 153.6 MHz (4/1 ratio - Integer Mode) (programmable via I2C)
- **Sample rate:** 153.6 MSPS
- **Nyquist bandwidth:** 76.8 MHz
- **Coverage:** 9 kHz - 72 MHz with margin
- **Reference:** 38.4 MHz TCXO
- **Mode:** Integer (38.4 MHz x 4/1 = 153.6 MHz) -> no fractional spurs
- **Usable bandwidth:** 0-68 MHz (with margin up to 72 MHz)
- **Filter transition band:** 68-76.8 MHz (~12% margin)
- **Data throughput:** 153.6 MSPS x 16 bit = 2.46 Gbit/s
- **Output format:** Differential LVDS/LVPECL (configurable)
- **Power:** 1.8V (VDD core) + 3.3V (VDDA analog)

#### **Clock Performance:**

**Jitter budget (for LTC2209 16-bit ADC):**

```
ADC SNR = -20 x log10(2*pi * f_sample * t_jitter)

@ 153.6 MSPS, 153.6 MHz clock:
  Si5391B: <50 fs RMS
  -> Jitter-limited SNR: ~94 dBFS
  -> ENOB: ~15.3 bits (theoretical)
  -> LTC2209 spec: 77.3 dBFS noise floor (16-bit ENOB achievable!)
```

#### **Integer Mode Advantage:**

**Fractional-N vs Integer Mode:**

- **Fractional:** 25 MHz x 6.144 = 153.6 MHz -> produces spectral spurs (birdies on SDR waterfall)
- **Integer:** 38.4 MHz x 4 = 153.6 MHz -> clean spectrum, ideal for SDR

**Integer Mode calculation:**

```
38.4 MHz x 4 = 153.6 MHz
-> Simple integer multiplier (4x)
-> No fractional dividers needed
-> Zero fractional spurs
```

#### **Block Functions:**

1. **Ultra-low jitter clock generation** (<50 fs RMS)
2. **Integer Mode operation** (clean spectrum without fractional spurs)
3. **Dual LDO power supply** (analog/digital power isolation)
4. **I2C programmability** (ClockBuilder Pro configuration)
5. **Input filtering** (ferrite beads + decoupling caps)
6. **Temperature-stable reference** (38.4 MHz TCXO)

#### **Control:**

- **I2C_SCL, I2C_SDA:** Si5391B programming (address 0x68)
- **Configuration:** Via ClockBuilder Pro software (Silicon Labs)
- **Startup:** Configuration loaded via I2C after power-on
  I2C_SCL / I2C_SDA are REQUIRED in the current design because:
  1. Standard Si5391B-A-GM is used - no custom NVM
  2. At each power-up the FPGA must load the configuration:
  - Input: 38.4 MHz TCXO on XA/XB
  - Output: 153.6 MHz LVDS on OUT0/OUT0b
  - Mode: Integer (no fractional spurs)
  - PLL settings: 38.4 x 4 = 153.6 MHz
  3. Without I2C programming, the Si5391B will operate with factory default settings
  Alternative (for future revisions):
  Order a factory pre-programmed Si5391B through ClockBuilder Pro:
  - Silicon Labs will assign a unique part number (Si5391B-Axxxxx-GM)
  - The IC will automatically start with the correct configuration
  - I2C can then be removed or kept only for monitoring

#### **FPGA Interface:**

**Connection via I2C:**

```
FPGA (J12) -> J701 -> Si5391B
  Pin 34 (IO1_16P, C8) -> I2C_SCL -> IC801.16 (with 2.2k ohm pull-up)
  Pin 35 (IO1_17N, A8) -> I2C_SDA -> IC801.18 (with 2.2k ohm pull-up)
```

**Configuration:**

- **I2C Address:** 0x68 (7-bit, A1=GND, A0=GND)
- **Clock speed:** 400 kHz (Fast Mode)
- **Config size:** ~300 bytes (ClockBuilder Pro generated)
- **Init time:** ~25-40 ms (once at boot)

**Initialization Sequence:**

1. Power-on -> POR (~10 ms)
2. FPGA I2C write (~5-10 ms) -> config registers
3. PLL lock (~10-20 ms)
4. Clock ready (153.6 MHz)

#### **Power Architecture (CRITICAL FEATURE!):**

**Always-on power supply (independent of RX/TX mode):**

```
+3.3V (always on) -> Si5391B
```

**Why always on:**

- TX path (MAX5885 DAC) also requires a clock
- RX and TX share a single clock source
- Configuration loaded once at boot (no re-init required after mode switching)
- PLL always locked -> stable clock

**Power Consumption:**

- **Si5391B:** ~165 mA @ 1.8V/3.3V -> **~357 mW** (always on)
- **For comparison:** LTC6432 = 850 mW, TX PA = 5-10W
- **Conclusion:** Negligible power consumption, but critical advantage in switching speed!

---

### **BLOCK 9: DAC & TX Interface** (900s prefix)

#### **Key Components:**

- **IC901** - MAX5885EGM+D: 16-bit DAC (153.6 MSPS, TX path)
- **U901** - TPS7A9401DSCR: Ultra-low noise LDO (3.3V, 0.46 uVRMS)
- **U902** - SN74LV1T04DCKR: Inverter for PD control
- **T902** - RF Transformer (center-tapped balun)
- **R903, R904** - 25 ohm: DC return path resistors
- **R901** - 2k ohm: Full-scale current set resistor
- **C915/C916, C917/C918** - Parallel decoupling (1uF || 100nF)

#### **DAC Specifications:**

- **Resolution:** 16 bits, 200 MSPS
- **Update Rate:** 153.6 MSPS (from Block 8, OUT3/OUT3b)
- **Nyquist Bandwidth:** 76.8 MHz (covers 9 kHz - 72 MHz)
- **Output Mode:** Differential current outputs (IOUTP, IOUTN)
- **Full-Scale Current:** 19.2 mA (with R901 = 2k ohm)
- **Output Format:** 2's Complement (16-bit parallel)

#### **Output Stage:**

```
IC901 Pin 19 (IOUTP) --+-- R904 (25 ohm) -> GND  (DC path)
                       +-- C917 (1uF)  |
                       +-- C918 (100nF)-+--> T902 Pin 3

IC901 Pin 18 (IOUTN) --+-- R903 (25 ohm) -> GND  (DC path)
                       +-- C915 (1uF)  |
                       +-- C916 (100nF)-+--> T902 Pin 1

T902 Center Tap --> C919 (100nF) --> GND
T902 Output --> RF Switches (shared RX/TX signal path)
```

**Parallel Decoupling:** 1uF || 100nF provides low impedance across the full 9 kHz - 72 MHz range

#### **TX Architecture:**

The TX signal uses the **shared signal path** with RX through:

- Block 2-3: Filter Bank (PE42582A switches)
- Block 5: DSA (bypass in TX mode)
- Block 6: LTC6432 amplifier
- RF switches control signal direction (RX/TX)

---

## SIGNAL FLOW

```
+-------------+
|   SMA RX    | RF Input (1-30 MHz)
|   (J101)    |
+------+------+
       | 50 ohm single-ended
+------v--------------------------------------------------------+
| BLOCK 1: Input Stage                                          |
|  TVS101 -> T101 (1:1) -> IC104 (DSA1: 0 to -31.75dB)        |
|                              ^ DSA_SCLK, DSA_DATA, DSA_LE1   |
|                              ^ +5V (always on)                |
+------+--------------------------------------------------------+
       | Differential
+------v--------------------------------------------------------+
| BLOCK 2: Switched Filter Bank                                 |
|  IC201 (SP8T) -> [7x BPF] -> IC203 (SP8T) -> Master LPF     |
|       ^ FLT_V1-V4            ^ FLT_V1-V4                     |
|       ^ +5V (always on)                                       |
+------+--------------------------------------------------------+
       | Filtered differential
+------v--------------------------------------------------------+
| BLOCK 5: Second DSA                                           |
|  IC502 (DSA2: 0 to -31.75dB)                                 |
|         ^ DSA_SCLK, DSA_DATA, DSA_LE2                        |
|         ^ +5V (always on)                                     |
+------+--------------------------------------------------------+
       | Attenuated differential
       | RF_1 (single-ended equivalent)
+------v--------------------------------------------------------+
| BLOCK 6: LNA/ADC Driver                                      |
|  T601 (1:2) -> IC601 (LTC6432, +15.2dB) -> T602 (1:1)       |
|               |                              |                |
|               IC602 (5V LDO)                VCM -> ADC        |
|               ^ +5V (always on)                               |
+------+----------------------------------------+---------------+
       | ADC_P_OUT, ADC_N_OUT (differential)     | ADC_VCM
       | 2Vp-p @ 200 ohm impedance               |
+------v-----------------------------------------v--------------+
| BLOCK 7: ADC & Interface                                      |
|  IC701 (LTC2209: 16-bit, 153.6 MSPS)                         |
|    +- ADC_CLK_IN <- Block 8 (153.6 MHz)                      |
|    +- ADC_D[15:0] + DCO -> FPGA (J701)                       |
|  IC702, IC703: Power supplies                                 |
|    ^ +5V (always on)                                          |
+------+-------------------+------------------------------------+
       |                   | ADC_CLK_IN
       |         +---------v------------------------------------------+
       |         | BLOCK 8: Clock Generation                          |
       |         |  X801 (38.4 MHz TCXO)                              |
       |         |    |                                               |
       |         |  IC801 (Si5391B: <50fs jitter)                     |
       |         |    | Integer Mode (no spurs)                       |
       |         |  OUT0/OUT0b -> 153.6 MHz diff.                     |
       |         |  U801/U802: Dual LDO (1.8V + 3.3V)                |
       |         |    ^ +5V (always on)                               |
       |         |    ^ I2C_SCL, I2C_SDA (FPGA config)                |
       |         +----------------------------------------------------+
       |
   +---v---------+                 +----------------------------------+
   |   FPGA      | <-/RX_TX_MODE--| BLOCK 4: RX/TX RF Switching      |
   |   (J701)    |   (atomic)     |  3x F2912NCGI8 (SP2T switches)   |
   +-------------+                |  + ADC SHDN + DAC PD control      |
                                  |  Current: ~3.5uA from FPGA pin    |
                                  +----------------------------------+
```

---

## POWER DISTRIBUTION

### **Supply Rails:**

| Rail              | Voltage | Source                | Load                        | Current |
| ----------------- | ------- | --------------------- | --------------------------- | ------- |
| **+5V**           | 5.0V    | External              | All blocks (always on)      | ~1.2A   |
| **+5V_OUT2**      | 5.0V    | IC602 (Block 6)       | IC601 (LTC6432)             | 50mA    |
| **+3.3V_CLK**     | 3.3V    | U802 (Block 8)        | IC801 VDDA (analog PLL)     | ~80mA   |
| **+1.8V_CLK**     | 1.8V    | U801 (Block 8)        | IC801 VDD + VDDO (digital)  | ~85mA   |
| **+3.3V_DIGITAL** | 3.3V    | IC702 (Block 7)       | IC701 digital I/O           | ~50mA   |
| **+3.3V_ANALOG**  | 3.3V    | LDOs (Blocks 1,2,4,5) | DSA + RF Switches           | ~180mA  |
| **+1.8V_ADC**     | 1.8V    | IC703 (Block 7)       | IC701 digital core          | ~40mA   |

### **Power Management:**

- **All blocks are always on** - no power switching

**ADC/DAC Control (single /RX_TX_MODE signal from FPGA):**

- **RX_TX_MODE** (3.3V LVCMOS via J901.32):
  - Direct connection to IC701.19 (LTC2209 SHDN)
  - Direct connection to F2912 switches (IC401-403.CTL2)
  - RX Mode: LOW -> SHDN=LOW (ADC active)
  - TX Mode: HIGH -> SHDN=HIGH (ADC shutdown)

- **U902** (SN74LV1T04): Inverter for DAC PD control
  - Input: /RX_TX_MODE (J901.32)
  - Output: IC901.10 (MAX5885 PD, inverted)
  - RX Mode: LOW -> PD=HIGH (DAC powered down)
  - TX Mode: HIGH -> PD=LOW (DAC active)

### **Power Architecture:**

- **All blocks are always on** - simplified architecture without power switching
- **Advantage:** Instant RX/TX switching via RF switches (<100 ns)
- **ADC/DAC control:** Shutdown via SHDN/PD pins instead of power supply switching
- **Power consumption:** ~1.2A @ 5V constant (TX and RX modes are identical)

---

## FPGA CONTROL INTERFACE

### **Control Signal Architecture:**

**DSA Control (SPI):**

- **Shared SPI bus** for DSA1 and DSA2 (IC104, IC502)
- **DSA_SCLK, DSA_DATA:** shared bus (clock + data)
- **DSA_LE1, DSA_LE2:** separate Latch Enable (chip select) for each DSA
- **Pins required:** 4 (saving 2 pins compared to separate SPI buses)

**Filter Control (Binary):**

- **4-bit binary control** for PE42582A-X switches (IC201, IC203)
- **FLT_V1-V4:** parallel control of both switches (synchronous switching)
- **Channels:** 16 possible (8 used: 7 BPF + Master LPF)

**System Control (atomic single-signal control):**

- **/RX_TX_MODE:** single control signal for RX/TX switching (J901.32, LVCMOS33)
  - LOW (0V) = RX mode (ADC active, DAC shutdown, RF1 path)
  - HIGH (3.3V) = TX mode (DAC active, ADC shutdown, RF2 path)
  - Controls: F2912 switches (IC401-403.CTL2) + LTC2209 SHDN (direct) + MAX5885 PD (U902)
  - Load current: ~3.5 uA (margin >2000x from FPGA IOH/IOL)

### **J701 Connector Pinout (LTC2209 LVDS Interface):**

J701 is now used **exclusively for the LVDS interface** with the LTC2209 ADC.

| Pin   | Signal    | Direction  | Description                |
| ----- | --------- | ---------- | -------------------------- |
| 1     | GND       | --         | Ground                     |
| 2     | +5V       | Power      | Main power supply          |
| 3-4   | DB0/DB1   | ADC -> FPGA | LVDS Data Bus B pair 0     |
| 5-6   | DB2/DB3   | ADC -> FPGA | LVDS Data Bus B pair 1     |
| 7-8   | DB4/DB5   | ADC -> FPGA | LVDS Data Bus B pair 2     |
| 9-10  | DB6/DB7   | ADC -> FPGA | LVDS Data Bus B pair 3     |
| 11-12 | DB8/DB9   | ADC -> FPGA | LVDS Data Bus B pair 4     |
| 13-14 | DB10/DB11 | ADC -> FPGA | LVDS Data Bus B pair 5     |
| 15-16 | CLKOUT+/- | ADC -> FPGA | **DCO (GC-capable IO1_7)** |
| 17-18 | DB12/DB13 | ADC -> FPGA | LVDS Data Bus B pair 6     |
| 19-20 | DB14/DB15 | ADC -> FPGA | LVDS Data Bus B pair 7     |
| 21-22 | DA0/DA1   | ADC -> FPGA | LVDS Data Bus A pair 0     |
| 23-24 | DA2/DA3   | ADC -> FPGA | LVDS Data Bus A pair 1     |
| 25-26 | DA4/DA5   | ADC -> FPGA | LVDS Data Bus A pair 2     |
| 27-28 | DA6/DA7   | ADC -> FPGA | LVDS Data Bus A pair 3     |
| 29-30 | DA8/DA9   | ADC -> FPGA | LVDS Data Bus A pair 4     |
| 31-32 | DA10/DA11 | ADC -> FPGA | LVDS Data Bus A pair 5     |
| 33-34 | DA12/DA13 | ADC -> FPGA | LVDS Data Bus A pair 6     |
| 35-36 | DA14/DA15 | ADC -> FPGA | LVDS Data Bus A pair 7     |
| 37-38 | GND       | --         | Ground                     |
| 39-40 | +3.3V     | Power      | 3.3V supply                |

**Note:** All 17 differential pairs on J701/J12 are used for LVDS. DCO is assigned to GC-capable pair IO1_7 (pins 15-16).

### **J901 Connector Pinout (DAC Data + Control Signals, 3.3V):**

J901 connects to ALINX J15 -> FPGA BANK25/26 (HR banks, VCCO = 3.3V LVCMOS).
Used for **DAC TX data, control signals, and ADC overflow**.

| Pin | Signal     | J15 FPGA Pin | Direction      | Description                                         |
| --- | ---------- | ------------ | -------------- | --------------------------------------------------- |
| 1   | GND        | --           | --             | Ground                                              |
| 2   | +5V        | VCC5V        | Power          | 5V supply from ALINX                                |
| 3   | --         | IO2_1N (A11) | --             | Not connected                                       |
| 4   | TX_SEL0    | IO2_1P (A12) | FPGA -> DAC    | MAX5885 mode select (IC901.30)                      |
| 5   | TX_D15     | IO2_2N (A13) | FPGA -> DAC    | DAC data bit 15 (IC901.33)                          |
| 6   | TX_D14     | IO2_2P (B13) | FPGA -> DAC    | DAC data bit 14 (IC901.34)                          |
| 7   | TX_D13     | IO2_3N (A14) | FPGA -> DAC    | DAC data bit 13 (IC901.35)                          |
| 8   | TX_D12     | IO2_3P (B14) | FPGA -> DAC    | DAC data bit 12 (IC901.36)                          |
| 9   | TX_D11     | IO2_4N (E13) | FPGA -> DAC    | DAC data bit 11 (IC901.37)                          |
| 10  | TX_D10     | IO2_4P (E14) | FPGA -> DAC    | DAC data bit 10 (IC901.38)                          |
| 11  | TX_D9      | IO2_5N (A15) | FPGA -> DAC    | DAC data bit 9 (IC901.39)                           |
| 12  | TX_D8      | IO2_5P (B15) | FPGA -> DAC    | DAC data bit 8 (IC901.40)                           |
| 13  | TX_D7      | IO2_6N (C13) | FPGA -> DAC    | DAC data bit 7 (IC901.41)                           |
| 14  | TX_D6      | IO2_6P (C14) | FPGA -> DAC    | DAC data bit 6 (IC901.44)                           |
| 15  | TX_D5      | IO2_7N (B10) | FPGA -> DAC    | DAC data bit 5 (IC901.45)                           |
| 16  | TX_D4      | IO2_7P (C11) | FPGA -> DAC    | DAC data bit 4 (IC901.46)                           |
| 17  | TX_D3      | IO2_8N (D14) | FPGA -> DAC    | DAC data bit 3 (IC901.47)                           |
| 18  | TX_D2      | IO2_8P (D15) | FPGA -> DAC    | DAC data bit 2 (IC901.48)                           |
| 19  | TX_D1      | IO2_9N (F11) | FPGA -> DAC    | DAC data bit 1 (IC901.1)                            |
| 20  | TX_D0      | IO2_9P (F12) | FPGA -> DAC    | DAC data bit 0 (IC901.2)                            |
| 21  | TX_XOR     | IO2_10N (H13)| FPGA -> DAC    | MAX5885 XOR input (IC901.3)                         |
| 22  | --         | IO2_10P (H14)| --             | Not connected                                       |
| 23  | ADC_OF_N   | IO2_11N (G14)| ADC -> FPGA    | LTC2209 Overflow- (IC701.59), 47 ohm series         |
| 24  | ADC_OF_P   | IO2_11P (G15)| ADC -> FPGA    | LTC2209 Overflow+ (IC701.60), 47 ohm series         |
| 25  | DSA_LE2    | IO2_12N (F10)| FPGA -> DSA2   | PE43711 #2 Latch Enable (via R912, 47 ohm)          |
| 26  | DSA_LE1    | IO2_12P (G11)| FPGA -> DSA1   | PE43711 #1 Latch Enable (via R913, 47 ohm)          |
| 27  | DSA_SCLK   | IO2_13N (H12)| FPGA -> DSA    | DSA SPI Clock (via R910, 47 ohm)                    |
| 28  | DSA_DATA   | IO2_13P (J12)| FPGA -> DSA    | DSA SPI Data (via R911, 47 ohm)                     |
| 29  | FLT_V2     | IO2_14N (J14)| FPGA -> Filters| Filter select bit 1 (via R918, 47 ohm)              |
| 30  | FLT_V1     | IO2_14P (K14)| FPGA -> Filters| Filter select bit 0 (via R914, 47 ohm)              |
| 31  | FLT_V4     | IO2_15N (K12)| FPGA -> Filters| Filter select bit 3 (via R917, 47 ohm)              |
| 32  | FLT_V3     | IO2_15P (K13)| FPGA -> Filters| Filter select bit 2 (via R915, 47 ohm)              |
| 33  | I2C_SDA    | IO2_16N (L13)| FPGA <-> Si5391B| I2C Data (4.7k ohm pull-up, no series R)            |
| 34  | RX_TX_MODE | IO2_16P (L14)| FPGA -> System | **Atomic RX/TX** (F2912+LTC2209+DAC, via R916 47 ohm)|
| 35  | --         | IO2_17N (G10)| --             | Not connected                                       |
| 36  | I2C_SCL    | IO2_17P (H11)| FPGA -> Si5391B| I2C Clock (4.7k ohm pull-up, no series R)            |
| 37  | GND        | --           | --             | Ground                                              |
| 38  | GND        | --           | --             | Ground                                              |
| 39  | +3.3V      | VCC_3V3      | Power          | 3.3V supply from ALINX                              |
| 40  | +3.3V      | VCC_3V3      | Power          | 3.3V supply from ALINX                              |

**Note:** All signals through J901/J15 use 3.3V LVCMOS (FPGA BANK25/26, HR banks).
DAC clock (DAC_CLK_P/N) is routed directly from Si5391B (IC801.34/35) to MAX5885 (IC901.6/7) -- NOT through J901.

**WARNING:** Maximum LVCMOS33 output speed on HR banks is ~100-125 MHz (DS925).
The DAC operates at 153.6 MSPS -- this exceeds the specification. See the compatibility analysis section below.

---

## SYSTEM PERFORMANCE TARGETS

### **RF Performance:**

- **Frequency Range:** 9 kHz - 72 MHz
- **Input Impedance:** 50 ohm (VSWR < 1.5:1)
- **Gain Range:** -63.5 dB to +15.2 dB (total)
  - DSA1: 0 to -31.75 dB (Block 1)
  - DSA2: 0 to -31.75 dB (Block 5)
  - LNA: +15.2 dB (Block 6, fixed)
- **Noise Figure:** ~3.0 dB @ 0dB attenuation (dominated by LTC6432)
- **IP3:** >+25 dBm (at ADC input)
- **Dynamic Range:** >100 dB (with dual DSA)

### **Filter Bank:**

- **Bands:** 7 switchable bandpass filters + Master LPF
- **Isolation:** >55 dB between bands
- **Insertion Loss**: <3 dB per band

---

## KEY DESIGN FEATURES

### **1. Dual DSA Architecture**

- **Wide dynamic range control:** 0 to -63.5 dB in 0.25 dB steps
- **Prevents ADC overload** for strong signals
- **Maintains low noise floor** for weak signals

### **2. LTC6432 LNA/Driver Integration**

- **Combines LNA + ADC driver** in a single IC
- **Fixed gain** +15.2 dB (no adjustment needed)
- **Low noise figure 3.0 dB** for sensitivity
- **High IP3 +33 dBm** for linearity
- **Dual balun architecture** for optimal impedance matching
- **Differential outputs** native for LTC2209

### **3. Switched Filter Bank**

- **7 programmable bands** for selectivity
- **Dual SP8T switches** (input + output) for isolation
- **Master LPF** wideband 9 kHz - 72 MHz range for digital analysis and signal processing

---

### 4. Power Distribution

- **Separate LDOs** for each block
- **Ultra-low noise TPS7A9401DSCR** for LTC6432
- **Dual LC filtering** in LNA power rails
- **Ferrite beads + bypass capacitors** on all rails

### **5. Differential Signal Path**

- **Balanced transmission** from Block 1 to ADC
- **Common-mode noise rejection**
- **Optimal ADC driving** with T602 balun

---

## DOCUMENTATION FILES

| File                                       | Description                               | Status           |
| ------------------------------------------ | ----------------------------------------- | ---------------- |
| **PROJECT_OVERVIEW_v2.0.md**               | This file - general project overview      | Complete         |
| **Block1_Input_DSA_Documentation.md**      | Detailed Block 1 documentation            | To be created    |
| **Block2_Filter_Bank_Documentation.md**    | Detailed Block 2 documentation            | To be created    |
| **Block4_Power_Control_Documentation.md**  | Detailed Block 4 documentation            | To be created    |
| **Block5_DSA2_Documentation.md**           | Detailed Block 5 documentation            | To be created    |
| **Block6_LTC6432_Driver_Documentation.md** | Detailed Block 6 documentation            | To be created    |
| **Block7_LTC2209_ADC_Documentation.md**    | Detailed Block 7 documentation            | To be created    |

---

## VERSION HISTORY

| Version | Date           | Changes                                                                          |
| ------- | -------------- | -------------------------------------------------------------------------------- |
| **1.0** | 2026-01-07     | Initial design with ADA4937 driver + PGA103+ LNA                                |
| **2.0** | **2026-01-09** | **Major redesign: removed Block 4, replaced Block 6 with LTC6432**              |
| **2.1** | **2026-01-10** | **Added new Block 4 (RX/TX Power Control with TPS22976)**                       |
| **2.2** | **2026-01-11** | **Block 4: Software inversion in FPGA instead of IC402**                        |
| **2.3** | **2026-01-11** | **Added Block 8 (Clock Generation with Si5391B)**                               |
| **2.4** | **2026-01-13** | **Block 7: Updated ADC documentation (153.6 MSPS, CMOS mode)**                  |
| **2.5** | **2026-01-13** | **Added TX path: Block 9 (MAX5885 DAC) + Block 10 (LTC6432 TX driver)**        |
| **2.6** | **2026-01-14** | **New TX architecture: removed Block 10, shared RX/TX path through Block 6**   |
| **2.7** | **2026-01-18** | **Atomic RX/TX control: single /RX_TX_MODE signal controls switches + ADC/DAC** |
| **2.8** | **2026-01-25** | **LVDS interface for LTC2209, 3.3V control signals, removed IC704**             |
| **2.9** | **2026-01-25** | **Block 6: 3-stage RF choke, optimized de-Qing, new baluns**                    |

### **Version 2.9 Changes:**

1. **Block 6 RF Choke redesigned:** 3-stage instead of 2-stage for continuous coverage
   - L601/L604: 470uH (SWPA8065S471M) - LF coverage
   - L611/L612: 47uH (SWPA4030S470MT) - MF coverage (NEW)
   - L602/L603: 560nH (SWPA252010SR56NT) - HF coverage
2. **De-Qing resistors optimized:** R603=390 ohm, R620=82 ohm, R604=54 ohm
3. **Shunt resistors added:** R622/R623 (2.2k ohm) for C613/C614
4. **Baluns updated:** T601=TTWB-2-A (1:2), T602=CX2041NLT (1:1), BW=50kHz-200MHz
5. **Total DCR:** 1.86 ohm -> VCC=4.87V (within LTC6432 specification)

### **Version 2.8 Changes:**

1. **LTC2209 LVDS interface:** Replaced CMOS single-ended with LVDS differential
   - 16 differential data pairs (DA0-DA15, DB0-DB15)
   - 1 differential clock pair (CLKOUTA/CLKOUTB) on GC-capable IO1_7
   - DDR @ 153.6 MHz = 307.2 Mbps per lane
   - FPGA internal 100 ohm termination (DIFF_TERM_ADV TERM_100)
2. **ADC_OF moved to J901.21/22:** Overflow signal on 3.3V HR banks
3. **F2912 LOGICCTL=GND:** 3.3V logic mode for direct control from FPGA
4. **Removed IC704 (U704):** Level shifter no longer needed
5. **All control signals on J901:** DSA, FLT, RX_TX_MODE, I2C on 3.3V LVCMOS
6. **Updated J701 pinout:** LVDS interface only (17 differential pairs)

### **Version 2.7 Changes:**

1. **Atomic RX/TX control:** merged TX_RX_SWITCH and RX_TX_MODE into a single signal
2. **Single FPGA pin** (/RX_TX_MODE, J901.32) controls all RX/TX switching:
   - 3x F2912 RF switches (IC401-403.CTL2)
   - LTC2209 SHDN (direct 3.3V connection)
   - MAX5885 PD (via U902 inverter)
3. **Control current budget:** ~3.5 uA (margin >2000x from FPGA IOH/IOL)
4. **F2912 1-pin mode:** ModeCTL=VCC, LogicCTL=GND (3.3V logic)
5. **Pin 33 freed** for future use

### **Version 2.6 Changes:**

1. **Removed Block 10** (separate TX Driver no longer needed)
2. **Updated Block 4:** RF Switching instead of Power Control
   - IC401, IC402, IC403: F2912NCGI8 (SP3T RF switches)
   - RF switches control (updated to atomic in v2.7)
   - <100 ns switching time, >74 dB isolation
3. **Updated Block 9:** Correct DAC output stage topology
   - R903/R904 (25 ohm): DC return path for current-output DAC
   - Parallel decoupling: C915/C916 (1uF || 100nF), C917/C918 (1uF || 100nF)
   - T902: RF transformer with center tap bypass (C919)
4. **New TX architecture:** Shared RX/TX signal path through Block 2-3-5-6
   - TX signal: DAC -> IC401 -> Filter Bank -> LTC6432 -> IC402/IC403 -> TX Antenna
   - RX signal: Antenna -> Filter Bank -> LTC6432 -> IC402/IC403 -> ADC
5. **Power Management:** All blocks always on, ADC/DAC controlled via SHDN/PD pins
6. **Updated Block Structure table:** 9 blocks instead of 10

### **Version 2.5 Changes:**

1. **Added Block 9** (DAC & FPGA Interface for TX)
2. **IC901** MAX5885EGM+D: 16-bit DAC (153.6 MSPS update rate)
3. **Reference circuit:** R901 = 2k ohm -> Full-scale current 19.2 mA
4. **U901** LP5912-3.3DRV: Ultra-low noise LDO for DAC
5. **J901** 2x20 header: TX data/control interface to FPGA

### **Version 2.4 Changes:**

1. **Updated Block 7 documentation** with complete LTC2209 information
2. **Sample rate: 153.6 MSPS** (was 122.88 MHz -> now 153.6 MHz)
3. **Added Control Pins Configuration table** with rationale for each pin
4. **CMOS vs LVDS explanation:** Pin count limitation (J701 has only 40 pins)
5. **Dual Supply Architecture:** VDD=3.3V (analog), OVDD=1.8V (digital outputs)
6. **Mode rationale:** DITH=GND (better SNR), PGA=GND (gain=1)
7. **Updated all references from 122.88 MHz to 153.6 MHz** in the document

### **Version 2.3 Changes:**

1. **Added Block 8** (Clock Generation)
2. **IC801** Si5391B-A-GM: Ultra-low jitter clock generator (<50 fs RMS)
3. **X801** 38.4 MHz TCXO: Integer Mode reference (no fractional spurs)
4. **Dual LDO architecture:** U801 (1.8V) + U802 (3.3V) with LP5912 (12uVRMS noise)
5. **I2C programmability:** Address 0x68, ClockBuilder Pro configuration
6. **Output:** 122.88 MHz differential -> LTC2209 ADC clock input
7. **Power filtering:** Ferrite beads + decoupling capacitors for minimal jitter

### **Version 2.2 Changes:**

1. **Removed IC402** (74LVC1G04 inverter) from Block 4
2. **Software inversion in FPGA:** 2 pins (RX_TX_MODE, RX_TX_MODE_NOT)
3. **Improved reliability:** VOL = 0V, VOH = 1.8V (ideal margins)
4. **Updated J701 pinout:** Pin 32 (RX_TX_MODE), Pin 33 (RX_TX_MODE_NOT)

### **Version 2.1 Changes:**

1. **Added Block 4** (RX/TX Power Control and Indication Circuit)
2. **IC401** TPS22976DPUR: Dual-channel load switch for power management
3. **+5V_RX and +5V_TX rails:** separate power for RX and TX paths
4. **LED indication:** Green (RX mode), Blue (TX mode)

### **Version 2.0 Changes:**

1. **Removed Block 4** (PGA103+ LNA + bypass switches)
2. **Replaced IC601** from ADA4937-1YCPZ-R7 to **LTC6432AIUF-15#PBF**
3. **Added input balun T601** (ADT2-1T+ 1:2)
4. **Added output balun T602** (ADT2-1T+ 1:2) for VCM biasing
5. **Simplified architecture:** single chip instead of LNA + driver
6. **Fixed gain** +15.2 dB (no VGA needed)
7. **Improved power supply:** TPS7A2050 ultra-low noise LDO

---

## NEXT STEPS

### **Design Phase:**

- [ ] Create detailed documentation for each block
- [ ] Optimize output matching network Block 6
- [ ] Thermal analysis for LTC6432 (165mA @ 5V)

### **Verification:**

- [ ] SPICE simulation Block 6 (gain, IP3, noise)
- [ ] Power budget analysis (total current consumption)
- [ ] Signal integrity analysis (differential pairs)
- [ ] EMI/EMC considerations

### **Implementation:**

- [ ] PCB layout design (4-layer board)
- [ ] Component procurement
- [ ] Assembly + bring-up testing
- [ ] Performance characterization

---

## CONTACT & SUPPORT

**Datasheet Links:**

- LTC6432: [Analog Devices LTC6432 Product Page](https://www.analog.com/en/products/ltc6432.html)
- LTC2209: [Analog Devices LTC2209 Product Page](https://www.analog.com/en/products/ltc2209.html)
- PE43711A: [pSemi PE43711A Product Page](https://www.psemi.com/)
- PE42582A: [pSemi PE42582A Product Page](https://www.psemi.com/)

---

**End of Document**
