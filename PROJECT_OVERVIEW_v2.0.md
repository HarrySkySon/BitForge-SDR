# SDR Receiver/Transceiver Project - Technical Overview

**Version:** 2.9
**Date:** 2026-01-25
**Status:** RX/TX Complete (LVDS interface, 3.3V control, optimized Block 6)

---

## 📋 PROJECT SUMMARY

**Назва:** Software Defined Radio (SDR) Receiver з LTC2209 ADC
**ADC:** LTC2209 (16-bit, 160 MSPS)
**DAC:** MAX5885 (16-bit, 200 MSPS)
**Діапазон частот:** 9 kHz - 72 MHz
**Динамічний діапазон:** >100 dB (з dual DSA)
**Gain:** 15.2 dB (fixed, LTC6432 LNA/Driver)

---

## 🏗️ BLOCK STRUCTURE

Проєкт складається з **9 функціональних блоків** з наскрізною нумерацією компонентів:

| Block          | Prefix | Функція              | Key ICs            | Призначення                                                |
| -------------- | ------ | -------------------- | ------------------ | ---------------------------------------------------------- |
| **Block 1**    | 100s   | Input Stage          | PE43711A-Z, TC1-1T | Вхідний захист, балансування, DSA1                         |
| **Block 2, 3** | 200s   | Switched Filter Bank | 2× PE42582A-X      | Багатосмугова фільтрація (7 bands) + Master LPF 0 - 72 MHz |
| **Block 4**    | 400s   | RX/TX RF Switching   | 3× F2912NCGI8      | RF комутація між RX та TX режимами                         |
| **Block 5**    | 500s   | Second DSA           | PE43711A-Z         | Додаткове цифрове затухання                                |
| **Block 6**    | 600s   | LNA/ADC Driver       | LTC6432AIUF-15     | LNA + differential driver (+15.2dB) для RX/TX              |
| **Block 7**    | 700s   | ADC + Interface      | LTC2209, TPS7A9101 | 16-bit ADC + FPGA interface (RX)                           |
| **Block 8**    | 800s   | Clock Generation     | Si5391B-A-GM       | Ultra-low jitter clock (38.4 MHz TCXO → 153.6 MHz)         |
| **Block 9**    | 900s   | DAC & TX Interface   | MAX5885, LP5912    | 16-bit DAC + FPGA interface (TX)                           |

---

## 📦 DETAILED BLOCK DESCRIPTIONS

### **BLOCK 1: Input Stage & DSA1** (100s prefix)

#### **Основні компоненти:**

- **IC101** - LP5907MFX-3.3: Ultra-low noise LDO (3.3V, 250mA)
- **IC104** - PE43711A-Z: 6-bit Digital Step Attenuator (0-31.75 dB, 0.25dB steps)
- **T101** - TC1-1T: 1:1 RF Transformer (50Ω impedance matching)
- **TVS101** - CDSOD323-T05LC: Transient Voltage Suppressor
- **J101** - SMA RX: Вхідний коаксіальний конектор

#### **Функції:**

1. **ESD/Overvoltage Protection** (TVS101)
2. **Single-ended to Differential Conversion** (T101)
3. **Digital Control Gain** (IC104): 0 to -31.75 dB, FPGA controlled
4. **Clean Power Supply** (IC101) для DSA

#### **Control:**

- **DSA_SCLK, DSA_DATA, DSA_LE1**: SPI control для DSA1 (7-bit attenuation values)

#### **Характеристики:**

- **Input impedance:** 50Ω
- **IP3:** +42 dBm (PE43711A)
- **Insertion Loss:** ~3 dB @ 0 dB attenuation

---

### **BLOCK 2: Switched Filter Bank** (200s prefix)

#### **Основні компоненти:**

- **IC201** - PE42582A-X: SP8T RF Switch (вхід)
- **IC202** - LP5907MFX-3.3: LDO для живлення switches
- **IC203** - PE42582A-X: SP8T RF Switch (вихід)
- **L301-L356, C301-C374**: 7× LC Band-Pass Filters
- **Master LPF:** - Широкосмуговий фільтр всього динамичного діапазону 9 kHz - 72 MHz. Призначений для забезпечиння повного огляду полоси пропускання із наступною цифровою обробкою і фільтацією в FPGA

#### **Архітектура:**

```
IC201 (SP8T) → [7× Bandpass Filters та Master LPF 9 kHz - 72 MHz] → IC203 (SP8T)
     ↓                                        ↓
  FLT_V1-V4                               FLT_V1-V4
  (4-bit binary)                          (4-bit binary)
```

#### **Фільтри (7 bands):**

| Band | Filter Path       | Frequency Range | Components           |
| ---- | ----------------- | --------------- | -------------------- |
| 1    | RF_FILTR_IN/OUT_1 | 0.09 - 1.6 MHz  | L301-L305, C301-C305 |
| 2    | RF_FILTR_IN/OUT_2 | 1.5 - 4 Mhz     | L306-L312, C306-C315 |
| 3    | RF_FILTR_IN/OUT_3 | 4 - 8 Mhz       | L313-L319, C316-C327 |
| 4    | RF_FILTR_IN/OUT_4 | 8 - 15.5 Mhz    | L320-L327, C328-C341 |
| 5    | RF_FILTR_IN/OUT_5 | 15 - 30 Mhz     | L328-L333, C342-C351 |
| 6    | RF_FILTR_IN/OUT_6 | 30 - 50 MHz     | L334-L337, C352-C358 |
| 7    | RF_FILTR_IN/OUT_7 | 47 - 73 MHz     | L338-L347, C359-C374 |
| 8    | MASTER_LPF_IN/OUT | 0.09 - 72MHz    | L338-L347, C359-C374 |

#### **Master LPF:**

- **MASTER_LPF_IN**→ L348-L356 → **MASTER_LPF_OUT**
- Широкосмуговий фільтр всього динамичного діапазону 9 kHz - 72 MHz.

#### **Control:**

- **FLT_V1, FLT_V2, FLT_V3, FLT_V4**: 4-bit binary control для обох switches (синхронне перемикання)
  Вибір діапазону відбувається програмно. Користувач спочатку бачить повністю всю смугу 9 kHz - 72 MHz. Це йому дозволяє бачити радіодіапазон і обирати програмні засоби і фільтри для роботи із сигналом. Якщо корсистувач наближає діапазон, відбувається переключення і вибір відповідного BPF фільтра діапазону, що забезпечує тонку фільтрацію сигналу в даному діапазоні.

#### **Функції:**

####

#### **Характеристики:**

- **Isolation:** >55 dB (PE42582A)
- **Insertion Loss:** ~0.5 dB per switch
- **Switching Speed:** <500 ns

---

### **BLOCK 4: RX/TX RF Switching** (400s prefix)

#### **Основні компоненти:**

- **IC401, IC402, IC403** - F2912NCGI8: SP2T RF switches (Renesas)
- **IC404, IC405** - LP5907MFX-3.3: LDO для живлення switches

#### **Функції:**

- **IC401:** Інжекція TX сигналу від DAC в спільний RX/TX тракт (Filter Bank)
- **IC402/IC403:** Маршрутизація differential outputs LTC6432 між ADC (RX) та TX antenna
- **1-pin control:** Всі switches + ADC/DAC керуються одним сигналом `/RX_TX_MODE`

#### **Архітектура:**

