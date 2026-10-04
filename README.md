# Schneider Electric EasyLogic PM2100 / PM2200 / PM2230 ESPHome Integration

Production-grade ESPHome configuration and documentation for monitoring **Schneider Electric EasyLogic PM2100, PM2200, PM2220, and PM2230** three-phase power meters over RS-485 Modbus RTU using an ESP32.

This implementation is architected for modern ESPHome releases (2026.x+), completely eliminates deprecated C++ command items and schema options, handles non-standard Schneider IEC 62053-22 power factor encoding, and organizes telemetry into a multi-tiered polling structure to optimize RS-485 bus throughput.

---

## Key Features

* **Multi-Tiered Polling Architecture:** Divides register polling across four priority controller tiers (5s fast electrical parameters, 20s demand/energy accumulators, 60s harmonics and extreme envelopes, 300s static hardware configuration) to avoid bus saturation.
* **Modbus Exception Code 02/03 Prevention:** Strictly queries valid, contiguous memory segments conforming to Schneider document **DOCA0086EN**, preventing address gap collisions.
* **4-Quadrant Power Factor Correction:** Translates Schneider’s IEC 62053-22 Quadrant 2/3 float encoding ($PF = 2.0 - |PF|$) into standard signed power factors (e.g., negative for capacitive leading loads).
* **Negative-Zero / Unallocated Sentinel Filtering:** Converts Schneider's uninitialized `0x80000000` IEEE 754 sentinels (which manifest as `-0.0` or `-0.000`) into clean zero states.
* **Automated Daily Real-Time Clock (RTC) Sync:** Broadcasts local SNTP network time daily at 03:00 AM directly to the meter's internal hardware clock via atomic Modbus FC16 writes.
* **Authenticated One-Touch Reset Controls:** Exposes Home Assistant buttons for resetting Peak Demand, Min/Max extremes, and cumulative energy using Modbus Function Code 16 (`[Password, Command Code]`).
* **Direct-Connect vs. PT Autodetection:** Decodes register 1999 bitmasks to identify 3PH4W, 3PH3W, 1PH2W, and Potential Transformer (PT) connection modes.
* **Integrated Hardware Alarm Monitoring:** Polls register 6099 to monitor active alarms for Over Voltage, Under Voltage, Over Current, Voltage Unbalance, and Phase Reversal.

---

## Hardware & Wiring

