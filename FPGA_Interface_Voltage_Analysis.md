# FPGA Interface Voltage Level Analysis

## Project: SDR LTC2209 on ALINX AXU2CGB

**Date**: 2026-01-24
**Version**: 1.0

---

## 1. ALINX AXU2CGB Expansion Connectors

### 1.1 Summary Table

| Connector | FPGA Banks | I/O Voltage | Bank Type | Notes |
|-----------|------------|-------------|-----------|-------|
| **J12** | BANK65, BANK66 | **1.2V** | HP (High Performance) | MIPI shares BANK65 at 1.2V |
| **J15** | BANK25, BANK26 | **3.3V** | HR (High Range) | Confirmed in User Manual |

### 1.2 Source Documentation

From ALINX AXU2CGAB User Manual (Part 13: 40-Pin Expansion Header):

> "The IO port of the **J15** expansion port is connected to the ZYNQ chip **BANK25 and BANK26**, and the **level standard is 3.3V**."

From Part 14 (MIPI Camera Interface):

> "The differential signal of MIPI is connected to the HP IO of **BANK64 and 65**, and the **level standard is +1.2V**"

Since BANK65 is shared between MIPI and J12, and MIPI operates at 1.2V, **J12 I/O voltage is 1.2V**.

### 1.3 Schematic References

| Schematic Page | Content |
|----------------|---------|
| Page 4 | BANK25, BANK26 - VCCO connected to VCCIO (3.3V from LDO) |
| Page 5 | BANK64 - VCCO_HP = 1.2V |
| Page 6 | BANK65, BANK66 - VCCO_HP = 1.2V |
| Page 23 | J12, J15 connector pinouts |
| Page 24 | PMIC - power rail generation |

---

## 2. Project Control Signal Mapping

### 2.1 J701 → J12 Connection (1.2V FPGA I/O)

From netlist analysis, the following control signals connect through **J701** to **J12**:

| Signal | J701 Pin | Function | Destination IC | Required Logic Level |
|--------|----------|----------|----------------|---------------------|
| DSA_SCLK | 27 | SPI Clock | PE43711 | 1.8V - 5V |
| DSA_DATA | 26 | SPI Data | PE43711 | 1.8V - 5V |
| DSA_LE1 | 24 | Latch Enable | PE43711 | 1.8V - 5V |
| DSA_LE2 | 25 | Latch Enable | PE43711 | 1.8V - 5V |
| I2C_SCL | 34 | I2C Clock | LTC2209 | 1.8V - 3.3V |
| I2C_SDA | 33 | I2C Data | LTC2209 | 1.8V - 3.3V |
| FLT_V1 | 28 | Filter Select | Relay/Switch | TBD |
| FLT_V2 | 29 | Filter Select | Relay/Switch | TBD |
| FLT_V3 | 30 | Filter Select | Relay/Switch | TBD |
| FLT_V4 | 31 | Filter Select | Relay/Switch | TBD |
| RX_TX_MODE | 32 | TX/RX Control | PE4257 | 1.8V - 5V |

### 2.2 J901 → J15 Connection (3.3V FPGA I/O)

J901 connects to J15 which operates at **3.3V**. Currently used primarily for:
- ADC data bus (D0-D15)
- Clock signals
- OF (Overflow) indicator

---

## 3. Voltage Compatibility Analysis

### 3.1 PROBLEM: 1.2V FPGA Output → 1.8V/3.3V Logic Devices

**FPGA J12 output characteristics (1.2V VCCO):**
- VOH (Output High): ~1.1V typical
- VOL (Output Low): ~0.1V typical

**PE43711 (DSA) input requirements (VDD = 1.8V):**
- VIH (Input High, min): 0.65 × VDD = **1.17V**
- VIL (Input Low, max): 0.35 × VDD = **0.63V**

**Analysis:**
- FPGA VOH (1.1V) is **marginally below** PE43711 VIH (1.17V)
- This creates unreliable logic HIGH detection!

**LTC2209 I2C requirements:**
- Operates at 1.8V or 3.3V logic
- At 1.8V: VIH ≈ 1.17V
- FPGA 1.2V output is marginal

### 3.2 Voltage Margin Calculation

```
PE43711 at VDD = 1.8V:
  VIH (min) = 0.65 × 1.8V = 1.17V
  FPGA VOH = ~1.1V (at 1.2V VCCO)
  Margin = 1.1V - 1.17V = -0.07V  ← NEGATIVE MARGIN!

PE43711 at VDD = 3.3V:
  VIH (min) = 0.65 × 3.3V = 2.145V
  FPGA VOH = ~1.1V
  Margin = 1.1V - 2.145V = -1.045V  ← SEVERE MISMATCH!
```

### 3.3 Risk Assessment

| Scenario | Risk Level | Description |
|----------|------------|-------------|
| 1.2V FPGA → 1.8V logic | **HIGH** | Marginal VIH, temperature-dependent failures |
| 1.2V FPGA → 3.3V logic | **CRITICAL** | Will not work - needs level shifter |
| 3.3V FPGA → 3.3V logic | **LOW** | Compatible (J15 → J901) |

---

## 4. Serial Resistor Impact Analysis

### 4.1 Voltage Drop Calculation

For a series resistor with FPGA output:

**Formula:** V_drop = I_drive × R_series

**Typical scenarios:**

| R_series | I_drive | V_drop | V_output (from 1.2V) |
|----------|---------|--------|---------------------|
| 22Ω | 4mA | 0.088V | 1.112V |
| 33Ω | 4mA | 0.132V | 1.068V |
| 47Ω | 4mA | 0.188V | 1.012V |
| 22Ω | 8mA | 0.176V | 1.024V |
| 33Ω | 8mA | 0.264V | 0.936V |

