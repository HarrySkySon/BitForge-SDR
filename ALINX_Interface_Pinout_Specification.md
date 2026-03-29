# ALINX AXU2CGB Interface Pinout Specification

## Project: SDR LTC2209 on ALINX AXU2CGB

**Date**: 2026-01-30
**Version**: 1.2

---

## 1. Overview

This document specifies the interface between the SDR project board and ALINX AXU2CGB FPGA development board. The design utilizes two 40-pin expansion connectors:

| Project Connector | ALINX Connector | FPGA Banks     | I/O Voltage   | Purpose                     |
| ----------------- | --------------- | -------------- | ------------- | --------------------------- |
| **J701**          | **J12**         | BANK65, BANK66 | **1.2V** (HP) | LTC2209 LVDS Data Interface |
| **J901**          | **J15**         | BANK25, BANK26 | **3.3V** (HR) | Control Signals, ADC Status |

---

## 2. FPGA Bank Characteristics

### 2.1 HP Banks (High Performance) - J12

| Parameter               | Value                   | Notes                       |
| ----------------------- | ----------------------- | --------------------------- |
| Banks                   | BANK65, BANK66          | Shared with MIPI interface  |
| VCCO                    | 1.2V                    | Set by PMIC                 |
| I/O Standards           | LVDS, LVDS_25, LVCMOS12 | Native differential support |
| Max Frequency           | 1.25 Gbps (LVDS)        | Per Xilinx DS925            |
| Differential Pairs      | 17 available on J12     | IO1_1 through IO1_17        |
| GC (Global Clock) Pairs | 3 pairs on J12          | IO1_6 (E5/D5), IO1_7 (D4/C4), IO1_11 (D7/D6) |

### 2.2 HR Banks (High Range) - J15

| Parameter      | Value                        | Notes               |
| -------------- | ---------------------------- | ------------------- |
| Banks          | BANK25, BANK26               | General purpose I/O |
| VCCO           | 3.3V                         | From LDO regulator  |
| I/O Standards  | LVCMOS33, LVCMOS25, LVCMOS18 | Single-ended only   |
| Max Frequency  | 125 MHz (LVCMOS33)           | Per Xilinx DS925    |
| Available Pins | 32 single-ended              | Directly accessible |

---

## 3. J701 → J12 Pinout (LTC2209 LVDS Interface)

### 3.1 LTC2209 LVDS Mode Overview

In LVDS mode, LTC2209 outputs 16-bit data using **two parallel differential buses**:

- **Bus A (DA0-DA15)**: 16 lines forming 8 differential pairs
- **Bus B (DB0-DB15)**: 16 lines forming 8 differential pairs
- **Clock (CLKOUTA/CLKOUTB)**: 1 differential pair
- **Overflow (OFA/OFB)**: 1 differential pair

**Total LVDS requirement: 18 differential pairs**

**J12 availability: 17 differential pairs**

**Solution**: Move OF signal to J901 as single-ended 3.3V (use OFA only, tie OFB to ground on project board)

### 3.2 LTC2209 LVDS Pinout Reference

| LTC2209 Pin | CMOS Function | LVDS Function | Pair                  |
| ----------- | ------------- | ------------- | --------------------- |
| 21          | D0-           | DB0           | DB0/DB1               |
| 22          | D0+           | DB1           |                       |
| 23          | D1-           | DB2           | DB2/DB3               |
| 24          | D1+           | DB3           |                       |
| 25          | D2-           | DB4           | DB4/DB5               |
| 26          | D2+           | DB5           |                       |
| 27          | D3-           | DB6           | DB6/DB7               |
| 28          | D3+           | DB7           |                       |
| 29          | D4-           | DB8           | DB8/DB9               |
| 30          | D4+           | DB9           |                       |
| 33          | D5-           | DB10          | DB10/DB11             |
| 34          | D5+           | DB11          |                       |
| 35          | D6-           | DB12          | DB12/DB13             |
| 36          | D6+           | DB13          |                       |
| 37          | D7-           | DB14          | DB14/DB15             |
| 38          | D7+           | DB15          |                       |
| 39          | CLKOUT-       | OFB           | -                     |
| 40          | CLKOUT+       | CLKOUTB       | CLKOUTA/CLKOUTB       |
| 41          | D8-           | CLKOUTA       |                       |
| 42          | D8+           | DA0           | DA0/DA1               |
| 43          | D9-           | DA1           |                       |
| 44          | D9+           | DA2           | DA2/DA3               |
| 45          | D10-          | DA3           |                       |
| 46          | D10+          | DA4           | DA4/DA5               |
| 47          | D11-          | DA5           |                       |
| 48          | D11+          | DA6           | DA6/DA7               |
| 51          | D12-          | DA7           |                       |
| 52          | D12+          | DA8           | DA8/DA9               |
| 53          | D13-          | DA9           |                       |
| 54          | D13+          | DA10          | DA10/DA11             |
| 55          | D14-          | DA11          |                       |
| 56          | D14+          | DA12          | DA12/DA13             |
| 57          | D15-          | DA13          |                       |
| 58          | D15+          | DA14          | DA14/DA15             |
| 59          | OF-           | DA15          |                       |
| 60          | OF+           | OFA           | → J901 (single-ended) |