### Recommended Hardware
1. **Microcontroller:** ESP32 Development Board (ESP-WROOM-32 or similar).
2. **RS-485 Transceiver:** 
   * **Automatic Direction Control (Recommended):** Modules based on MAX13487 or XY-017 (only requires `VCC`, `GND`, `TXD`, `RXD`).
   * **Manual Transceiver:** MAX485 / SP3485 modules (requires bridging `DE` and `RE` to a designated GPIO).
   * [XY-017](https://www.aliexpress.com/item/1005002863807590.html) 'Noname' AliExpress TTL <--> RS485 module
   * Simple, cheap module, Supports both 3.3 and 5v, 'No hazzle' module with hardware automatic flow control
   ![XY-017 TTL/RS485 module](https://github.com/htvekov/iem3155_esphome/blob/main/XY-017.png)



### Pinout Connections

| ESP32 Pin | Transceiver Pin | PM2230 Terminal Block | Signal Description |
| :--- | :--- | :--- | :--- |
| **5V / VIN** | `VCC` | — | Module Power (5V required for MAX485) |
| **GND** | `GND` | **Shield / COM** | Common Signal Ground / Reference |
| **GPIO17** | `RO` | — | UART RX (ESP32 Receiver In $\leftarrow$ Transceiver RO) |
| **GPIO16** | `DI` | — | UART TX (ESP32 Driver Out $\rightarrow$ Transceiver DI) |
| — | `A` (or `+`) | **Terminal + (D1)** | RS-485 Non-inverting differential signal |
| — | `B` (or `-`) | **Terminal - (D0)** | RS-485 Inverting differential signal |
| *GPIO4 (Optional)* | `DE` & `RE` tied | — | Hardware Direction Control (if not auto-flow) |

> **Note on RS-485 Polarity:** If the ESPHome log reports `Stop waiting for response from 1 ... after last send`, swap the wires on terminals `+` and `-`.

---

## Meter Setup (Front-Panel Verification)

Prior to polling, confirm the meter's serial port settings match the configuration using the front-panel navigation buttons:

1. Navigate to: **`Menu` $\rightarrow$ `Maint` $\rightarrow$ `Comm`** (Default password: `0000`).
2. Verify:
   * **Protocol:** `Modbus`
   * **Address (Slave ID):** `1`
   * **Baud Rate:** `19200`
   * **Parity:** `EVEN`
   * **Stop Bit:** `1`

---

## Modbus Register Master Reference (PM2200 / PM2230)

*All addresses listed below use 0-based PDU addressing (Schneider 1-based register minus 1).*

| Group | Parameter Description | Schneider Reg (1-based) | ESPHome Address (0-based) | Words | Type | Unit | Modbus FC |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Instantaneous Current** | Phase A Current ($I_a$) | 3000 | 2999 | 2 | FP32 | A | FC03 |
| **Instantaneous Current** | Phase B Current ($I_b$) | 3002 | 3001 | 2 | FP32 | A | FC03 |
| **Instantaneous Current** | Phase C Current ($I_c$) | 3004 | 3003 | 2 | FP32 | A | FC03 |
| **Instantaneous Current** | Neutral Current ($I_n$) | 3006 | 3005 | 2 | FP32 | A | FC03 |
| **Instantaneous Current** | 3-Phase Avg Current | 3010 | 3009 | 2 | FP32 | A | FC03 |
| **Line-to-Line Voltage** | Voltage Phase A-B ($U_{12}$) | 3020 | 3019 | 2 | FP32 | V | FC03 |
| **Line-to-Line Voltage** | Voltage Phase B-C ($U_{23}$) | 3022 | 3021 | 2 | FP32 | V | FC03 |
| **Line-to-Line Voltage** | Voltage Phase C-A ($U_{31}$) | 3024 | 3023 | 2 | FP32 | V | FC03 |
| **Line-to-Line Voltage** | Voltage L-L Average | 3026 | 3025 | 2 | FP32 | V | FC03 |
| **Line-to-Neutral Voltage** | Voltage Phase A-N ($V_1$) | 3028 | 3027 | 2 | FP32 | V | FC03 |
| **Line-to-Neutral Voltage** | Voltage Phase B-N ($V_2$) | 3030 | 3029 | 2 | FP32 | V | FC03 |
| **Line-to-Neutral Voltage** | Voltage Phase C-N ($V_3$) | 3032 | 3031 | 2 | FP32 | V | FC03 |
| **Line-to-Neutral Voltage** | Voltage L-N Average | 3036 | 3035 | 2 | FP32 | V | FC03 |
| **Active Power** | Active Power Phase A ($P_1$) | 3054 | 3053 | 2 | FP32 | kW | FC03 |
| **Active Power** | Active Power Phase B ($P_2$) | 3056 | 3055 | 2 | FP32 | kW | FC03 |
| **Active Power** | Active Power Phase C ($P_3$) | 3058 | 3057 | 2 | FP32 | kW | FC03 |
| **Active Power** | Total Active Power ($P_{tot}$) | 3060 | 3059 | 2 | FP32 | kW | FC03 |
| **Reactive / Apparent** | Total Reactive Power ($Q_{tot}$) | 3068 | 3067 | 2 | FP32 | kvar | FC03 |
| **Reactive / Apparent** | Total Apparent Power ($S_{tot}$) | 3076 | 3075 | 2 | FP32 | kVA | FC03 |
| **Power Factor / Freq** | Total True Power Factor | 3084 | 3083 | 2 | FP32 | -1 to 1 | FC03 |
| **Power Factor / Freq** | Total Displacement PF | 3092 | 3091 | 2 | FP32 | -1 to 1 | FC03 |
| **Power Factor / Freq** | Grid Line Frequency | 3110 | 3109 | 2 | FP32 | Hz | FC03 |
| **Demand Telemetry** | Present Active Demand | 3142 | 3141 | 2 | FP32 | kW | FC03 |
| **Demand Telemetry** | Last Closed Active Demand | 3148 | 3147 | 2 | FP32 | kW | FC03 |
| **Demand Telemetry** | Peak Active Demand | 3154 | 3153 | 2 | FP32 | kW | FC03 |
| **Energy Accumulators** | Active Energy Import | 3204 | 3203 | 2 | FP32 | kWh | FC03 |
| **Energy Accumulators** | Active Energy Export | 3206 | 3205 | 2 | FP32 | kWh | FC03 |
| **Energy Accumulators** | Reactive Energy Delivered | 3208 | 3207 | 2 | FP32 | kvarh | FC03 |
| **Energy Accumulators** | Apparent Energy Received | 3214 | 3213 | 2 | FP32 | kVAh | FC03 |
| **Energy Accumulators** | Net Active Energy | 3222 | 3221 | 2 | FP32 | kWh | FC03 |
| **Energy Accumulators** | TOU Tariff 1 Active | 3254 | 3253 | 2 | FP32 | kWh | FC03 |
| **Energy Accumulators** | TOU Tariff 2 Active | 3256 | 3255 | 2 | FP32 | kWh | FC03 |
| **Harmonic Distortion** | Current THD Phase A | 2130 | 2129 | 2 | FP32 | % | FC03 |
| **Harmonic Distortion** | Current THD Phase B | 2132 | 2131 | 2 | FP32 | % | FC03 |
| **Harmonic Distortion** | Current THD Phase C | 2134 | 2133 | 2 | FP32 | % | FC03 |
| **Harmonic Distortion** | Voltage THD Phase A-N | 2138 | 2137 | 2 | FP32 | % | FC03 |
| **Harmonic Distortion** | Voltage THD Phase B-N | 2140 | 2139 | 2 | FP32 | % | FC03 |
| **Harmonic Distortion** | Voltage THD Phase C-N | 2142 | 2141 | 2 | FP32 | % | FC03 |
| **Individual Harmonics** | Current 3rd Harmonic ($I_a$) | 2150 | 2149 | 2 | FP32 | % | FC03 |
| **Individual Harmonics** | Current 5th Harmonic ($I_a$) | 2152 | 2151 | 2 | FP32 | % | FC03 |
| **Individual Harmonics** | Current 7th Harmonic ($I_a$) | 2154 | 2153 | 2 | FP32 | % | FC03 |
| **Historical Extremes** | Min Recorded Voltage L-N | 3288 | 3287 | 2 | FP32 | V | FC03 |
| **Historical Extremes** | Max Recorded Voltage L-N | 3308 | 3307 | 2 | FP32 | V | FC03 |
| **Historical Extremes** | Min Recorded Power Factor | 3348 | 3347 | 2 | FP32 | -1 to 1 | FC03 |
| **Historical Extremes** | Min Recorded Frequency | 3374 | 3373 | 2 | FP32 | Hz | FC03 |
| **Historical Extremes** | Max Recorded Frequency | 3376 | 3375 | 2 | FP32 | Hz | FC03 |
| **Hardware Config** | Meter Serial Number | 130 | 129 | 2 | U_DWORD | Integer | FC03 |
| **Hardware Config** | Firmware Revision | 1637 | 1636 | 2 | RAW (4B) | Packed | FC03 |
| **Hardware Config** | Hardware Revision | 1639 | 1638 | 1 | U_WORD | Integer | FC03 |
| **Hardware Config** | System Wiring Type | 2000 | 1999 | 1 | RAW (2B) | Enum | FC03 |
| **Hardware Config** | CT Primary Rating | 2004 | 2003 | 1 | U_WORD | A | FC03 |
| **Hardware Config** | CT Secondary Rating | 2005 | 2004 | 1 | U_WORD | A | FC03 |
| **Alarm Bitmask** | Active Alarm Register 1 | 6100 | 6099 | 1 | U_WORD | Bits | FC03 |
| **Command Protocol** | Password & Command Execution| 5249 | 5248 | 2 | U_WORD | Vector | FC16 |
| **Clock Synchronize** | RTC Date & Time Broadcast | 1836 | 1835 | 3 | U_WORD | Packed | FC16 |

<img width="468" height="870" alt="image" src="https://github.com/user-attachments/assets/94d813dd-79c5-41b1-bbed-262aa98eab21" />
<img width="463" height="302" alt="image" src="https://github.com/user-attachments/assets/b434e129-0a8b-4527-8fad-f282b8c54ae4" />
<img width="467" height="732" alt="image" src="https://github.com/user-attachments/assets/72ea81b2-cca6-4d0d-a2fb-76ff9fe2947b" />



---

## Home Assistant Integration & Energy Dashboard

### Entity Categorization
Once flashed, Home Assistant automatically sorts entities into logical categories on the Device page:
* **Primary Sensors:** Real-time Volts, Amps, Active/Reactive/Apparent Power, and Grid Frequency.
* **Configuration:** One-touch execution buttons (`Reset Peak Demand`, `Reset Min-Max Extremes`, `Reset Cumulative Energy`, and `Sync Clock Now`).
* **Diagnostic:** Link Health, Firmware Version, Serial Number, Hardware Revision, Phase Loss Anomaly, Active Alarms, and Historical Min/Max Envelopes.

### Energy Dashboard Configuration
To integrate the PM2230 into the official Home Assistant Energy Dashboard:
1. Navigate to: **Settings $\rightarrow$ Dashboards $\rightarrow$ Energy**.
2. Under **Electricity Grid**:
   * **Grid Consumption:** Add `sensor.pm2230_active_energy_delivered` (Import).
   * **Return to Grid:** Add `sensor.pm2230_active_energy_received` (Export).

---

## License

MIT License. Designed and maintained for industrial automation and smart home energy monitoring.


