```
        RX Mode (/RX_TX_MODE = LOW):
Antenna → Filter Bank → LTC6432 → IC402/IC403 (RF1) → ADC (active)
                      (DAC shutdown)

        TX Mode (/RX_TX_MODE = HIGH):
DAC (active) → IC401 (RF2) → Filter Bank → LTC6432 → IC402/IC403 (RF2) → TX Antenna
                                                    (ADC shutdown)
```

#### **Атомарне управління RX/TX (один сигнал):**

**Switch Control Truth Table:**

| Mode   | /RX_TX_MODE | F2912 (IC401-403) | LTC2209 SHDN | MAX5885 PD   |
| ------ | ----------- | ----------------- | ------------ | ------------ |
| **RX** | LOW (0V)    | RF1 ON (RX path)  | LOW (active) | HIGH (off)   |
| **TX** | HIGH (3.3V) | RF2 ON (TX path)  | HIGH (off)   | LOW (active) |

**Control Signal Path:**

```
FPGA (J15, BANK25/26, 3.3V) → J901.32 → RX_TX_MODE
                                          ├── IC401.16 (CTL2) ─→ F2912 #1
                                          ├── IC402.16 (CTL2) ─→ F2912 #2
                                          ├── IC403.16 (CTL2) ─→ F2912 #3
                                          └── IC701.19 (SHDN) ─→ LTC2209
```

**Control Current Budget (FPGA 3.3V LVCMOS output):**

| Component       | Pins          | Current per pin | Total       |
| --------------- | ------------- | --------------- | ----------- |
| F2912 CTL2 (×3) | IC401-403.16  | 500 nA max      | **1.5 µA**  |
| LTC2209 SHDN    | IC701.19      | ~1 µA           | **1 µA**    |
| U902 input      | Level shifter | ~1 µA           | **1 µA**    |
| **Total**       |               |                 | **~3.5 µA** |

**FPGA IOH/IOL capability: 8-12 mA → margin >2000×**

#### **F2912 Configuration (3.3V Control Mode):**

- **Pin 18 (LOGICCTL)** = **GND** → **3.3V logic mode enabled**
- **Pin 19 (ModeCTL)** = VCC → 1-pin control mode
- **Pin 16 (CTL2)** = RX_TX_MODE → Control input (3.3V logic)
- **Logic thresholds (3.3V):** VIH min = 2.0V, VIL max = 0.8V

**Перевага LOGICCTL=GND:**

- Прямий контроль через 3.3V сигнал з FPGA J15 (BANK25/26)
- Не потребує level shifter IC704 (видалено)
- Спрощена схема, менше компонентів

#### **Характеристики:**

- **Switching time:** <100 ns (RF switches)
- **Isolation:** >74 dB (RF1/RF2 @ 1 GHz)
- **Insertion Loss:** <0.5 dB @ HF frequencies
- **Control:** Один FPGA pin `/RX_TX_MODE` (LVCMOS33, J901.32)

---

### **BLOCK 5: Second DSA** (500s prefix)

#### **Основні компоненти:**

- **IC501** - LP5907MFX-3.3: LDO для DSA2
- **IC502** - PE43711A-Z: 6-bit Digital Step Attenuator

#### **Функції:**

- **Цифрове затухання** Додаткове цифрове затухання (0 to -31.75 dB). Крок 0.25 dB
- **Dual DSA cascade** з Block 1 для wide dynamic range (0 to -63.5 dB total)

#### **Control:**

- **DSA_SCLK, DSA_DATA, DSA_LE2**: SPI control для DSA2 (7-bit attenuation values)

#### **Характеристики:**

- Ідентичні до IC104 (Block 1)
- **Total Gai- **Range:\*\* 0 to -63 dB (з Block 1)

---

### **BLOCK 6: LNA/ADC Driver** (600s prefix) ⭐ **NEW DESIGN**

#### **Основні компоненти:**

- **IC601** - **LTC6432AIUF-15#PBF**: Differential LNA/Driver (+15.2 dB gain)
- **IC602** - TPS7A9401DSCR: Ultra-low noise LDO
- **T601** - TTWB-2-A: 1:2 Input Balun (single-ended to differential), Bandwidth = 50 kHz to 200 MHz
- **T602** - CX2041NLT: 1:1 Output Balun (re-balancing + common-mode biasing), Bandwidth = 50 kHz to 200 MHz
- **L601, L611, L602, L603, l612, L604**: Power supply chokes + output filter inductors
- **C601-C622**: Coupling, bypass, and filter capacitors
- **R601-R608**: Input termination, biasing, output termination

#### **Архітектура:**

```
RF_1 → T601 (1:2 balun) → IC601 (LTC6432) → T602 (2:1 balun) → ADC
       50Ω single-ended     Diff. Amp         Output balancing   Differential
                           Gai- **= +15.2dB                        to LTC2209
                                  ↓
                            IC602 (5V LDO)
```

#### **Pinout LTC6432 (IC601):**

| Pin                      | Function | Connection            |
| ------------------------ | -------- | --------------------- |
| 5                        | NFILT2   | C605 (1µF) filter cap |
| 6                        | NFILT1   | C606 (1µF) filter cap |
| 7                        | -In      | Від T601 через C602   |
| 8, 15, 17, 23, 25        | GND      | Ground plane          |
| 9, 22                    | +VS      | +5V (від IC602)       |
| 12                       | -FDBK    | C608 (1µF) feedback   |
| 13                       | -OUT     | До T602 через C614    |
| 18                       | +OUT     | До T602 через C613    |
| 19                       | +FDBK    | C607 (1µF) feedback   |
| 24                       | +In      | Від T601 через C601   |
| 1-4, 10-11, 14-16, 20-21 | DNC      | Do Not Connect        |

#### **Input Network:**

```
RF_1 (from Block 5) → T601 (TTWB-2-A, 1:2) → ±In (IC601)
                      50Ω SE → 200Ω diff      Pins 24, 7
```

#### **Output Network:**

```
IC601 ±OUT → C613/C614 (1µF) ──┬── R622/R623 (2.2kΩ shunt)
                               └── T602 (CX2041NLT, 1:1) → ADC differential input
                                   Center tap → LTC2209 VCM
```

#### **Power Supply: 3-Stage RF Choke Network**

Трьохступенева мережа RF choke для безперервного покриття діапазону 9 kHz - 72 MHz:

```
+5V_B6 ─┬─ L601 (470µH) ─┬─ L611 (47µH) ─┬─ L602 (560nH) ─┬─ IC601 +VS
        │                │               │                │
        └─ R603 (390Ω) ──┘─ R620 (82Ω) ──┘─ R604 (54Ω) ───┘
```

| Stage | Inductor     | De-Qing R   | Effective Range |
| ----- | ------------ | ----------- | --------------- |
| LF    | L601 (470µH) | R603 (390Ω) | 9 kHz - 1 MHz   |
| MF    | L611 (47µH)  | R620 (82Ω)  | 500 kHz - 6 MHz |
| HF    | L602 (560nH) | R604 (54Ω)  | 5 MHz - 72 MHz  |

**Характеристики:**

- Total DCR: 1.86Ω → DC drop: 130 mV → VCC = 4.87V (в межах специфікації)
- Z_choke @ 72 MHz: 516Ω
- Worst resonance: Q=72 @ 213 kHz (добре демпфовано)

#### **Характеристики LTC6432:**