### 3.3 Differential Pair Mapping to J701/J12

| Netlist Signal  | LTC2209 Pins | J701 Pins | J12 Pins | FPGA Pair | FPGA Pins (P/N)  | GC?    | Notes                      |
| --------------- | ------------ | --------- | -------- | --------- | ---------------- | ------ | -------------------------- |
| ADC_D0_P/N      | 22, 21       | 4, 3      | 4, 3     | IO1_1     | G8 (P), F7 (N)  |        | DB0/DB1 (LVDS Bus B)       |
| ADC_D1_P/N      | 24, 23       | 6, 5      | 6, 5     | IO1_2     | G6 (P), F6 (N)  |        | DB2/DB3                    |
| ADC_D2_P/N      | 26, 25       | 8, 7      | 8, 7     | IO1_3     | E9 (P), D9 (N)  |        | DB4/DB5                    |
| ADC_D3_P/N      | 28, 27       | 10, 9     | 10, 9    | IO1_4     | G5 (P), F5 (N)  |        | DB6/DB7                    |
| ADC_D4_P/N      | 30, 29       | 12, 11    | 12, 11   | IO1_5     | F8 (P), E8 (N)  |        | DB8/DB9                    |
| ADC_D5_P/N      | 34, 33       | 14, 13    | 14, 13   | IO1_6     | E5 (P), D5 (N)  | **GC** | DB10/DB11, clock-capable   |
| **ADC_DCO_P/N** | 40, 39       | 16, 15    | 16, 15   | IO1_7     | D4 (P), C4 (N)  | **GC** | **DCO Clock (current)**    |
| ADC_D6_P/N      | 36, 35       | 18, 17    | 18, 17   | IO1_8     | E4 (P), E3 (N)  |        | DB12/DB13                  |
| ADC_D7_P/N      | 38, 37       | 20, 19    | 20, 19   | IO1_9     | G1 (P), F1 (N)  |        | DB14/DB15                  |
| ADC_D8_P/N      | 42, 41       | 22, 21    | 22, 21   | IO1_10    | F2 (P), E2 (N)  |        | DA0/DA1 (LVDS Bus A)       |
| ADC_D9_P/N      | 44, 43       | 24, 23    | 24, 23   | IO1_11    | D7 (P), D6 (N)  | **GC** | DA2/DA3, clock-capable+QBC |
| ADC_D10_P/N     | 46, 45       | 26, 25    | 26, 25   | IO1_12    | C9 (P), B9 (N)  |        | DA4/DA5                    |
| ADC_D11_P/N     | 48, 47       | 28, 27    | 28, 27   | IO1_13    | B4 (P), A4 (N)  |        | DA6/DA7                    |
| ADC_D12_P/N     | 52, 51       | 30, 29    | 30, 29   | IO1_14    | C6 (P), B6 (N)  |        | DA8/DA9                    |
| ADC_D13_P/N     | 54, 53       | 32, 31    | 32, 31   | IO1_15    | A7 (P), A6 (N)  |        | DA10/DA11                  |
| ADC_D14_P/N     | 56, 55       | 34, 33    | 34, 33   | IO1_16    | C8 (P), B8 (N)  |        | DA12/DA13                  |
| ADC_D15_P/N     | 58, 57       | 36, 35    | 36, 35   | IO1_17    | A9 (P), A8 (N)  |        | DA14/DA15                  |

### 3.4 J701/J12 Complete Pin Assignment

