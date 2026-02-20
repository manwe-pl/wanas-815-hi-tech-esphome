# Wanas 815 V Hi-Tech Modbus RTU Integration (ESPHome)

This repository contains the reverse-engineered Modbus RTU register map and an ESPHome integration configuration for the **Wanas 815 V Hi-Tech** heat recovery ventilation unit (rekuperator). 

The data was gathered by reverse-engineering the Modbus communication of a unit running **firmware version 2.1.18G**, equipped with the official **Wanas Humidifier** and a **Room Humidity/Temperature Sensor**.

## ⚠️ Disclaimer & Hardware Context
The controller board inside the Wanas 815 V Hi-Tech is manufactured by **Tech Controllers** (often based on the ST-340 architecture). The reverse-engineered map provided here should theoretically work for other Wanas/Tech Controllers units, but registers may shift depending on your firmware version and installed add-ons.

## 🔌 Hardware Setup & Wiring

To connect the unit to Home Assistant (or any other BMS), you need an ESP32 connected via an RS-485 transceiver. 

For this project, I successfully used the **KAmodESP32 POW+RS485** evaluation board (which includes a built-in ST485 transceiver and hardware flow control).

### Wiring to the Wanas Motherboard
Locate the **Modbus 2** terminal on the Wanas motherboard. Do NOT use the "TECH RS" port, as it is a closed, proprietary bus used exclusively for the wall-mounted display panel.

| Wanas "Modbus 2" | RS-485 Module | Notes |
| :--- | :--- | :--- |
| `A` | `A` | Data + |
| `B` | `B` | Data - |
| `GND` | `GND` | **Crucial:** Must be connected to equalize ground potential! |

**Important Tip regarding Termination Resistors (R120):** If your ESP32/RS-485 module and the Wanas unit are close to each other (e.g., 1-10 meters of cable), **remove** the 120-ohm termination jumper/resistor from your RS-485 module. On short distances, the resistor can dampen the signal and cause `Timeout (0xE2)` errors.

## ⚙️ Wanas Modbus Settings
Before the ESP32 can communicate with the unit, you must enable Modbus in the Wanas Wall Panel.
Navigate to the **Installer / Service Menu** -> **Modbus RTU** and apply the following settings:
* **Baud Rate:** `9600` (Highly recommended for stability over 115200)
* **Modbus Address:** `1`
* **Mode:** `n-8-1` (None parity, 8 data bits, 1 stop bit)

## 🗺️ Modbus Register Map
*Note: All discovered data resides in the **Holding Registers** (Function Code `0x03` for Read, `0x06` for WriteSingle). `Input Registers` and `Coils` are not populated for these functions.*

### 📊 Sensors (Read-Only)
Temperatures are stored as 16-bit integers. You must multiply them by `0.1` to get the actual float value (e.g., `209` = `20.9 °C`). Negative temperatures use Two's Complement (signed 16-bit integer).

| Register | Name / Description | Format | Notes |
| :---: | :--- | :---: | :--- |
| `000` | Supply Airflow (Nawiew) | `uint16` | Current airflow in m³/h. |
| `001` | Extract Airflow (Wywiew) | `uint16` | Current airflow in m³/h. |
| `003` | Current Gear (Bieg) | `uint16` | Actual running global gear (1, 2, or 3). |
| `004` | Outside Temp. (Czerpnia) | `int16` | Can be negative. Requires signed integer. |
| `005` | Exhaust Temp. (Wyrzutnia) | `int16` | Temperature of air leaving the house. |
| `006` | Inside Unit Temp. | `int16` | **[Add-on]** Temperature inside the unit as "seen" by the optional Humidifier module. |
| `007` | Room Temp. | `int16` | From the main wall panel or room sensor. |
| `029` | Supply Temp. (Nawiewana) | `int16` | Temperature of air supplied to the rooms. |
| `055` | Room Humidity | `uint16` | **[Add-on]** From the room sensor. Multiply by 0.1 (e.g., `389` = 38.9%). |

### 🚩 Operation Flags / Statuses (Read-Only)
These registers appear (value `1`) when a physical action is happening and disappear/reset when it stops.

| Register | Name / Description | Format | Notes |
| :---: | :--- | :---: | :--- |
| `031` | Bypass Status (Main) | `uint16` | `1` when Bypass/GWC flap is physically open. |
| `039` | Bypass Status (Aux) | `uint16` | Appears alongside `031`. |
| `032` | Humidifier Active | `uint16` | **[Add-on]** `1` when the humidifier is actively spraying water (droplet icon on panel). |
| `033` | Heater Active | `uint16` | `1` when the internal heater (e.g., pre-heater) is actively running. |

### 🎛️ Controls (Read / Write)
To change these values, use Modbus Function `0x06` (Write Single Register).

| Register | Name / Description | Valid Values | Notes |
| :---: | :--- | :---: | :--- |
| `002` | Target Gear | `1`, `2`, `3` | Changes the global fan speed. **See Limitations below!** |
| `040` | Humidifier Toggle | `0` (Off) / `1` (On) | **[Add-on]** Enables/Disables the humidifier globally in settings. |
| `045` | Party Mode (Timer) | `0` to `720` | Sets "Party Mode" duration in minutes. Send `0` to immediately stop Party Mode. |

## 🚧 Known Limitations & Quirks
During my reverse engineering, I discovered several hardcoded limitations in the Tech Controllers protocol implementation:

1.  **"Program" Mode Blocks Gear Changes:** If the wall panel is set to "Program" (Harmonogram/Schedule) mode, sending a gear change to Register `002` will be ignored by the main unit. **To control gears via Modbus/Home Assistant, the unit MUST be physically set to "Manual" (Ręczny) mode on the wall panel.**
2.  **No Modbus Control for Program/Manual Switch:** The ability to toggle between "Program" and "Manual" modes is not exposed over the Modbus 2 interface. It is handled exclusively by the proprietary "TECH RS" bus.
3.  **No Independent Fan Control:** Modbus only provides access to the global Gear (1, 2, or 3). If you use the wall panel to manually set independent fan speeds (e.g., Supply on 1, Extract on 3), the Modbus interface completely ignores this. Registers `002` and `003` will remain frozen on the last known global gear.
4.  **No Total OFF Command:** Turning the entire unit off (0 m³/h) is also hidden within the proprietary TECH RS bus and cannot be triggered by writing `0` to the gear register via Modbus.

*Workaround:* For a fully functional Smart Home integration, leave the physical wall panel permanently in "Manual" mode. Replicate your schedules using Home Assistant automations (e.g., changing gears based on time or triggering Party Mode based on bathroom humidity).

## 🚀 ESPHome Integration
A complete, ready-to-use configuration file (`esphome.yaml`) is included in this repository. It maps all the above registers into native Home Assistant entities (Sensors, Switches, Numbers, and Selects) with proper unit conversions and icons.

## 🛠️ Contributing & Final Thoughts
**Disclaimer:** This entire reverse-engineering process and documentation is the result of a single afternoon's work. While I tested it thoroughly on my own hardware, the Modbus implementation of Tech Controllers can be quite quirky and undocumented.

If you find any mistakes, or if you manage to discover more hidden registers (like the missing independent fan controls or the Program/Manual toggle), please open an issue or submit a Pull Request! I gladly welcome any contributions from the community.