- **Gain:** +15.2 dB (fixed)
- **Noise Figure:** 3.0 dB @ 100 MHz
- **Output IP3:** +33 dBm
- **Bandwidth:** 100 kHz to 1400 MHz (-3dB)
- **Output Impedance:** 50Ω differential
- **Supply Current:** 165 mA @ 5V
- **DC Power:** 850mW

#### **Функції блоку:**

1. **Low Noise Amplification:** +15.2 dB з NF = 3.0 dB
2. **Single-ended to Differential Conversion** (T601)
3. **Differential Output Balancing** (T602)
4. **ADC Common-Mode Biasing** (T602 center tap → VCM)
5. **Output Filtering** (L605/L606 + C617/C618/C619)
6. **Clean Power Supply** (IC602 ultra-low noise LDO)

#### **Особливості:**

- **Dual Balun Architecture:** T601 (TTWB-2-A, 1:2) + T602 (CX2041NLT, 1:1), BW = 50 kHz - 200 MHz
- **3-Stage RF Choke:** Безперервне покриття 9 kHz - 72 MHz без резонансних провалів
- **Optimized De-Qing:** R603=390Ω, R620=82Ω, R604=54Ω для контролю Q-фактора
- **Shunt Resistors:** R622/R623 (2.2kΩ) для додаткового демпфування LC резонансів
- **T602 VCM Biasing:** Center tap з'єднано з LTC2209 VCM для DC зміщення ADC

---

### **BLOCK 7: ADC & FPGA Interface** (700s prefix)

#### **Основні компоненти:**

- **IC701** - LTC2209IUPPBF: 16-bit ADC (153.6 MSPS)
- **IC702** - TPS7A9101DSKT: Precision LDO для analog supply (3.3V)
- **IC703** - LP5907MFX-1.8: LDO для digital outputs (1.8V)
- **J701** - Pin Header 2×20: FPGA interface connector (LVDS data)
- **J901** - Pin Header 2×20: Control signals + ADC_OF (3.3V)
- **C701** - 10µF: VCM bypass capacitor
- **C702, C711, C712** - Decoupling capacitors для VDD (3.3V)
- **C703, C718** - Decoupling capacitors для OVDD (1.8V)

#### **ADC Specifications:**

- **Resolution:** 16 bits
- **Sample Rate:** 153.6 MSPS (від Block 8)
- **Nyquist Bandwidth:** 76.8 MHz (covers 9 kHz - 72 MHz range)
- **Noise Floor:** 77.3 dBFS
- **SFDR:** 100 dB (typical)
- **Output Format:** 2's Complement (16-bit parallel)
- **Output Mode:** **LVDS Differential** (17 pairs via J701)
- **Power Dissipation:** ~1.0W @ 153.6 MSPS

#### **Output Mode: LVDS Differential**

**Чому LVDS:**

- **Висока швидкість:** DDR на 153.6 MHz = 307.2 Mbps per lane
- **Завадостійкість:** Диференційні сигнали з common-mode rejection
- **Сумісність:** FPGA HP banks (BANK65/66) мають native LVDS підтримку
- **Термінація:** Вбудовані 100Ω резистори в FPGA (DIFF_TERM_ADV TERM_100)

**LTC2209 LVDS Output Structure:**

- **DA0-DA15:** 8 differential pairs (Bus A)
- **DB0-DB15:** 8 differential pairs (Bus B)
- **CLKOUTA/CLKOUTB:** 1 differential pair (Data Clock Output)
- **OFA/OF-:** Overflow → J901 pins 21/22 (single-ended 3.3V)
- **Total LVDS pairs on J701:** 17 (all J12 pairs utilized)

**LVDS Electrical Characteristics:**

| Parameter              | Value                      |
| ---------------------- | -------------------------- |
| Common Mode (VOS)      | 1.125V - 1.2V - 1.375V     |
| Differential (VOD)     | 247mV - 350mV - 454mV      |
| FPGA Input Common Mode | 0.3V - 1.5V (compatible)   |
| FPGA Min Differential  | >100mV (350mV >> 100mV OK) |
| Termination            | 100Ω internal in FPGA      |

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
  - Decoupling: C702 (1µF), C711 (47µF), C712 (10nF)

- **OVDD (1.8V):** Digital output drivers (pins 32,49)
  - Source: IC703 (LP5907-1.8) ultra-low noise LDO
  - Decoupling: C703 (1µF), C718 (2.2µF)
  - Powers LVDS output drivers

- **VCM (Pin 3):** Output voltage reference (0.95V typical)
  - Bypass: C701 (10µF, ≥2.2µF per datasheet)
  - Connected to T602 center tap for DC bias

#### **Analog Inputs:**

- **IN+ (Pin 8), IN- (Pin 9):** Differential analog inputs
  - Connected to ADC_P_OUT, ADC_N_OUT (від Block 6 через T602)
  - Input range: 2.25Vp-p differential (PGA=0)
  - Series termination: R607/R608 (25Ω each)

#### **Clock Input:**

- **ENC+ (Pin 12), ENC- (Pin 13):** Differential clock inputs
  - Connected to CLK_0, CLK_B (від Block 8)
  - Clock frequency: 153.6 MHz (від Si5391B OUT0/OUT0b)
  - AC coupled через C809, C818 (100nF)
  - Terminated: R701 (100Ω differential)

#### **Digital Outputs (LVDS to FPGA via J701):**

- **DA0-DA15, DB0-DB15:** 16 differential pairs (32 lines)
  - Output standard: LVDS (VOS=1.2V, VOD=350mV)
  - Data format: 2's Complement (MODE pin = VDD)
  - DDR transfer: both edges of CLKOUT
  - Connected to J701 pins 3-36 → FPGA J12 (BANK65/66)

- **CLKOUTA/CLKOUTB (Pins 41, 40):** Data clock output
  - LVDS differential pair
  - Connected to J701 pins 15/16 → FPGA IO1_7 (GC-capable)
  - Frequency: 153.6 MHz (same as ENC clock)

- **OFA/OF- (Pins 60, 59):** Overflow indicator
  - Connected to J901 pins 21/22 → FPGA BANK25/26 (3.3V)
  - Used as single-ended signals on HR banks

---

### **BLOCK 8: Clock Generation** (800s prefix)

#### **Основні компоненти:**

- **IC801** - Si5391B-A-GM: Ultra-low jitter clock generator (QFN-65, 12 outputs)
- **X801** - 38.4 MHz TCXO: Temperature-compensated crystal oscillator
- **C801-C817** - Decoupling capacitors: Power supply filtering
- **R801, R802** - 2.2kΩ: I2C pull-up resistors

#### **Архітектура:**

```
X801 (38.4 MHz TCXO) → IC801 (Si5391B PLL) → OUT0/OUT0b (153.6 MHz) → LTC2209 CLK±
                         ↑
                    I2C_SCL, I2C_SDA
                    (FPGA configuration)
```

#### **Clock Generation:**

```
38.4 MHz TCXO (X801)
    ↓ XA/XB (pins 8/9)
Si5391B DSPLL (IC801)
    ├─ VCO: ~5 GHz (internal)
    ├─ Integer Mode (чистий спектр, без fractional spurs)
    └─ MultiSynth dividers
         ↓
OUT0/OUT0b (pins 24/23) → 153.6 MHz differential → LTC2209 CLK± (IC701.12/13)
```

#### **I2C Interface:**