```
J701/J12 Connector (40-pin) — Corrected FPGA Pins per ALINX AXU2CGAB Manual
=============================================================================
GC = Global Clock capable pair (can drive BUFGCE/MMCM/PLL)

Pin 1  ────────────────── GND
Pin 2  ────────────────── VCC5V (5V from ALINX)
Pin 3  ──[ADC_D0_N]────── IO1_1_N  (F7)     ┐ Data pair 0
Pin 4  ──[ADC_D0_P]────── IO1_1_P  (G8)     ┘
Pin 5  ──[ADC_D1_N]────── IO1_2_N  (F6)     ┐ Data pair 1
Pin 6  ──[ADC_D1_P]────── IO1_2_P  (G6)     ┘
Pin 7  ──[ADC_D2_N]────── IO1_3_N  (D9)     ┐ Data pair 2
Pin 8  ──[ADC_D2_P]────── IO1_3_P  (E9)     ┘
Pin 9  ──[ADC_D3_N]────── IO1_4_N  (F5)     ┐ Data pair 3
Pin 10 ──[ADC_D3_P]────── IO1_4_P  (G5)     ┘
Pin 11 ──[ADC_D4_N]────── IO1_5_N  (E8)     ┐ Data pair 4
Pin 12 ──[ADC_D4_P]────── IO1_5_P  (F8)     ┘
Pin 13 ──[ADC_D5_N]────── IO1_6_N  (D5)     ┐ Data pair 5   ★ GC
Pin 14 ──[ADC_D5_P]────── IO1_6_P  (E5)     ┘               ★ Clock-Capable
Pin 15 ──[ADC_DCO_N]───── IO1_7_N  (C4)     ┐ DCO Clock     ★ GC
Pin 16 ──[ADC_DCO_P]───── IO1_7_P  (D4)     ┘               ★ Clock-Capable (current)
Pin 17 ──[ADC_D6_N]────── IO1_8_N  (E3)     ┐ Data pair 6
Pin 18 ──[ADC_D6_P]────── IO1_8_P  (E4)     ┘
Pin 19 ──[ADC_D7_N]────── IO1_9_N  (F1)     ┐ Data pair 7
Pin 20 ──[ADC_D7_P]────── IO1_9_P  (G1)     ┘   NOT clock-capable!
Pin 21 ──[ADC_D8_N]────── IO1_10_N (E2)     ┐ Data pair 8
Pin 22 ──[ADC_D8_P]────── IO1_10_P (F2)     ┘
Pin 23 ──[ADC_D9_N]────── IO1_11_N (D6)     ┐ Data pair 9   ★ GC+QBC
Pin 24 ──[ADC_D9_P]────── IO1_11_P (D7)     ┘               ★ Clock-Capable
Pin 25 ──[ADC_D10_N]───── IO1_12_N (B9)     ┐ Data pair 10
Pin 26 ──[ADC_D10_P]───── IO1_12_P (C9)     ┘
Pin 27 ──[ADC_D11_N]───── IO1_13_N (A4)     ┐ Data pair 11
Pin 28 ──[ADC_D11_P]───── IO1_13_P (B4)     ┘
Pin 29 ──[ADC_D12_N]───── IO1_14_N (B6)     ┐ Data pair 12
Pin 30 ──[ADC_D12_P]───── IO1_14_P (C6)     ┘
Pin 31 ──[ADC_D13_N]───── IO1_15_N (A6)     ┐ Data pair 13
Pin 32 ──[ADC_D13_P]───── IO1_15_P (A7)     ┘
Pin 33 ──[ADC_D14_N]───── IO1_16_N (B8)     ┐ Data pair 14
Pin 34 ──[ADC_D14_P]───── IO1_16_P (C8)     ┘
Pin 35 ──[ADC_D15_N]───── IO1_17_N (A8)     ┐ Data pair 15
Pin 36 ──[ADC_D15_P]───── IO1_17_P (A9)     ┘
Pin 37 ────────────────── GND
Pin 38 ────────────────── GND
Pin 39 ────────────────── VCC_3V3_BUCK4 (3.3V from ALINX)
Pin 40 ────────────────── VCC_3V3_BUCK4 (3.3V from ALINX)
```

### 3.5 Clock-Capable (GC) Pairs on J12

**IMPORTANT:** Only GC-designated FPGA pins can properly drive clock buffers (BUFGCE, MMCM, PLL).
The ADC DCO clock **must** be assigned to one of these pairs:

| J12 Pins | FPGA Pair | FPGA Pins (P/N) | Full Xilinx Pin Name                    | Currently Assigned |
| -------- | --------- | ---------------- | --------------------------------------- | ------------------ |
| 13, 14   | IO1_6     | E5 (P), D5 (N)  | IO_L14P/N_T2L_N2_GC_66                 | ADC_D5 (data)      |
| **15, 16** | **IO1_7** | **D4 (P), C4 (N)** | **IO_L11P/N_T1U_N8_GC_66**          | **ADC_DCO (clock)** |
| 23, 24   | IO1_11    | D7 (P), D6 (N)  | IO_L13P/N_T2L_N0_GC_QBC_66             | ADC_D9 (data)      |

