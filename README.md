# Smart IoT Power Bank — Hardware (PD Edition)

A bidirectional USB-C Power Delivery (PD) power bank built around a software-defined PD stack and an I²C buck-boost charger. This repository holds the **KiCad hardware design** (schematic, layout, BOM, fabrication outputs). The firmware lives in a separate repository.

> **Firmware:** [`smart-iot-power-bank-firmware`](#) 

---

<!-- ───────────────────────────────────────────────────────────── -->
<!-- PROJECT IMAGE — replace the placeholder below with a real photo -->
<!-- Suggested: top-down render or photo of the assembled PCB.       -->
<!-- Drop the file into docs/img/ and update the path.               -->
<!-- ───────────────────────────────────────────────────────────── -->

<p align="center">
  <img src="https://github.com/user-attachments/assets/813476f6-0370-4ced-84c9-323b823582fd" alt="Smart IoT Power Bank PCB" width="640">
  <br>
  <em>Smart IoT Power Bank</em>
</p>

---

## 1. Overview

The board implements a professional power-management architecture comparable to those found in modern laptops. The same USB-C port handles **input (sink)** and **output (source/OTG)**, limited to a robust **30 W**. Energetic characterization (V/I telemetry, efficiency) is done entirely with the **internal 16-bit ADCs of the charge controller**, avoiding external sense amplifiers.

| Property | Value |
| :--- | :--- |
| Topology | Bidirectional USB-C PD (sink + source) |
| Battery | 4S1P Li-ion, 14.8 V nominal (~45 Wh) with integrated protection BMS |
| Max power | 30 W (5 V / 9 V / 12 V / 15 V / 20 V profiles) |
| MCU | ESP32-C3 (Wi-Fi/BLE) |
| Charger | TI BQ25792 I²C buck-boost |
| PD PHY | onsemi FUSB302B |
| Logic rail | TPS54202 buck → 3.3 V |
| EDA tool | KiCad |
| Fab house | [Aisler](https://aisler.net) (stackup & design rules tuned to Aisler) |

---

## 2. Hardware Architecture

### A. Power Path & USB-C Communication

| Designator | Part | Function | System Role |
| :--- | :--- | :--- | :--- |
| **U3** | `BQ25792RQMR` (TI) | I²C buck-boost charger | Handles charging (buck) and discharging (OTG/boost) in hardware. Single inductor L2 (10 µH). |
| **U1** | `FUSB302B11MPX` (onsemi) | USB Type-C PD PHY | Monitors the CC pins and translates PD signalling into I²C messages for the MCU. |
| **U6** | `TPS54202HDDCR` (TI) | 2 A step-down | Generates a clean **3.3 V** logic rail from the fluctuating system voltage. Inductor L1 (2.2 µH). |
| **Q1, Q2** | `CSD17581Q3A` (TI) | Dual N-channel MOSFETs | Back-to-back FETs for **reverse-polarity protection** / VBUS load isolation; block reverse current from the USB port into the system. |
| **J2** | Würth WR-MPC4 `64900311122` | Battery connector | Connects the external **4S1P 18650 pack** (14.8 V, ~45 Wh, with internal BMS). |

### B. Protection (TVS / ESD / Clamps)

| Designator | Part | Function | Notes |
| :--- | :--- | :--- | :--- |
| **U2, U4** | `TVS2200DRVR` (TI) | Flat-clamp surge protection | **VBUS TVS** — tight clamping on the high-current bus. |
| **D2, D4** | `TPD4E05U06DQAR` (TI) | 4-channel low-cap ESD | Protects **CC lines and the differential pairs** without loading the high-speed signals. |
| **D1** | `ZLLS350` (Diodes Inc.) | 40 V Schottky | Power-path / bootstrap diode. |
| **Z1** | `MMSZ5242B` | 12 V Zener | Rail clamp. |

### C. Sensing & Intelligence (IoT layer)

| Designator | Part | Function | Engineering benefit |
| :--- | :--- | :--- | :--- |
| **U5** | `ESP32-C3-WROOM-02-H4` (Espressif) | Master MCU | FSM control, software PD stack, I²C master, BLE server. |
| — | BQ25792 (internal) | 16-bit ADC telemetry | Reads V_BUS, I_BUS, V_BAT, I_BAT from the charger — no external current sensors. |
| **(TS pin)** | 10 k NTC | Thermistor | Cell temperature monitoring via the BQ25792 TS pin. |
| **J5** | 1×04 header | OLED / I²C | 0.96" OLED for local PD-profile and efficiency readout. |

### D. Connectors, Switches, Indicators

| Designator | Part | Function |
| :--- | :--- | :--- |
| **J1, J3** | USB-C receptacle (Würth `629722000214`) | Two USB-C ports — see differential-pair routing below. |
| **J4** | 2×03 header | Programming / debug header. |
| **SW1, SW2** | Würth tactile `434121025816` | Boot / reset buttons. |
| **D3** | Green LED (Würth) | Status indication. |

> Full part list with manufacturer numbers, footprints and datasheets: **[`ESP_Smart_IoT_Power_Bank.csv`](ESP_Smart_IoT_Power_Bank.csv)**.

---

## 3. Engineering Features & Physics

### I. Cascaded Power Supply (Logic Isolation)

To avoid system reboots under heavy load, the supply is cascaded:

1. **Primary stage** — the BQ25792 generates the system rail `V_SYS` from the 4S pack.
2. **Secondary stage** — a dedicated **TPS54202** bucks `V_SYS` down to a constant **3.3 V** for the logic. The ESP32 therefore always sees a clean rail, independent of current spikes in the power stage.

### II. Efficient 4S Power Routing (Buck & OTG)

The 4S pack (14.8 V) minimizes conversion losses:

* **Buck mode (sink):** at 20 V input the IC acts as an efficient step-down.
* **Buck/boost mode (source):** in OTG, 5 V / 9 V / 12 V are produced highly efficiently by buck conversion; only 20 V profiles (e.g. laptops) require a slight boost.

### III. Thermal & Electrical Derating

Thanks to the 4S configuration, PCB currents stay **below 3 A** even at 30 W:

```
I_bat_max = P_out / (U_bat · η)
I_bat_max = 30 W / (12.0 V · 0.9) ≈ 2.7 A   (worst case, empty pack)
```

On overheat (NTC > 60 °C) the ESP32 throttles PD power via software re-negotiation down to **5 W (5 V / 1 A)** as a fallback.

---

## 4. PCB Stackup & Layout

**4-layer board, built to Aisler's stackup and design rules.**

| Layer | Role |
| :--- | :--- |
| **Top** | Signal |
| **Inner 1** | GND |
| **Inner 2** | GND |
| **Bottom** | Power / Signal |

The dual inner ground planes give the high-speed pairs a continuous reference and a low-impedance return for the power stage.

### Differential pairs

Two **impedance-controlled 90 Ω differential pairs** (USB 2.0 spec):

1. **USB power port (BQ25792):** backwards-compatible USB data for the power path.
2. **Debug/flash port (ESP32-C3):** USB used to flash and debug the MCU.

Both pairs are protected by the low-capacitance `TPD4E05U06` ESD arrays so the added capacitance does not degrade signal integrity.

### Layout guidelines

* **Ground partitioning** — clean ground concept separating the power stage (BQ25792) from the logic (ESP32 / TPS54202).
* **Copper polygons** — power paths (V_BUS, V_BAT, V_SYS) routed as filled polygons.
* **Thermal management** — the BQ25792 thermal pad uses a dense GND-via array to conduct heat into the inner copper layers.

---

## 5. Design Decisions Log

* **Dual-MOSFET reverse-polarity protection** (Q1/Q2, CSD17581Q3A) instead of a single series diode — near-zero conduction loss on the power path.
* **VBUS TVS** (TVS2200) for surge clamping on the high-current bus.
* **Low-capacitance TVS/ESD** (TPD4E05U06) on CC lines and the differential pairs to preserve signal integrity.
* **Two 90 Ω impedance-controlled diff pairs:** one for backwards-compatible USB power data (BQ), one dedicated to flashing/debugging the MCU.
* **Aisler** chosen as fab house — the stackup and design rules in this project are tuned to Aisler's process.

---



## 6. Status

Hardware design in KiCad. See the firmware repository for the FreeRTOS PD stack.