- **I2C_SEL (pin 39)** = +1.8V → I2C mode
- **SCLK (pin 16)** → FPGA I2C_SCL (з 2.2kΩ pull-up)
- **SDA (pin 18)** → FPGA I2C_SDA (з 2.2kΩ pull-up)
- **A1/A0 (pins 17/19)** = GND → I2C address **0x68**

#### **Input Selection:**

- **IN_SEL0/IN_SEL1 (pins 3/4)** = GND → явний вибір XA/XB (crystal input)
- **XA/XB (pins 8/9)** → підключено до 38.4 MHz TCXO (X801)

#### **Характеристики Si5391B для потреб проєкту:**

- **Jitter:** <50 fs RMS (Integer Mode, Precision Calibration)
- **Output frequency:** 153.6 MHz (4/1 ratio - Integer Mode) (programmable via I2C)
- **Sample rate:** 153.6 MSPS
- **Nyquist bandwidth:** 76.8 MHz
- **Coverage:** 9 kHz - 72 MHz with margin
- **Reference:** 38.4 MHz TCXO
- **Mode:** Integer (38.4 MHz × 4/1 = 153.6 MHz) → без fractional spurs
- **Корисна смуга:** 0-68 МГц (з запасом до 72 МГц)
- **Перехідна смуга фільтру:** 68-76.8 МГц (~12% запас)
- **Потік даних:** 153.6 MSPS × 16 bit = 2.46 Гбіт/с
- **Output format:** Differential LVDS/LVPECL (configurable)
- **Power:** 1.8V (VDD core) + 3.3V (VDDA analog)

#### **Clock Performance:**

**Jitter budget (для LTC2209 16-bit ADC):**

```
ADC SNR = -20 × log10(2π × f_sample × t_jitter)

@ 153.6 MSPS, 153.6 MHz clock:
  Si5391B: <50 fs RMS
  → Jitter-limited SNR: ~94 dBFS
  → ENOB: ~15.3 bits (теоретично)
  → LTC2209 spec: 77.3 dBFS noise floor (16-bit ENOB можливо!)
```

#### **Integer Mode перевага:**

**Fractional-N vs Integer Mode:**

- **Fractional:** 25 MHz × 6.144 = 153.6 MHz → створює спектральні спури (birdies на водоспаді SDR)
- **Integer:** 38.4 MHz × 4 = 153.6 MHz → чистий спектр, ідеально для SDR

**Integer Mode calculation:**

```
38.4 MHz × 4 = 153.6 MHz
→ Simple integer multiplier (4×)
→ No fractional dividers needed
→ Zero fractional spurs
```

#### **Функції блоку:**

1. **Ultra-low jitter clock generation** (<50 fs RMS)
2. **Integer Mode operation** (чистий спектр без fractional spurs)
3. **Dual LDO power supply** (ізоляція аналогового/цифрового живлення)
4. **I2C programmability** (ClockBuilder Pro configuration)
5. **Input filtering** (ferrite beads + decoupling caps)
6. **Temperature-stable reference** (38.4 MHz TCXO)

#### **Control:**

- **I2C_SCL, I2C_SDA:** Програмування Si5391B (address 0x68)
- **Configuration:** Via ClockBuilder Pro software (Silicon Labs)
- **Startup:** Завантаження конфігурації через I2C після power-on
  I2C_SCL / I2C_SDA ПОТРІБНІ в поточному дизайні, оскільки:
  1. Використовується стандартний Si5391B-A-GM - без custom NVM
  2. При кожному power-up FPGA повинна завантажити конфігурацію:
  - Input: 38.4 MHz TCXO на XA/XB
  - Output: 153.6 MHz LVDS на OUT0/OUT0b
  - Mode: Integer (без fractional spurs)
  - PLL settings: 38.4 × 4 = 153.6 MHz
  3. Без I2C програмування Si5391B буде працювати з factory default налаштуваннями
  Альтернатива (для майбутніх версій)
  Замовити factory pre-programmed Si5391B через ClockBuilder Pro:
  - Silicon Labs присвоїть унікальний part number (Si5391B-Axxxxx-GM)
  - Мікросхема автоматично стартує з правильною конфігурацією
  - I2C можна буде видалити або залишити тільки для моніторингу

#### **FPGA Interface:**

**Підключення через I2C:**

```
FPGA (J12) → J701 → Si5391B
  Pin 34 (IO1_16P, C8) → I2C_SCL → IC801.16 (з 2.2kΩ pull-up)
  Pin 35 (IO1_17N, A8) → I2C_SDA → IC801.18 (з 2.2kΩ pull-up)
```

**Конфігурація:**

- **I2C Address:** 0x68 (7-bit, A1=GND, A0=GND)
- **Clock speed:** 400 kHz (Fast Mode)
- **Config size:** ~300 bytes (ClockBuilder Pro generated)
- **Init time:** ~25-40 ms (один раз при boot)

**Послідовність ініціалізації:**

1. Power-on → POR (~10 ms)
2. FPGA I2C write (~5-10 ms) → config registers
3. PLL lock (~10-20 ms)
4. ✅ Clock ready (153.6 MHz)

#### **Power Architecture (КРИТИЧНА ОСОБЛИВІСТЬ!):**

**Постійне живлення (незалежно від RX/TX режиму):**

```
+3.3V (постійно) → Si5391B
```

**Чому постійно увімкнений:**

- ✅ TX тракт (MAX5885 DAC) також потребує clock
- ✅ RX і TX використовують один clock source
- ✅ Конфігурація один раз при boot (не треба re-init після перемикання)
- ✅ PLL завжди locked → стабільний clock

**Споживання:**

- **Si5391B:** ~165 mA @ 1.8V/3.3V → **~357 mW** (постійно)
- **Порівняно:** LTC6432 = 850 mW, TX PA = 5-10W
- **Висновок:** Незначне споживання, але критична перевага у швидкості перемикання!

---

### **BLOCK 9: DAC & TX Interface** (900s prefix)

#### **Основні компоненти:**

- **IC901** - MAX5885EGM+D: 16-bit DAC (153.6 MSPS, TX path)
- **U901** - TPS7A9401DSCR: Ultra-low noise LDO (3.3V, 0,46 µVRMS)
- **U902** - SN74LV1T04DCKR: Inverter для PD control
- **T902** - RF Transformer (center-tapped balun)
- **R903, R904** - 25Ω: DC return path resistors
- **R901** - 2kΩ: Full-scale current set resistor
- **C915/C916, C917/C918** - Parallel decoupling (1µF || 100nF)

#### **DAC Specifications:**

- **Resolution:** 16 bits, 200 MSPS
- **Update Rate:** 153.6 MSPS (від Block 8, OUT3/OUT3b)
- **Nyquist Bandwidth:** 76.8 MHz (covers 9 kHz - 72 MHz)
- **Output Mode:** Differential current outputs (IOUTP, IOUTN)
- **Full-Scale Current:** 19.2 mA (з R901 = 2kΩ)
- **Output Format:** 2's Complement (16-bit parallel)

#### **Output Stage:**

```
IC901 Pin 19 (IOUTP) ──┬── R904 (25Ω) → GND  (DC path)
                       ├── C917 (1µF)  ┐
                       └── C918 (100nF)├──→ T902 Pin 3

IC901 Pin 18 (IOUTN) ──┬── R903 (25Ω) → GND  (DC path)
                       ├── C915 (1µF)  ┐
                       └── C916 (100nF)├──→ T902 Pin 1

T902 Center Tap ──→ C919 (100nF) ──→ GND
T902 Output ──→ RF Switches (спільний RX/TX тракт)
```