All other J12 pairs (IO1_1-IO1_5, IO1_8-IO1_10, IO1_12-IO1_17) are **regular I/O** — NOT clock-capable.
Pins 19/20 (IO1_9, G1/F1 = `IO_L1P/N_T0L_N0_DBC_66`) are **NOT GC** and must NOT carry the DCO clock.

### 3.5 Pair Utilization Summary

| Resource                  | Available | Used   | Free         |
| ------------------------- | --------- | ------ | ------------ |
| Differential pairs on J12 | 17        | 17     | 0            |
| - Clock pair (GC)         | 1         | 1      | 0            |
| - Data pairs              | 16        | 16     | 0            |
| Overflow (OF)             | -         | → J901 | Single-ended |

**Note**: All 17 differential pairs on J12 are utilized. Overflow signal moved to J901

---

## 4. J901 → J15 Pinout (Control Signals, 3.3V)

### 4.1 Control Signal Mapping

| Signal     | J901 Pin | J15 Pin | Function          | Destination IC | Series R |
| ---------- | -------- | ------- | ----------------- | -------------- | -------- |
| ADC_OF_P   | 21       | 21      | Overflow+ (OFA)   | LTC2209.60     | 47Ω      |
| ADC_OF_N   | 22       | 22      | Overflow- (OF-)   | LTC2209.59     | 47Ω      |
| DSA_LE1    | 24       | 24      | Latch Enable 1    | PE43711 #1     | 47Ω      |
| DSA_LE2    | 25       | 25      | Latch Enable 2    | PE43711 #2     | 47Ω      |
| DSA_DATA   | 26       | 26      | SPI Data          | PE43711        | 47Ω      |
| DSA_SCLK   | 27       | 27      | SPI Clock         | PE43711        | 47Ω      |
| FLT_V1     | 28       | 28      | Filter Select 1   | PE42582A       | 47Ω      |
| FLT_V2     | 29       | 29      | Filter Select 2   | PE42582A       | 47Ω      |
| FLT_V3     | 30       | 30      | Filter Select 3   | PE42582A       | 47Ω      |
| FLT_V4     | 31       | 31      | Filter Select 4   | PE42582A       | 47Ω      |
| RX_TX_MODE | 32       | 32      | TX/RX Control     | F2912, LTC2209 | 47Ω      |
| I2C_SDA    | 33       | 33      | I2C Data          | Si5391B        | None\*   |
| I2C_SCL    | 34       | 34      | I2C Clock         | Si5391B        | None\*   |

\*I2C lines are open-drain with external pull-ups - no series resistors

**Note**: RX_TX_MODE (3.3V) directly controls F2912 RF switches (LOGICCTL=GND mode) and LTC2209 SHDN pin.

### 4.2 J901/J15 Complete Pin Assignment

```
J901/J15 Connector (40-pin) - Control Signals (3.3V LVCMOS)
===========================================================

Pin 1  ────────────── GND
Pin 2  ────────────── VCC (3.3V)
Pin 3  ────────────── Reserved (DAC interface)
Pin 4  ────────────── Reserved (DAC interface)
Pin 5  ────────────── Reserved (DAC interface)
Pin 6  ────────────── Reserved (DAC interface)
Pin 7  ────────────── Reserved (DAC interface)
Pin 8  ────────────── Reserved (DAC interface)
Pin 9  ────────────── Reserved (DAC interface)
Pin 10 ────────────── Reserved (DAC interface)
Pin 11 ────────────── Reserved (DAC interface)
Pin 12 ────────────── Reserved (DAC interface)
Pin 13 ────────────── Reserved (DAC interface)
Pin 14 ────────────── Reserved (DAC interface)
Pin 15 ────────────── Reserved (DAC interface)
Pin 16 ────────────── Reserved (DAC interface)
Pin 17 ────────────── Reserved (DAC interface)
Pin 18 ────────────── Reserved (DAC interface)
Pin 19 ────────────── Reserved
Pin 20 ────────────── Reserved
Pin 21 ──[ADC_OF_P]── BANK25/26 (Overflow+, LTC2209.60)
Pin 22 ──[ADC_OF_N]── BANK25/26 (Overflow-, LTC2209.59)
Pin 23 ────────────── Reserved
Pin 24 ──[DSA_LE1]─── BANK25/26 (PE43711 #1 Latch Enable)
Pin 25 ──[DSA_LE2]─── BANK25/26 (PE43711 #2 Latch Enable)
Pin 26 ──[DSA_DATA]── BANK25/26 (DSA SPI Data)
Pin 27 ──[DSA_SCLK]── BANK25/26 (DSA SPI Clock)
Pin 28 ──[FLT_V1]──── BANK25/26 (Filter Select Bit 0)
Pin 29 ──[FLT_V2]──── BANK25/26 (Filter Select Bit 1)
Pin 30 ──[FLT_V3]──── BANK25/26 (Filter Select Bit 2)
Pin 31 ──[FLT_V4]──── BANK25/26 (Filter Select Bit 3)
Pin 32 ──[RX_TX_MODE] BANK25/26 (TX/RX Mode Control)
Pin 33 ──[I2C_SDA]─── BANK25/26 (Si5391B I2C Data)
Pin 34 ──[I2C_SCL]─── BANK25/26 (Si5391B I2C Clock)
Pin 35 ────────────── Reserved
Pin 36 ────────────── Reserved
Pin 37 ────────────── GND
Pin 38 ────────────── GND
Pin 39 ────────────── VCC (3.3V)
Pin 40 ────────────── VCC (3.3V)
```