**For 3.3V FPGA output (J15):**

| R_series | I_drive | V_drop | V_output |
|----------|---------|--------|----------|
| 22Ω | 8mA | 0.176V | 3.124V |
| 33Ω | 8mA | 0.264V | 3.036V |
| 47Ω | 8mA | 0.376V | 2.924V |

### 4.2 Impact Assessment

**For 1.2V J12 outputs:**
- Series resistors will **worsen** the already marginal VIH situation
- 33Ω at 4mA reduces output to 1.068V (below VIH threshold)
- **NOT RECOMMENDED** without level shifting

**For 3.3V J15 outputs:**
- Series resistors are acceptable
- 33Ω at 8mA gives 3.036V - still well above 3.3V VIH threshold (~2.0V)
- **RECOMMENDED** for ESD protection and signal integrity

---

## 5. Solutions

### 5.1 Option A: Move Control Signals to J15 (3.3V) - RECOMMENDED

**Advantages:**
- Native 3.3V compatibility with PE43711, LTC2209
- No level shifters needed
- Simple PCB modification

**Implementation:**
- Reconnect DSA_*, I2C_*, FLT_*, RX_TX_MODE from J701 to J901
- Use J901 pins currently marked as UNCONNECTED

**J901 Available Pins:**
- Pins 21-36 appear to be unconnected in current design
- Sufficient for all control signals

### 5.2 Option B: Add Level Shifters on Project Board

**For 1.2V → 1.8V conversion:**
- Use bidirectional level shifter (e.g., TXS0108E, SN74LVC8T245)
- I2C requires open-drain compatible level shifter (e.g., PCA9306)

**Circuit example:**
```
FPGA (1.2V) → Level Shifter → PE43711 (1.8V)
     |              |
   VCCO_A=1.2V   VCCO_B=1.8V
```

**Recommended ICs:**

| IC | Type | Channels | Speed | Application |
|----|------|----------|-------|-------------|
| TXS0108E | Auto-direction | 8 | 100Mbps | SPI, GPIO |
| SN74LVC8T245 | Unidirectional | 8 | 420Mbps | High-speed SPI |
| PCA9306 | I2C bidirectional | 2 | 400kHz | I2C bus |

### 5.3 Option C: Verify Actual BANK66 Voltage

BANK66 might be at 1.8V (VCCAUX) instead of 1.2V.

**Verification method:**
1. Measure VCCO_66 voltage on ALINX board
2. Check pins B7, D3, E6 (VCCO_66) on FPGA

If BANK66 = 1.8V:
- Some J12 pins would be 1.8V compatible
- Need to identify which pins are in BANK66 vs BANK65

---

## 6. Serial Resistor Recommendations

### 6.1 For J15 Signals (3.3V) - Project Board

| Signal Type | R_series | Justification |
|-------------|----------|---------------|
| SPI CLK | 22-33Ω | Edge rate control, reflection reduction |
| SPI Data | 22-33Ω | Signal integrity |
| Latch Enable | 33-47Ω | Slow signal, ESD protection |
| I2C | 0Ω (none) | Open-drain with external pull-ups |
| GPIO | 33-47Ω | ESD protection |

### 6.2 For J12 Signals (1.2V) - NOT RECOMMENDED

If level shifters are used:
- Place series resistors **after** level shifter (on 1.8V/3.3V side)
- Do NOT add resistors on 1.2V FPGA side (worsens margin)

---

## 7. Recommended Design Changes

### 7.1 Immediate Actions

1. **Verify BANK66 voltage** by measurement
2. **Evaluate moving control signals** from J701→J901 (Option A)
3. If J701 must be used, **add level shifters**

### 7.2 PCB Modifications

**If moving to J901 (J15, 3.3V):**

| Signal | New J901 Pin | Old J701 Pin |
|--------|--------------|--------------|
| DSA_SCLK | 27 | 27 |
| DSA_DATA | 26 | 26 |
| DSA_LE1 | 24 | 24 |
| DSA_LE2 | 25 | 25 |
| I2C_SCL | 34 | 34 |
| I2C_SDA | 33 | 33 |
| FLT_V1 | 28 | 28 |
| FLT_V2 | 29 | 29 |
| FLT_V3 | 30 | 30 |
| FLT_V4 | 31 | 31 |
| RX_TX_MODE | 32 | 32 |

**Add series resistors on J901 lines:**

```
J901.27 (DSA_SCLK) ──[22Ω]── PE43711
J901.26 (DSA_DATA) ──[22Ω]── PE43711
J901.24 (DSA_LE1) ──[33Ω]── PE43711
J901.25 (DSA_LE2) ──[33Ω]── PE43711
J901.34 (I2C_SCL) ──────── LTC2209 (no series R, has pull-up)
J901.33 (I2C_SDA) ──────── LTC2209 (no series R, has pull-up)
J901.28-31 (FLT_Vx) ──[33Ω]── Filter control
J901.32 (RX_TX_MODE) ──[33Ω]── PE4257
```

---

## 8. Summary

### Critical Findings

1. **J12 (connected to J701) operates at 1.2V** - insufficient for 1.8V/3.3V logic devices
2. **J15 (connected to J901) operates at 3.3V** - compatible with all project ICs
3. **Serial resistors on 1.2V outputs will worsen voltage margins**
4. **Recommended solution: Move control signals to J901 (3.3V)**

### Risk Matrix

| Current Design | Risk | Mitigation |
|----------------|------|------------|
| DSA control via J701 (1.2V) | HIGH | Move to J901 or add level shifter |
| I2C via J701 (1.2V) | HIGH | Move to J901 or add PCA9306 |
| ADC data via J901 (3.3V) | LOW | Add series resistors for protection |

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-01-24 | Initial analysis |