**Parallel Decoupling:** 1µF || 100nF забезпечує низький Z у всьому діапазоні 9 kHz - 72 MHz

#### **TX Architecture:**

TX сигнал використовує **спільний тракт** з RX через:

- Block 2-3: Filter Bank (PE42582A switches)
- Block 5: DSA (bypass в TX mode)
- Block 6: LTC6432 amplifier
- RF switches керують напрямком сигналу (RX/TX)

---

## 🔗 SIGNAL FLOW

```
┌─────────────┐
│   SMA RX    │ RF Input (1-30 MHz)
│   (J101)    │
└──────┬──────┘
       │ 50Ω single-ended
┌──────▼──────────────────────────────────────────────────────┐
│ BLOCK 1: Input Stage                                        │
│  TVS101 → T101 (1:1) → IC104 (DSA1: 0 to -31.75dB)          │
│                              ↑ DSA_SCLK, DSA_DATA, DSA_LE1  │
│                              ↑ +5V (постійно)               │
└──────┬──────────────────────────────────────────────────────┘
       │ Differential
┌──────▼──────────────────────────────────────────────────────┐
│ BLOCK 2: Switched Filter Bank                               │
│  IC201 (SP8T) → [7× BPF] → IC203 (SP8T) → Master LPF        │
│       ↑ FLT_V1-V4            ↑ FLT_V1-V4                    │
│       ↑ +5V (постійно)                                      │
└──────┬──────────────────────────────────────────────────────┘
       │ Filtered differential
┌──────▼──────────────────────────────────────────────────────┐
│ BLOCK 5: Second DSA                                         │
│  IC502 (DSA2: 0 to -31.75dB)                                │
│         ↑ DSA_SCLK, DSA_DATA, DSA_LE2                       │
│         ↑ +5V (постійно)                                    │
└──────┬──────────────────────────────────────────────────────┘
       │ Attenuated differential
       │ RF_1 (single-ended equivalent)
┌──────▼──────────────────────────────────────────────────────┐
│ BLOCK 6: LNA/ADC Driver ⭐                                 │
│  T601 (1:2) → IC601 (LTC6432, +15.2dB) → T602 (1:1)         │
│               │                              │              │
│               IC602 (5V LDO)                VCM → ADC       │
│               ↑ +5V (постійно)                              │
└──────┬──────────────────────────────────────┬───────────────┘
       │ ADC_P_OUT, ADC_N_OUT (differential)  │ ADC_VCM
       │ 2Vp-p @ 200Ω impedance               │
┌──────▼──────────────────────────────────────▼───────────────┐
│ BLOCK 7: ADC & Interface                                    │
│  IC701 (LTC2209: 16-bit, 153.6 MSPS)                        │
│    ├─ ADC_CLK_IN ← Block 8 (153.6 MHz)                      │
│    └─ ADC_D[15:0] + DCO → FPGA (J701)                       │
│  IC702, IC703: Power supplies                               │
│    ↑ +5V (постійно)                                         │
└──────┬───────────────────┬──────────────────────────────────┘
       |                   │ ADC_CLK_IN
       |         ┌─────────▼────────────────────────────────┐
       |         │ BLOCK 8: Clock Generation                │
       |         │  X801 (38.4 MHz TCXO)                    │
       |         │    ↓                                     │
       |         │  IC801 (Si5391B: <50fs jitter)           │
       |         │    ↓ Integer Mode (no spurs)             │
       |         │  OUT0/OUT0b → 153.6 MHz diff.            │
       |         │  U801/U802: Dual LDO (1.8V + 3.3V)       │
       |         │    ↑ +5V (постійно)                      │
       |         │    ↑ I2C_SCL, I2C_SDA (FPGA config)      │
       |         └──────────────────────────────────────────┘
       │
   ┌───▼───────┐                 ┌──────────────────────────────────┐
   │   FPGA    │ ←─/RX_TX_MODE───│ BLOCK 4: RX/TX RF Switching      │
   │   (J701)  │   (atomic)      │  3× F2912NCGI8 (SP2T switches)   │
   └───────────┘                 │  + ADC SHDN + DAC PD control     │
                                 │  Current: ~3.5µA from FPGA pin   │
                                 └──────────────────────────────────┘
```

---

## ⚡ POWER DISTRIBUTION

### **Supply Rails:**

| Rail              | Voltage | Source                | Load                       | Current |
| ----------------- | ------- | --------------------- | -------------------------- | ------- |
| **+5V**           | 5.0V    | External              | Всі блоки (постійно)       | ~1.2A   |
| **+5V_OUT2**      | 5.0V    | IC602 (Block 6)       | IC601 (LTC6432)            | 50mA    |
| **+3.3V_CLK**     | 3.3V    | U802 (Block 8)        | IC801 VDDA (analog PLL)    | ~80mA   |
| **+1.8V_CLK**     | 1.8V    | U801 (Block 8)        | IC801 VDD + VDDO (digital) | ~85mA   |
| **+3.3V_DIGITAL** | 3.3V    | IC702 (Block 7)       | IC701 digital I/O          | ~50mA   |
| **+3.3V_ANALOG**  | 3.3V    | LDOs (Blocks 1,2,4,5) | DSA + RF Switches          | ~180mA  |
| **+1.8V_ADC**     | 1.8V    | IC703 (Block 7)       | IC701 digital core         | ~40mA   |

### **Power Management:**

- **Всі блоки постійно увімкнені** - немає комутації живлення

**ADC/DAC Control (один сигнал /RX_TX_MODE від FPGA):**

- **RX_TX_MODE** (3.3V LVCMOS via J901.32):
  - Direct connection to IC701.19 (LTC2209 SHDN)
  - Direct connection to F2912 switches (IC401-403.CTL2)
  - RX Mode: LOW → SHDN=LOW (ADC active)
  - TX Mode: HIGH → SHDN=HIGH (ADC shutdown)

- **U902** (SN74LV1T04): Inverter for DAC PD control
  - Input: /RX_TX_MODE (J901.32)
  - Output: IC901.10 (MAX5885 PD, inverted)
  - RX Mode: LOW → PD=HIGH (DAC powered down)
  - TX Mode: HIGH → PD=LOW (DAC active)

### **Power Architecture:**

- **Всі блоки постійно увімкнені** - спрощена архітектура без комутації живлення
- **Перевага:** Миттєве RX/TX перемикання через RF switches (<100 ns)
- **ADC/DAC control:** Shutdown через SHDN/PD pins замість відключення живлення
- **Споживання:** ~1.2A @ 5V постійно (TX і RX режими однакові)

---

## 🎛️ FPGA CONTROL INTERFACE

### **Control Signal Architecture:**

**DSA Control (SPI):**

- **Shared SPI bus** для DSA1 та DSA2 (IC104, IC502)
- **DSA_SCLK, DSA_DATA:** спільна шина (clock + data)
- **DSA_LE1, DSA_LE2:** окремі Latch Enable (chip select) для кожного DSA
- **Pins required:** 4 (економія 2 pins порівняно з окремими SPI)

**Filter Control (Binary):**