---

## 5. Vivado XDC Constraints

### 5.1 LTC2209 LVDS Interface (J12, BANK65/66)

```tcl
# =============================================================================
# LTC2209 LVDS Interface - ALINX AXU2CGB J12 Connector
# BANK65/66 (HP Banks) - VCCO = 1.2V
# Pin numbers verified against ALINX AXU2CGAB User Manual (2026-01-30)
# =============================================================================

# -----------------------------------------------------------------------------
# DCO Clock Input (MUST be on GC-capable pins IO1_7)
# IO_L11P/N_T1U_N8_GC_66 — Global Clock pair in BANK66
# -----------------------------------------------------------------------------
set_property PACKAGE_PIN D4 [get_ports adc_dco_p]
set_property PACKAGE_PIN C4 [get_ports adc_dco_n]
set_property IOSTANDARD LVDS [get_ports adc_dco_p]
set_property IOSTANDARD LVDS [get_ports adc_dco_n]
set_property DIFF_TERM_ADV TERM_100 [get_ports adc_dco_p]

# DCO Clock Constraint (153.6 MHz from LTC2209)
create_clock -period 6.510 -name adc_dco_clk [get_ports adc_dco_p]

# -----------------------------------------------------------------------------
# LVDS Data Bus D[15:0] — 16 differential pairs
# -----------------------------------------------------------------------------
# D0 - IO1_1 (J12 pins 3/4)
set_property PACKAGE_PIN G8 [get_ports {adc_data_p[0]}]
set_property PACKAGE_PIN F7 [get_ports {adc_data_n[0]}]

# D1 - IO1_2 (J12 pins 5/6)
set_property PACKAGE_PIN G6 [get_ports {adc_data_p[1]}]
set_property PACKAGE_PIN F6 [get_ports {adc_data_n[1]}]

# D2 - IO1_3 (J12 pins 7/8)
set_property PACKAGE_PIN E9 [get_ports {adc_data_p[2]}]
set_property PACKAGE_PIN D9 [get_ports {adc_data_n[2]}]

# D3 - IO1_4 (J12 pins 9/10)
set_property PACKAGE_PIN G5 [get_ports {adc_data_p[3]}]
set_property PACKAGE_PIN F5 [get_ports {adc_data_n[3]}]

# D4 - IO1_5 (J12 pins 11/12)
set_property PACKAGE_PIN F8 [get_ports {adc_data_p[4]}]
set_property PACKAGE_PIN E8 [get_ports {adc_data_n[4]}]

# D5 - IO1_6 (J12 pins 13/14) ★ GC-capable (IO_L14_GC_66)
set_property PACKAGE_PIN E5 [get_ports {adc_data_p[5]}]
set_property PACKAGE_PIN D5 [get_ports {adc_data_n[5]}]

# D6 - IO1_8 (J12 pins 17/18)
set_property PACKAGE_PIN E4 [get_ports {adc_data_p[6]}]
set_property PACKAGE_PIN E3 [get_ports {adc_data_n[6]}]

# D7 - IO1_9 (J12 pins 19/20) — NOT clock-capable
set_property PACKAGE_PIN G1 [get_ports {adc_data_p[7]}]
set_property PACKAGE_PIN F1 [get_ports {adc_data_n[7]}]

# D8 - IO1_10 (J12 pins 21/22)
set_property PACKAGE_PIN F2 [get_ports {adc_data_p[8]}]
set_property PACKAGE_PIN E2 [get_ports {adc_data_n[8]}]

# D9 - IO1_11 (J12 pins 23/24) ★ GC-capable (IO_L13_GC_QBC_66)
set_property PACKAGE_PIN D7 [get_ports {adc_data_p[9]}]
set_property PACKAGE_PIN D6 [get_ports {adc_data_n[9]}]

# D10 - IO1_12 (J12 pins 25/26)
set_property PACKAGE_PIN C9 [get_ports {adc_data_p[10]}]
set_property PACKAGE_PIN B9 [get_ports {adc_data_n[10]}]

# D11 - IO1_13 (J12 pins 27/28)
set_property PACKAGE_PIN B4 [get_ports {adc_data_p[11]}]
set_property PACKAGE_PIN A4 [get_ports {adc_data_n[11]}]

# D12 - IO1_14 (J12 pins 29/30)
set_property PACKAGE_PIN C6 [get_ports {adc_data_p[12]}]
set_property PACKAGE_PIN B6 [get_ports {adc_data_n[12]}]

# D13 - IO1_15 (J12 pins 31/32)
set_property PACKAGE_PIN A7 [get_ports {adc_data_p[13]}]
set_property PACKAGE_PIN A6 [get_ports {adc_data_n[13]}]

# D14 - IO1_16 (J12 pins 33/34)
set_property PACKAGE_PIN C8 [get_ports {adc_data_p[14]}]
set_property PACKAGE_PIN B8 [get_ports {adc_data_n[14]}]

# D15 - IO1_17 (J12 pins 35/36)
set_property PACKAGE_PIN A9 [get_ports {adc_data_p[15]}]
set_property PACKAGE_PIN A8 [get_ports {adc_data_n[15]}]

# Apply LVDS standard and termination to all data pins
set_property IOSTANDARD LVDS [get_ports adc_data_p[*]]
set_property IOSTANDARD LVDS [get_ports adc_data_n[*]]
set_property DIFF_TERM_ADV TERM_100 [get_ports adc_data_p[*]]

# -----------------------------------------------------------------------------
# Timing Constraints for LVDS Data
# -----------------------------------------------------------------------------
# DDR data is center-aligned with DCO clock
set_input_delay -clock adc_dco_clk -max 0.5 [get_ports adc_data_p[*]]
set_input_delay -clock adc_dco_clk -min -0.5 [get_ports adc_data_p[*]]
set_input_delay -clock adc_dco_clk -max 0.5 -clock_fall [get_ports adc_data_p[*]] -add_delay
set_input_delay -clock adc_dco_clk -min -0.5 -clock_fall [get_ports adc_data_p[*]] -add_delay
```

