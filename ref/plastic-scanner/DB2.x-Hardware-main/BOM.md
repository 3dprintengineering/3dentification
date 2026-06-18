# Bill of Materials — Plastic Scanner DB2.x

Extracted from the KiCad netlist (`PCB/PCB KiCad/PCB KiCad.xml`): **74 component instances across 44 unique part lines.**

> **Note:** This is the upstream Plastic Scanner Development Board v2.x reference design that 3dentification is based on — not a custom 3dentification board. The DB2.x README flags it as an early development prototype, so treat this as the reference design rather than a guaranteed match to any assembled board.

## Active components / ICs

| Qty | Ref | Part | Function |
|----|------|------|----------|
| 1 | A1 | Arduino UNO R3 | Host MCU |
| 1 | U2 | ADS1256IDBT | 24-bit ΔΣ ADC (reads photodiode, SPI) |
| 1 | U5 | TLC59208FIPWR | 8-channel LED driver (I²C) |
| 1 | U1 | OPA2376 (xxD) | Dual precision op-amp (transimpedance front-end) |
| 1 | U3 | LM285D-2.5 | 2.5 V voltage reference |
| 1 | U4 | AMS1117-3.3 | 3.3 V LDO regulator |
| 1 | Y1 | 7.68 MHz crystal | ADS1256 clock (HC49-U) |
| 1 | FB1 | Ferrite bead | Supply filtering |

## Optoelectronics

| Qty | Ref | Part | Function |
|----|------|------|----------|
| 1 | D1 | Photodiode 0090-3111-185 (1206) | NIR detector |
| 8 | D4–D11 | NIR LEDs: 855, 940, 1050, 1200, 1300, 1460, 1550, 1650 nm (1206) | Spectroscopy illumination |
| 1 | D2 | Blue LED (0805) | Status |
| 1 | D3 | Red LED (0805) | Status |

## Resistors (all 0805 SMD) — 32 total

| Qty | Refs | Value |
|----|------|-------|
| 7 | R9,R10,R12,R14–R17 | 100 Ω |
| 5 | R11,R13,R21,R22,R23 | 0 Ω (jumpers) |
| 4 | R7,R8,R18,R19 | 10 kΩ |
| 3 | R27,R30,R31 | 150 Ω |
| 2 | R20,R24 | 1 kΩ |
| 2 | R1,R2 | 240 kΩ (TIA feedback) |
| 2 | R3,R4 | 301 Ω |
| 2 | R5,R6 | 49.9 Ω |
| 1 | R25 | 52 Ω |
| 1 | R26 | 200 Ω |
| 1 | R28 | 220 Ω |
| 1 | R29 | 240 Ω |
| 1 | R32 | 40 Ω |

## Capacitors (all 0805 SMD) — 21 total

| Qty | Refs | Value |
|----|------|-------|
| 6 | C1,C7,C13,C14,C16,C18 | 100 nF |
| 3 | C6,C12,C17 | 22 µF |
| 2 | C5,C8 | 100 pF |
| 2 | C11,C15 | 10 µF |
| 2 | C9,C10 | 18 pF (crystal load) |
| 2 | C2,C3 | 51 pF |
| 1 | C4 | 47 nF |

## Connectors & switches

| Qty | Ref | Part |
|----|------|------|
| 1 | J1 | ADC debug header (1×8) |
| 1 | J2 | LED debug header (1×4) |
| 1 | J3 | Qwiic / JST-SH 1×4 (I²C) |
| 1 | SW1 | Reset push button (6 mm) |
| 1 | SW2 | Trigger push button (6 mm) |

## ⚠️ Firmware/hardware LED mismatch

This reference board populates LEDs at **855, 940, 1050, 1200, 1300, 1460, 1550, 1650 nm**, but the firmware/GUI in this repo is hardcoded to `LEDS = [850, 940, 1050, 890, 1300, 880, 1550, 1650]`. The 1200/1460 nm hardware positions appear as 890/880 nm in software — either the assembled board differs from this reference design, or the software wavelength labels are out of sync with the populated parts. Reconcile these before trusting any wavelength axis.
