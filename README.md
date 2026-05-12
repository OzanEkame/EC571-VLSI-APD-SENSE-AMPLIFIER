# Resilient & Power-Efficient APD Sense Amplifier for SRAM Design

**EC 571 — Digital VLSI Circuit Design | Boston University | Spring 2026**  
**Authors:** Ozan Ekame Pekgoz, William Borland  
**Instructor:** Prof. Rabia Yazicigil Kirby  

> Based on: Y.-C. Lai and S.-Y. Huang, "A Resilient and Power-Efficient Automatic-Power-Down Sense Amplifier for SRAM Design," IEEE TCAS-II, vol. 55, no. 10, pp. 1031–1035, Oct. 2008.

---

## Overview

SRAM sense amplifiers detect the small voltage differential between bitlines (BL, BLBAR) during a read operation and amplify it to a full logic swing. Two standard approaches exist, each with a critical tradeoff:

- **Latch-type SA** — fast and power-efficient (~217 nW), but susceptible to sensing failure when the bitline swing is smaller than the amplifier's input offset voltage. This failure mode worsens at advanced process nodes due to increased random dopant fluctuation and Vt mismatch.
- **Current-mirror SA** — robust against sensing failure by design, but draws continuous DC bias current while sense-enable (SE) is asserted, regardless of whether sensing has completed.

This project implements an **Automatic Power-Down (APD) sense amplifier**: a current-mirror SA paired with a monitoring circuit that detects output settling and immediately cuts the bias current. We additionally implement the paper's **dual-V_HL** extension, which keeps the APD functional across process corners by providing two selectable Schmitt trigger thresholds.

All design and simulation work was performed in **Cadence Virtuoso** using a 50 nm CMOS technology node.

---

## Results

| Configuration | Power | Delay |
|---|---|---|
| Current-Mirror SA (no APD) | 123.6 µW | 3.1 ns |
| **Current-Mirror SA + APD** | **22.9 µW** | **3.1 ns** |
| Latch-Type SA | ~217 nW | — *(unreliable for small swings)* |

**81% power reduction at identical delay.** The PD signal fires after the output has already settled, placing it entirely off the critical timing path. This result falls within the paper's reported 28–89% range across operating frequencies.

---

## Circuit Architecture

### 6T SRAM Cell

The memory array baseline. Both bitlines are precharged to VDD/2 before each read. When WL goes high, the access transistors connect the cell to the bitlines and drive a small differential. Design parameters: VDD = 1 V, L = 50 nm throughout, inverter NMOS W = 90 nm / PMOS W = 180 nm, access NMOS W = 120 nm, precharge PMOS W = 10 µm.

### Latch-Type Sense Amplifier

Cross-coupled inverters that amplify through positive feedback once SE fires. Total silicon cost: **40 unit transistors**. Correct operation requires the bitline swing to exceed the SA's input offset voltage at the moment SE asserts. The minimum safe swing was characterized at approximately **18 mV**:

$$\Delta V_{BL} = \frac{t_{SE}}{T_R} \times (V_1 - V_2) = \frac{100\ \text{ps}}{1510\ \text{ps}} \times 0.28\ \text{V} \approx 18\ \text{mV}$$

Below this threshold, Vt mismatch between latch transistors causes the amplifier to resolve to the wrong value.

### Current-Mirror Sense Amplifier

A differential amplifier topology with no latching failure mode. BL and BLBAR connect to NMOS input transistors; their drain currents are compared through a PMOS current mirror, and the imbalance drives SAOUT and SAOUTBAR to opposite rails. The circuit will always converge to the correct output.

Two sub-blocks:
- **NMOS source-follower level shifter** — shifts BL/BLBAR down by ~Vt, providing the PMOS loads sufficient VGS headroom when bitlines are precharged near VDD/2.
- **Main differential amplifier** — PMOS current-mirror load with NMOS differential pair.

Total silicon cost: **51 unit transistors**. Power without APD: **123.6 µW**.

### APD Block

Monitors SAOUT and SAOUTBAR and generates the power-down signal (PD, active low) automatically once sensing completes. Three stages:

**1. Schmitt-trigger buffers** — one per SA output. Each has a programmable high-to-low threshold V_HL set by transistor sizing. When either SA output drops below V_HL, the Schmitt trigger fires. Hysteresis prevents false triggering on the slowly-ramping SA output. Sizing follows Filanovsky & Bakes (1994):

$$\frac{k_1}{k_3} = \left(\frac{V_{DD} - V_{HL}}{V_{HL} - V_{TN}}\right)^2 = 16$$

*Note: the Schmitt trigger implementation is inverting. An inverter is inserted before each Schmitt trigger input to maintain correct downstream logic polarity.*

**2. High-skew NAND2** — when either Schmitt trigger output goes low, the NAND output goes high. Sized with weak PMOS (720 nm) and strong NMOS (180 nm) for fast high-going transitions.

**3. Dynamic inverter** — drives PD low. Sized with stronger NMOS than PMOS to pull PD down quickly, and must drive 6 PD transistors in the main amplifier simultaneously.

| Device | Wp | Wn |
|---|---|---|
| Input inverters (×2) | 1.44 µm | 720 nm |
| High-skew NAND | 720 nm | 180 nm |
| Dynamic inverter | 360 nm | 720 nm |

### APD Current-Mirror SA

The main amplifier is identical to the no-APD version with 6 PMOS transistors added — three pairs that pull up SO/SOBAR and SAOUT/SAOUTBAR when PD goes low. Once active, these devices turn off the NMOS differential pair and stop all current flow through the amplifier.

### Dual-V_HL APD

Process variation can shift V_HL by more than ±10%, causing the APD to fire too early (corrupting the read) or too late (wasting power). The dual-V_HL design instantiates two parallel Schmitt trigger pairs — one with an upper threshold, one with a lower threshold. After fabrication, a selection signal (`vhl_sel`) activates whichever pair lands within the safe operating window for that chip's process corner. Selection can be performed manually or automated via at-speed BIST during power-on self-test.

---

## Transistor Sizing

All devices use L = 50 nm. One unit = Wn 90 nm / Wp 180 nm.

**Latch-type SA (40 units total)**
- Cross-coupled inverters: 2W PMOS / 1W NMOS
- SE tail NMOS: 2W — faster common-source pulldown
- SE precharge PMOS: 4W — symmetric timing

**Current-mirror SA (51 units total)**
- Level-shifter devices: 1W baseline
- SE and PD NMOS (stacked in series): 2W each — series connection requires 2W per device to maintain unit equivalent resistance
- SO/SOBAR input NMOS in main amplifier: 4W — empirically required for reliable sensing margin

---

## Repository Contents