- **4-bit binary control** для PE42582A-X switches (IC201, IC203)
- **FLT_V1-V4:** паралельне керування обома switches (синхронне перемикання)
- **Channels:** 16 можливих (8 використовується: 7 BPF + Master LPF)

**System Control (атомарне управління одним сигналом):**

- **/RX_TX_MODE:** єдиний control signal для RX/TX перемикання (J901.32, LVCMOS33)
  - LOW (0V) = RX mode (ADC active, DAC shutdown, RF1 path)
  - HIGH (3.3V) = TX mode (DAC active, ADC shutdown, RF2 path)
  - Керує: F2912 switches (IC401-403.CTL2) + LTC2209 SHDN (direct) + MAX5885 PD (U902)
  - Струм навантаження: ~3.5 µA (margin >2000× від FPGA IOH/IOL)

### **J701 Connector Pinout (LTC2209 LVDS Interface):**

J701 тепер використовується **виключно для LVDS інтерфейсу** з LTC2209 ADC.

| Pin   | Signal    | Direction  | Description                |
| ----- | --------- | ---------- | -------------------------- |
| 1     | GND       | —          | Ground                     |
| 2     | +5V       | Power      | Main power supply          |
| 3-4   | DB0/DB1   | ADC → FPGA | LVDS Data Bus B pair 0     |
| 5-6   | DB2/DB3   | ADC → FPGA | LVDS Data Bus B pair 1     |
| 7-8   | DB4/DB5   | ADC → FPGA | LVDS Data Bus B pair 2     |
| 9-10  | DB6/DB7   | ADC → FPGA | LVDS Data Bus B pair 3     |
| 11-12 | DB8/DB9   | ADC → FPGA | LVDS Data Bus B pair 4     |
| 13-14 | DB10/DB11 | ADC → FPGA | LVDS Data Bus B pair 5     |
| 15-16 | CLKOUT±   | ADC → FPGA | **DCO (GC-capable IO1_7)** |
| 17-18 | DB12/DB13 | ADC → FPGA | LVDS Data Bus B pair 6     |
| 19-20 | DB14/DB15 | ADC → FPGA | LVDS Data Bus B pair 7     |
| 21-22 | DA0/DA1   | ADC → FPGA | LVDS Data Bus A pair 0     |
| 23-24 | DA2/DA3   | ADC → FPGA | LVDS Data Bus A pair 1     |
| 25-26 | DA4/DA5   | ADC → FPGA | LVDS Data Bus A pair 2     |
| 27-28 | DA6/DA7   | ADC → FPGA | LVDS Data Bus A pair 3     |
| 29-30 | DA8/DA9   | ADC → FPGA | LVDS Data Bus A pair 4     |
| 31-32 | DA10/DA11 | ADC → FPGA | LVDS Data Bus A pair 5     |
| 33-34 | DA12/DA13 | ADC → FPGA | LVDS Data Bus A pair 6     |
| 35-36 | DA14/DA15 | ADC → FPGA | LVDS Data Bus A pair 7     |
| 37-38 | GND       | —          | Ground                     |
| 39-40 | +3.3V     | Power      | 3.3V supply                |

**Note:** Всі 17 диференційних пар J701/J12 використовуються для LVDS. DCO призначено на GC-capable пару IO1_7 (pins 15-16).

### **J901 Connector Pinout (DAC Data + Control Signals, 3.3V):**

J901 підключається до ALINX J15 → FPGA BANK25/26 (HR banks, VCCO = 3.3V LVCMOS).
Використовується для **DAC TX data, control signals та ADC overflow**.

| Pin | Signal     | J15 FPGA Pin | Direction      | Description                                        |
| --- | ---------- | ------------ | -------------- | -------------------------------------------------- |
| 1   | GND        | —            | —              | Ground                                             |
| 2   | +5V        | VCC5V        | Power          | 5V supply from ALINX                               |
| 3   | —          | IO2_1N (A11) | —              | Not connected                                      |
| 4   | TX_SEL0    | IO2_1P (A12) | FPGA → DAC     | MAX5885 mode select (IC901.30)                     |
| 5   | TX_D15     | IO2_2N (A13) | FPGA → DAC     | DAC data bit 15 (IC901.33)                         |
| 6   | TX_D14     | IO2_2P (B13) | FPGA → DAC     | DAC data bit 14 (IC901.34)                         |
| 7   | TX_D13     | IO2_3N (A14) | FPGA → DAC     | DAC data bit 13 (IC901.35)                         |
| 8   | TX_D12     | IO2_3P (B14) | FPGA → DAC     | DAC data bit 12 (IC901.36)                         |
| 9   | TX_D11     | IO2_4N (E13) | FPGA → DAC     | DAC data bit 11 (IC901.37)                         |
| 10  | TX_D10     | IO2_4P (E14) | FPGA → DAC     | DAC data bit 10 (IC901.38)                         |
| 11  | TX_D9      | IO2_5N (A15) | FPGA → DAC     | DAC data bit 9 (IC901.39)                          |
| 12  | TX_D8      | IO2_5P (B15) | FPGA → DAC     | DAC data bit 8 (IC901.40)                          |
| 13  | TX_D7      | IO2_6N (C13) | FPGA → DAC     | DAC data bit 7 (IC901.41)                          |
| 14  | TX_D6      | IO2_6P (C14) | FPGA → DAC     | DAC data bit 6 (IC901.44)                          |
| 15  | TX_D5      | IO2_7N (B10) | FPGA → DAC     | DAC data bit 5 (IC901.45)                          |
| 16  | TX_D4      | IO2_7P (C11) | FPGA → DAC     | DAC data bit 4 (IC901.46)                          |
| 17  | TX_D3      | IO2_8N (D14) | FPGA → DAC     | DAC data bit 3 (IC901.47)                          |
| 18  | TX_D2      | IO2_8P (D15) | FPGA → DAC     | DAC data bit 2 (IC901.48)                          |
| 19  | TX_D1      | IO2_9N (F11) | FPGA → DAC     | DAC data bit 1 (IC901.1)                           |
| 20  | TX_D0      | IO2_9P (F12) | FPGA → DAC     | DAC data bit 0 (IC901.2)                           |
| 21  | TX_XOR     | IO2_10N (H13)| FPGA → DAC     | MAX5885 XOR input (IC901.3)                        |
| 22  | —          | IO2_10P (H14)| —              | Not connected                                      |
| 23  | ADC_OF_N   | IO2_11N (G14)| ADC → FPGA     | LTC2209 Overflow- (IC701.59), 47Ω series           |
| 24  | ADC_OF_P   | IO2_11P (G15)| ADC → FPGA     | LTC2209 Overflow+ (IC701.60), 47Ω series           |
| 25  | DSA_LE2    | IO2_12N (F10)| FPGA → DSA2    | PE43711 #2 Latch Enable (via R912, 47Ω)           |
| 26  | DSA_LE1    | IO2_12P (G11)| FPGA → DSA1    | PE43711 #1 Latch Enable (via R913, 47Ω)           |
| 27  | DSA_SCLK   | IO2_13N (H12)| FPGA → DSA     | DSA SPI Clock (via R910, 47Ω)                      |
| 28  | DSA_DATA   | IO2_13P (J12)| FPGA → DSA     | DSA SPI Data (via R911, 47Ω)                       |
| 29  | FLT_V2     | IO2_14N (J14)| FPGA → Filters | Filter select bit 1 (via R918, 47Ω)               |
| 30  | FLT_V1     | IO2_14P (K14)| FPGA → Filters | Filter select bit 0 (via R914, 47Ω)               |
| 31  | FLT_V4     | IO2_15N (K12)| FPGA → Filters | Filter select bit 3 (via R917, 47Ω)               |
| 32  | FLT_V3     | IO2_15P (K13)| FPGA → Filters | Filter select bit 2 (via R915, 47Ω)               |
| 33  | I2C_SDA    | IO2_16N (L13)| FPGA ↔ Si5391B | I2C Data (4.7kΩ pull-up, no series R)              |
| 34  | RX_TX_MODE | IO2_16P (L14)| FPGA → System  | **Atomic RX/TX** (F2912+LTC2209+DAC, via R916 47Ω)|
| 35  | —          | IO2_17N (G10)| —              | Not connected                                      |
| 36  | I2C_SCL    | IO2_17P (H11)| FPGA → Si5391B | I2C Clock (4.7kΩ pull-up, no series R)             |
| 37  | GND        | —            | —              | Ground                                             |
| 38  | GND        | —            | —              | Ground                                             |
| 39  | +3.3V      | VCC_3V3      | Power          | 3.3V supply from ALINX                             |
| 40  | +3.3V      | VCC_3V3      | Power          | 3.3V supply from ALINX                             |