### 5.2 Control Signals (J15, BANK25/26)

```tcl
# =============================================================================
# Control Signals - ALINX AXU2CGB J15 Connector
# BANK25/26 (HR Banks) - VCCO = 3.3V
# =============================================================================

# -----------------------------------------------------------------------------
# PE43711 DSA SPI Interface
# -----------------------------------------------------------------------------
set_property PACKAGE_PIN [TBD] [get_ports dsa_sclk]
set_property PACKAGE_PIN [TBD] [get_ports dsa_data]
set_property PACKAGE_PIN [TBD] [get_ports dsa_le1]
set_property PACKAGE_PIN [TBD] [get_ports dsa_le2]
set_property IOSTANDARD LVCMOS33 [get_ports dsa_*]
set_property SLEW SLOW [get_ports dsa_*]
set_property DRIVE 4 [get_ports dsa_*]

# -----------------------------------------------------------------------------
# LTC2209 I2C Interface
# -----------------------------------------------------------------------------
set_property PACKAGE_PIN [TBD] [get_ports i2c_scl]
set_property PACKAGE_PIN [TBD] [get_ports i2c_sda]
set_property IOSTANDARD LVCMOS33 [get_ports i2c_*]
set_property PULLUP TRUE [get_ports i2c_*]

# -----------------------------------------------------------------------------
# Filter Control
# -----------------------------------------------------------------------------
set_property PACKAGE_PIN [TBD] [get_ports {flt_v[0]}]
set_property PACKAGE_PIN [TBD] [get_ports {flt_v[1]}]
set_property PACKAGE_PIN [TBD] [get_ports {flt_v[2]}]
set_property PACKAGE_PIN [TBD] [get_ports {flt_v[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports flt_v[*]]
set_property SLEW SLOW [get_ports flt_v[*]]
set_property DRIVE 4 [get_ports flt_v[*]]

# -----------------------------------------------------------------------------
# RX/TX Mode Control
# -----------------------------------------------------------------------------
set_property PACKAGE_PIN [TBD] [get_ports rx_tx_mode]
set_property IOSTANDARD LVCMOS33 [get_ports rx_tx_mode]
set_property SLEW SLOW [get_ports rx_tx_mode]
set_property DRIVE 4 [get_ports rx_tx_mode]

# -----------------------------------------------------------------------------
# ADC Overflow Status (LTC2209 OF+/OF- on J901.21/22)
# -----------------------------------------------------------------------------
set_property PACKAGE_PIN [TBD] [get_ports adc_of_p]
set_property PACKAGE_PIN [TBD] [get_ports adc_of_n]
set_property IOSTANDARD LVCMOS33 [get_ports adc_of_p]
set_property IOSTANDARD LVCMOS33 [get_ports adc_of_n]
```