**Note:** Всі сигнали через J901/J15 використовують 3.3V LVCMOS (FPGA BANK25/26, HR banks).
DAC clock (DAC_CLK_P/N) надходить напряму з Si5391B (IC801.34/35) на MAX5885 (IC901.6/7) — НЕ через J901.

**⚠️ УВАГА:** Максимальна швидкість виходу LVCMOS33 на HR банках ~100-125 МГц (DS925).
DAC працює на 153.6 MSPS — це перевищує специфікацію. Див. розділ аналізу сумісності нижче.

---

## 📊 SYSTEM PERFORMANCE TARGETS

### **RF Performance:**

- **Frequency Range:** 9 kHz - 72 MHz
- **Input Impedance:** 50Ω (VSWR < 1.5:1)
- **Gain Range:** -63.5 dB to +15.2 dB (total)
  - DSA1: 0 to -31.75 dB (Block 1)
  - DSA2: 0 to -31.75 dB (Block 5)
  - LNA: +15.2 dB (Block 6, fixed)
- **Noise Figure:** ~3.0 dB @ 0dB attenuation (dominated by LTC6432)
- **IP3:** >+25 dBm (at ADC input)
- **Dynamic Range:** >100 dB (з dual DSA)

### **Filter Bank:**

- **Bands:** 7 switchable bandpass filters + Master LPF
- **Isolation:** >55 dB between bands
- **Insertion Loss**: <3 dB per band

---

## 🔧 KEY DESIGN FEATURES

### **1. Dual DSA Architecture**

- **Wide dynamic range control:** 0 to -63.5 dB 0.25 dB steps
- **Prevents ADC overload** для сильних сигналів
- **Maintains low noise floor** для слабких сигналів

### **2. LTC6432 LNA/Driver Integration** ⭐

- **Combines LNA + ADC driver** в одній мікросхемі
- **Fixed gain** +15.2 dB (no adjustment needed)
- **Low noise figure 3.0 dB** для чутливості
- **High IP3 +33 dBm** для лінійності
- **Dual balunr achitecture** для optimal impedance matching
- **Differential outputs** native для LTC2209

### **3. Switched Filter Bank**

- **7 programmable bands** для селективності
- **Dual SP8T switches** (input + output) для isolation
- **Master LPF** широкосмуговий діапазон 9 kHz - 72 MHz для цифрового аналізу і роботи із сигналом

---

### 4. Power Distribution

- **Окремі LDO** для кожного блоку
- **Ultra-low noise TPS7A9401DSCR** для LTC6432
- **Dual LC filtering** в power rails LNA
- **Ferrite beads + bypass capacitors** на всіх rails

### **5. Differential Signal Path**

- **Balanced transmission** від Block 1 до ADC
- **Common-mode noise rejection**
- **Optimal ADC driving** з T602 balun

---

## 📂 DOCUMENTATION FILES

| File                                       | Description                       | Status           |
| ------------------------------------------ | --------------------------------- | ---------------- |
| **PROJECT_OVERVIEW_v2.0.md**               | Цей файл - загальний опис проєкту | ✅ Complete      |
| **Block1_Input_DSA_Documentation.md**      | Детальна документація Block 1     | 🔄 To be created |
| **Block2_Filter_Bank_Documentation.md**    | Детальна документація Block 2     | 🔄 To be created |
| **Block4_Power_Control_Documentation.md**  | Детальна документація Block 4     | 🔄 To be created |
| **Block5_DSA2_Documentation.md**           | Детальна документація Block 5     | 🔄 To be created |
| **Block6_LTC6432_Driver_Documentation.md** | Детальна документація Block 6 ⭐  | 🔄 To be created |
| **Block7_LTC2209_ADC_Documentation.md**    | Детальна документація Block 7     | 🔄 To be created |

---

## 🔄 VERSION HISTORY

| Version | Date           | Changes                                                                        |
| ------- | -------------- | ------------------------------------------------------------------------------ |
| **1.0** | 2026-01-07     | Initial design з ADA4937 driver + PGA103+ LNA                                  |
| **2.0** | **2026-01-09** | **Major redesign: видалено Block 4, замінено Block 6 на LTC6432**              |
| **2.1** | **2026-01-10** | **Додано новий Block 4 (RX/TX Power Control з TPS22976)**                      |
| **2.2** | **2026-01-11** | **Block 4: Програмна інверсія в FPGA замість IC402**                           |
| **2.3** | **2026-01-11** | **Додано Block 8 (Clock Generation з Si5391B)**                                |
| **2.4** | **2026-01-13** | **Block 7: Оновлено документацію ADC (153.6 MSPS, CMOS mode)**                 |
| **2.5** | **2026-01-13** | **Додано TX тракт: Block 9 (MAX5885 DAC) + Block 10 (LTC6432 TX driver)**      |
| **2.6** | **2026-01-14** | **Нова TX архітектура: видалено Block 10, спільний RX/TX тракт через Block 6** |
| **2.7** | **2026-01-18** | **Atomic RX/TX control: один сигнал /RX_TX_MODE керує switches + ADC/DAC**     |
| **2.8** | **2026-01-25** | **LVDS interface for LTC2209, 3.3V control signals, removed IC704**            |
| **2.9** | **2026-01-25** | **Block 6: 3-stage RF choke, optimized de-Qing, new baluns**                   |

### **Version 2.9 Changes:**

1. ✅ **Block 6 RF Choke redesigned:** 3-stage замість 2-stage для безперервного покриття
   - L601/L604: 470µH (SWPA8065S471M) - LF coverage
   - L611/L612: 47µH (SWPA4030S470MT) - MF coverage (NEW)
   - L602/L603: 560nH (SWPA252010SR56NT) - HF coverage
2. ✅ **De-Qing resistors optimized:** R603=390Ω, R620=82Ω, R604=54Ω
3. ✅ **Shunt resistors added:** R622/R623 (2.2kΩ) для C613/C614
4. ✅ **Baluns updated:** T601=TTWB-2-A (1:2), T602=CX2041NLT (1:1), BW=50kHz-200MHz
5. ✅ **Total DCR:** 1.86Ω → VCC=4.87V (в межах специфікації LTC6432)

### **Version 2.8 Changes:**

1. ✅ **LTC2209 LVDS interface:** Замінено CMOS single-ended на LVDS differential
   - 16 differential data pairs (DA0-DA15, DB0-DB15)
   - 1 differential clock pair (CLKOUTA/CLKOUTB) на GC-capable IO1_7
   - DDR @ 153.6 MHz = 307.2 Mbps per lane
   - FPGA internal 100Ω termination (DIFF_TERM_ADV TERM_100)
2. ✅ **ADC_OF moved to J901.21/22:** Overflow signal на 3.3V HR banks
3. ✅ **F2912 LOGICCTL=GND:** 3.3V logic mode для прямого контролю з FPGA
4. ✅ **Removed IC704 (U704):** Level shifter більше не потрібен
5. ✅ **All control signals on J901:** DSA, FLT, RX_TX_MODE, I2C на 3.3V LVCMOS
6. ✅ **Updated J701 pinout:** Тільки LVDS інтерфейс (17 differential pairs)

### **Version 2.7 Changes:**

1. ✅ **Atomic RX/TX control:** об'єднано TX_RX_SWITCH та RX_TX_MODE в один сигнал
2. ✅ **Один pin FPGA** (/RX_TX_MODE, J901.32) керує всім RX/TX перемиканням:
   - 3× F2912 RF switches (IC401-403.CTL2)
   - LTC2209 SHDN (direct 3.3V connection)
   - MAX5885 PD (через U902 inverter)
3. ✅ **Control current budget:** ~3.5 µA (margin >2000× від FPGA IOH/IOL)
4. ✅ **F2912 1-pin mode:** ModeCTL=VCC, LogicCTL=GND (3.3V logic)
5. ✅ **Звільнено pin 33** для майбутнього використання

### **Version 2.6 Changes:**

1. ✅ **Видалено Block 10** (окремий TX Driver більше не потрібен)
2. ✅ **Оновлено Block 4:** RF Switching замість Power Control
   - IC401, IC402, IC403: F2912NCGI8 (SP3T RF switches)
   - RF switches control (оновлено до atomic в v2.7)
   - <100 ns switching time, >74 dB isolation
3. ✅ **Оновлено Block 9:** Правильна топологія DAC output stage
   - R903/R904 (25Ω): DC return path для current-output DAC
   - Parallel decoupling: C915/C916 (1µF || 100nF), C917/C918 (1µF || 100nF)
   - T902: RF transformer з center tap bypass (C919)
4. ✅ **Нова TX архітектура:** Спільний RX/TX тракт через Block 2-3-5-6
   - TX signal: DAC → IC401 → Filter Bank → LTC6432 → IC402/IC403 → TX Antenna
   - RX signal: Antenna → Filter Bank → LTC6432 → IC402/IC403 → ADC
5. ✅ **Power Management:** Всі блоки постійно увімкнені, ADC/DAC керуються через SHDN/PD pins
6. ✅ **Оновлено Block Structure table:** 9 blocks замість 10

### **Version 2.5 Changes:**

1. ✅ **Додано Block 9** (DAC & FPGA Interface для TX)
2. ✅ **IC901** MAX5885EGM+D: 16-bit DAC (153.6 MSPS update rate)
3. ✅ **Reference circuit:** R901 = 2kΩ → Full-scale current 19.2 mA
4. ✅ **U901** LP5912-3.3DRV: Ultra-low noise LDO для DAC
5. ✅ **J901** 2×20 header: TX data/control interface до FPGA

### **Version 2.4 Changes:**

1. ✅ **Оновлено Block 7 документацію** з повною інформацією про LTC2209
2. ✅ **Sample rate: 153.6 MSPS** (було 122.88 MHz → зараз 153.6 MHz)
3. ✅ **Додано Control Pins Configuration table** з обгрунтуванням кожного pin
4. ✅ **Пояснення CMOS vs LVDS:** Pin count limitation (J701 має тільки 40 pins)
5. ✅ **Dual Supply Architecture:** VDD=3.3V (analog), OVDD=1.8V (digital outputs)
6. ✅ **Обгрунтування режиму:** DITH=GND (краща SNR), PGA=GND (gain=1)
7. ✅ **Оновлено всі згадки 122.88 MHz → 153.6 MHz** в документі

### **Version 2.3 Changes:**

1. ✅ **Додано Block 8** (Clock Generation)
2. ✅ **IC801** Si5391B-A-GM: Ultra-low jitter clock generator (<50 fs RMS)
3. ✅ **X801** 38.4 MHz TCXO: Integer Mode reference (без fractional spurs)
4. ✅ **Dual LDO architecture:** U801 (1.8V) + U802 (3.3V) з LP5912 (12µVRMS noise)
5. ✅ **I2C programmability:** Address 0x68, ClockBuilder Pro configuration
6. ✅ **Output:** 122.88 MHz differential → LTC2209 ADC clock input
7. ✅ **Power filtering:** Ferrite beads + decoupling capacitors для мінімального джиттера

### **Version 2.2 Changes:**

1. ✅ **Видалено IC402** (74LVC1G04 inverter) з Block 4
2. ✅ **Програмна інверсія в FPGA:** 2 pins (RX_TX_MODE, RX_TX_MODE_NOT)
3. ✅ **Покращена надійність:** VOL = 0V, VOH = 1.8V (ідеальні margins)
4. ✅ **Оновлено J701 pinout:** Pin 32 (RX_TX_MODE), Pin 33 (RX_TX_MODE_NOT)

### **Version 2.1 Changes:**

1. ✅ **Додано Block 4** (RX/TX Power Control and Indication Circuit)
2. ✅ **IC401** TPS22976DPUR: Dual-channel load switch для керування живленням
3. ✅ **+5V_RX та +5V_TX rails:** окреме живлення для RX і TX трактів
4. ✅ **LED індикація:** Green (RX mode), Blue (TX mode)

### **Versio- **2.0 Changes:\*\*

1. ❌ **Видалено Block 4** (PGA103+ LNA + bypass switches)
2. ✅ **Замінено IC601** з ADA4937-1YCPZ-R7 на **LTC6432AIUF-15#PBF**
3. ✅ **Додано input balun **T601\*\* (ADT2-1T+ 1:2)
4. ✅ **Додано output balun **T602\*\* (ADT2-1T+ 1:2) для VCM biasing
5. ✅ **Simplified architecture:** один chip замість LNA + driver
6. ✅ **Fixed gain**+15.2 dB\*\* (no VGA needed)
7. ✅ **Improved power supply:** TPS7A2050 ultra-low noise LDO

---

## 📌 NEXT STEPS

### **Design Phase:**

- [ ] Створити детальну документацію для кожного блоку
- [ ] Оптимізувати output matching network Block 6
- [ ] Thermal analysis для LTC6432 (165mA @ 5V)

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

## 📧 CONTACT & SUPPORT

**Datasheet Links:**

- LTC6432: [Analog Devices LTC6432 Product Page](https://www.analog.com/en/products/ltc6432.html)
- LTC2209: [Analog Devices LTC2209 Product Page](https://www.analog.com/en/products/ltc2209.html)
- PE43711A: [pSemi PE43711A Product Page](https://www.psemi.com/)
- PE42582A: [pSemi PE42582A Product Page](https://www.psemi.com/)

---

**End of Document**