---

## 6. Interface Block Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          SDR PROJECT BOARD                                   │
│                                                                              │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │ LTC2209  │    │ PE43711  │    │ PE43711  │    │  F2912   │              │
│  │   ADC    │    │  DSA #1  │    │  DSA #2  │    │ Switches │              │
│  │  (LVDS)  │    │          │    │          │    │  (×3)    │              │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘              │
│       │               │               │               │                     │
│       │ LVDS          │ SPI           │ SPI           │ CTL (3.3V)          │
│       │ (17 pairs)    │               │               │ LOGICCTL=GND        │
│       │               │               │               │                     │
│  ┌────┴────┐     ┌────┴───────────────┴───────────────┴────┐               │
│  │  J701   │     │                   J901                   │               │
│  │ (40pin) │     │                 (40pin)                  │               │
│  │         │     │                                          │               │
│  │ LVDS:   │     │ ADC_OF_P ──[47Ω]──→ LTC2209.60          │               │
│  │ DA0-15  │     │ ADC_OF_N ──[47Ω]──→ LTC2209.59          │               │
│  │ DB0-15  │     │ DSA_SCLK ──[47Ω]──→ PE43711             │               │
│  │ DCO     │     │ DSA_DATA ──[47Ω]──→ PE43711             │               │
│  │ (17     │     │ DSA_LE1  ──[47Ω]──→ PE43711 #1          │               │
│  │ pairs)  │     │ DSA_LE2  ──[47Ω]──→ PE43711 #2          │               │
│  │         │     │ FLT_V1-4 ──[47Ω]──→ PE42582A            │               │
│  │         │     │ RX_TX    ──[47Ω]──→ F2912, LTC2209.SHDN │               │
│  │         │     │ I2C_SCL  ─────────→ Si5391B (no R)      │               │
│  │         │     │ I2C_SDA  ─────────→ Si5391B (no R)      │               │
│  └────┬────┘     └────────────────────┬────────────────────┘               │
│       │                               │                                     │
└───────┼───────────────────────────────┼─────────────────────────────────────┘
        │                               │
        │ 40-pin cable                  │ 40-pin cable
        │                               │
┌───────┼───────────────────────────────┼─────────────────────────────────────┐
│       │                               │                                     │
│  ┌────┴────┐                     ┌────┴────┐                               │
│  │   J12   │                     │   J15   │                               │
│  │ (40pin) │                     │ (40pin) │                               │
│  └────┬────┘                     └────┬────┘                               │
│       │                               │                                     │
│       │                               │                                     │
│  ┌────┴────────────┐            ┌────┴────────────┐                        │
│  │   BANK65/66     │            │   BANK25/26     │                        │
│  │   (HP Banks)    │            │   (HR Banks)    │                        │
│  │   VCCO = 1.2V   │            │   VCCO = 3.3V   │                        │
│  │                 │            │                 │                        │
│  │   Native LVDS   │            │   LVCMOS33      │                        │
│  │   100Ω int.term │            │   Single-ended  │                        │
│  │   (DIFF_TERM)   │            │                 │                        │
│  └────────┬────────┘            └────────┬────────┘                        │
│           │                              │                                  │
│           └──────────────┬───────────────┘                                  │
│                          │                                                  │
│               ┌──────────┴──────────┐                                       │
│               │   XCZU2CG-1SFVC784E │                                       │
│               │   Zynq UltraScale+  │                                       │
│               └─────────────────────┘                                       │
│                                                                             │
│                      ALINX AXU2CGB                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Signal Integrity Considerations

### 7.1 LVDS Interface (J701 → J12)

| Parameter              | Specification     | Notes                           |
| ---------------------- | ----------------- | ------------------------------- |
| Differential Impedance | 100Ω ± 10%        | Controlled impedance PCB traces |
| Trace Length Matching  | ≤ 5mm within pair | Critical for signal integrity   |
| Maximum Trace Length   | 150mm             | LTC2209 to FPGA                 |
| Termination            | 100Ω on-die       | FPGA internal DIFF_TERM         |
| Common Mode Voltage    | 1.2V nominal      | Set by LTC2209 driver           |

### 7.2 Control Signals (J901 → J15)

| Parameter         | Specification           | Notes                             |
| ----------------- | ----------------------- | --------------------------------- |
| Logic Level       | 3.3V LVCMOS             | Compatible with all target ICs    |
| Series Resistor   | 47Ω                     | Edge rate control, ESD protection |
| Noise Margin      | 1.0V (high), 0.8V (low) | Excellent noise immunity          |
| Maximum Frequency | 10 MHz (SPI)            | Well within LVCMOS33 capability   |
| I2C Pull-ups      | 4.7kΩ to 3.3V           | Standard I2C specification        |

---

## 8. PCB Design Guidelines

### 8.1 LVDS Traces

1. Route P and N traces as tightly coupled differential pairs
2. Maintain 100Ω differential impedance
3. Length match P and N within 0.5mm
4. Avoid vias in differential pairs where possible
5. Keep differential pairs at least 3x trace width apart
6. Route DCO pair with same length as data pairs (±5mm)

### 8.2 Control Signal Traces

1. Place 47Ω series resistors within 10mm of connector
2. Avoid parallel routing with LVDS pairs
3. I2C traces: keep under 100mm, add 4.7kΩ pull-ups near FPGA
4. Ground plane under all signal traces

---

## 9. Summary of Design

### 9.1 J701/J12 Interface (LVDS, 1.2V HP Banks)

| Resource            | Count | Description                               |
| ------------------- | ----- | ----------------------------------------- |
| Data Pairs (DA/DB)  | 16    | DA0-DA15 + DB0-DB15 (16-bit DDR)          |
| Clock Pair (DCO)    | 1     | CLKOUTA/CLKOUTB on GC-capable IO1_7       |
| **Total LVDS Pairs**| 17    | All J12 differential pairs utilized       |
| FPGA Termination    | 100Ω  | Internal (DIFF_TERM_ADV TERM_100)         |

### 9.2 J901/J15 Interface (3.3V HR Banks)

| Signal      | J901 Pin | Function                                  |
| ----------- | -------- | ----------------------------------------- |
| ADC_OF_P/N  | 21, 22   | LTC2209 Overflow (pins 60/59)             |
| DSA_LE1/2   | 24, 25   | PE43711 Latch Enable (SPI chip select)    |
| DSA_DATA    | 26       | PE43711 SPI Data (shared)                 |
| DSA_SCLK    | 27       | PE43711 SPI Clock (shared)                |
| FLT_V1-V4   | 28-31    | PE42582A Filter Bank Select (4-bit)       |
| RX_TX_MODE  | 32       | F2912 switches + LTC2209 SHDN (3.3V)      |
| I2C_SDA/SCL | 33, 34   | Si5391B I2C Configuration                 |

### 9.3 ADC Interface Configuration

| Parameter        | Value                                            |
| ---------------- | ------------------------------------------------ |
| Interface Type   | LVDS Differential                                |
| Data Lines       | 16 differential pairs (DA0-15 + DB0-15)          |
| Clock            | 1 differential pair (CLKOUTA/CLKOUTB)            |
| Data Rate        | DDR at 153.6 MHz = 307.2 Mbps per pair           |
| Overflow (OF)    | Single-ended on J901 (pins 21/22)                |
| FPGA Termination | 100Ω internal (DIFF_TERM_ADV TERM_100)           |
| LTC2209 LVDS     | VOS = 1.2V, VOD = 350mV (compatible with HP 1.2V)|

---

## 10. Revision History

| Version | Date       | Changes                                                      |
| ------- | ---------- | ------------------------------------------------------------ |
| 1.0     | 2026-01-25 | Initial specification                                        |
| 1.1     | 2026-01-25 | Updated ADC_OF to J901.21/22, corrected control signal table |
| 1.2     | 2026-01-30 | Fixed all FPGA ball numbers per ALINX manual; added 3 GC clock-capable pairs; updated signal names to match netlist; updated XDC constraints |

---

## 11. References

1. ALINX AXU2CGAB User Manual
2. ALINX AXU2CGA_B Schematic (Rev 1.0)
3. Xilinx UltraScale+ Device Data Sheet (DS925)
4. Linear Technology LTC2209 Datasheet
5. Peregrine PE43711 Datasheet
