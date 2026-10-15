# JBD UP Series (UP16S) Modbus Protocol Specification

This document details the reverse-engineered serial protocol used by the **JBD UP Series** battery management systems (specifically the **JBD UP16S 48V BMS**), found inside **Datou Boss** and other 48V server-rack / wall-mount LiFePO4 batteries.

---

## 1. Protocol Architecture & Physical Layer

Unlike classic JBD / Xiaoxiang / Overkill Solar BMS units that utilize the standard `0xDD` / `0xA5` UART framing, the **JBD UP Series** uses a proprietary **Modbus RTU-like** variant over RS485.

| Parameter | Value |
| :--- | :--- |
| **Physical Interface** | RS485 (Half-Duplex) |
| **Default Baud Rate** | `9600` bps (some models configurable to `19200`) |
| **Data Bits** | `8` |
| **Parity** | `None` |
| **Stop Bits** | `1` |
| **Modbus Slave Address** | `0x01` (Battery 1), `0x02` (Battery 2), etc., configured via front DIP switches |
| **CRC / Checksum** | Modbus CRC-16 (Polynomial: `0xA001`, Initial: `0xFFFF`, Little-Endian) |

---

## 2. Command Framing

### 2.1 Read Request (`Function 0x78`)

To request a telemetry or parameter block, send an 8-byte frame followed by 2 bytes of Modbus CRC-16 (10 bytes total):

```text
[ Slave Addr ] [ 0x78 ] [ Block ID ] [ 0x00 ] [ Block ID ] [ Length ] [ 0x00 ] [ 0x00 ] [ CRC Low ] [ CRC High ]
```

* **Slave Addr:** `0x01` to `0x0F` (matching the battery DIP switch).
* **Function:** `0x78` (Read Register Block).
* **Block ID:** Register group identifier (e.g., `0x10`, `0x18`, `0x1C`, `0x20`).
* **Length:** Requested data length in bytes (hex).

#### Response Format:
```text
[ Slave Addr ] [ 0x78 ] [ Block ID ] [ Length ] [ ... Payload Bytes (Length - 4) ... ] [ CRC Low ] [ CRC High ]
```
> **Note:** The actual telemetry parameters generally begin at **Byte Index 8** of the response payload.

---

### 2.2 Write Request (`Function 0x79`)

Writing parameters requires an authentication password block embedded directly within the request frame:

```text
[ Slave Addr ] [ 0x79 ] [ Block ID ] [ Start Reg ] [ Block ID ] [ End Reg ] [ 0x00 ] [ 0x06 ] [ 0x18 0x4A 0x42 0x44 ] [ Data High ] [ Data Low ] [ CRC Low ] [ CRC High ]
```

* **Function:** `0x79` (Write Register Block).
* **Length Field:** `0x00, 0x06` (6 data bytes: 4-byte key + 2-byte value).
* **Unlock Key:** `0x18 0x4A 0x42 0x44` (ASCII `"JBD"` with leading byte `0x18`).
* **Data High / Low:** 16-bit Big-Endian value to write.

---

## 3. Register Blocks & Telemetry Maps

### 3.1 Block `0x10`: Main Electricals & Cell Telemetry

* **Read Command:** `[Addr, 0x78, 0x10, 0x00, 0x10, 0xA0, 0x00, 0x00, CRC_L, CRC_H]`
* **Expected Response Length:** $\ge$ 122 bytes.

| Byte Index (BE) | Name | Type / Scale | Description |
| :--- | :--- | :--- | :--- |
| `8 - 9` | **Pack Voltage** | `UInt16 / 100` | Total battery voltage (Volts) |
| `12 - 15` | **Pack Current** | `(UInt32 - 300,000) / 100` | Signed pack current (Amps). Positive = Charging, Negative = Discharging |
| `16 - 17` | **State of Charge (SOC)**| `UInt16 / 100` | Battery SOC percentage (0.00% – 100.00%) |
| `18 - 19` | **Remaining Capacity** | `UInt16 / 100` | Remaining capacity in Ah |
| `20 - 21` | **Full Capacity** | `UInt16 / 100` | Calibrated full pack capacity in Ah |
| `24 - 25` | **MOSFET Temperature** | `(UInt16 - 500) / 10` | Power MOSFET heatsink temp (°C) |
| `26 - 27` | **Ambient Temperature** | `(UInt16 - 500) / 10` | Battery enclosure ambient temp (°C) |
| `30 - 31` | **State of Health (SOH)**| `UInt16` | Pack SOH percentage (0 – 100%) |
| `32 - 35` | **Problem / Alarm Mask** | `UInt32` | Bitmask of active protection/alarm flags |
| `40 - 41` | **MOSFET State** | `UInt16` | `1`: Charge ON, `2`: Discharge ON, `3`: Both ON |
| `46 - 47` | **Cycle Count** | `UInt16` | Total charge/discharge cycles |
| `74 - 75` | **Cell Count** | `UInt16` | Number of series cells (typically `16`) |
| `76 - 107` | **Cell Voltages (1–16)**| `16 x UInt16` | Individual cell voltages in **millivolts (mV)** |
| `110 - 117` | **Cell Temps (1–4)** | `4 x (UInt16 - 500) / 10` | 4 cell temperature probes (°C) |
| `118 - 121` | **Balancing Bitmask** | `UInt32` | Bit $i$ indicates Cell $i+1$ active passive balancing |

---

### 3.2 Block `0x18`: Protection Thresholds & Alarms

* **Read Command:** `[Addr, 0x78, 0x18, 0x00, 0x18, 0xD0, 0x00, 0x00, CRC_L, CRC_H]`
* **Expected Response Length:** $\ge$ 216 bytes. All offsets below are relative to payload start (**Byte Index 8**).

| Relative Offset | Parameter | Unit / Scale | Description |
| :--- | :--- | :--- | :--- |
| `+0` | `cell_ov_alarm` | mV | Cell Over-Voltage Alarm threshold |
| `+2` | `cell_ov_alarm_recov`| mV | Cell Over-Voltage Alarm recovery |
| `+4` | `cell_ov_alarm_delay`| Seconds | Alarm trigger delay |
| `+6` | `cell_ov_prot` | mV | Cell Over-Voltage Protection (MOS cutoff) |
| `+8` | `cell_ov_prot_recov` | mV | Cell Over-Voltage Protection recovery |
| `+10` | `cell_ov_prot_delay` | Seconds | Protection trigger delay |
| `+12` | `cell_uv_alarm` | mV | Cell Under-Voltage Alarm |
| `+18` | `cell_uv_prot` | mV | Cell Under-Voltage Protection |
| `+24` | `pack_ov_alarm` | `UInt16 / 100` (V) | Total Pack Over-Voltage Alarm |
| `+30` | `pack_ov_prot` | `UInt16 / 100` (V) | Total Pack Over-Voltage Protection |
| `+36` | `pack_uv_alarm` | `UInt16 / 100` (V) | Total Pack Under-Voltage Alarm |
| `+42` | `pack_uv_prot` | `UInt16 / 100` (V) | Total Pack Under-Voltage Protection |
| `+48` | `chg_oc_alarm` | `UInt16 / 10` (A) | Charge Over-Current Alarm |
| `+54` | `chg_oc_prot` | `UInt16 / 10` (A) | Charge Over-Current Protection |
| `+68` | `dsg_oc_alarm` | `UInt16 / 10` (A) | Discharge Over-Current Alarm |
| `+74` | `dsg_oc_prot` | `UInt16 / 10` (A) | Discharge Over-Current Protection |
| `+88` | `chg_ot_alarm` | `(UInt16 - 500) / 10` | Charge Over-Temperature Alarm (°C) |
| `+94` | `chg_ot_prot` | `(UInt16 - 500) / 10` | Charge Over-Temperature Protection (°C) |
| `+100` | `chg_ut_alarm` | `(UInt16 - 500) / 10` | Charge Under-Temperature Alarm (°C) |
| `+106` | `chg_ut_prot` | `(UInt16 - 500) / 10` | Charge Under-Temperature Protection (°C) |
| `+112` | `dsg_ot_alarm` | `(UInt16 - 500) / 10` | Discharge Over-Temperature Alarm (°C) |
| `+118` | `dsg_ot_prot` | `(UInt16 - 500) / 10` | Discharge Over-Temperature Protection (°C) |
| `+124` | `dsg_ut_alarm` | `(UInt16 - 500) / 10` | Discharge Under-Temperature Alarm (°C) |
| `+130` | `dsg_ut_prot` | `(UInt16 - 500) / 10` | Discharge Under-Temperature Protection (°C) |
| `+136` | `mos_ot_alarm` | `(UInt16 - 500) / 10` | Power MOSFET Over-Temperature Alarm (°C) |
| `+142` | `mos_ot_prot` | `(UInt16 - 500) / 10` | Power MOSFET Over-Temperature Prot (°C) |
| `+148` | `ambient_ot_alarm` | `(UInt16 - 500) / 10` | Ambient Over-Temperature Alarm (°C) |
| `+154` | `ambient_ot_prot` | `(UInt16 - 500) / 10` | Ambient Over-Temperature Prot (°C) |
| `+184` | `volt_diff_alarm` | mV | Cell Voltage Delta Alarm threshold |
| `+190` | `volt_diff_prot` | mV | Cell Voltage Delta Protection threshold |
| `+196` | `soc_low_alarm` | `%` | Low State of Charge Alarm threshold |

---

### 3.3 Block `0x1C`: Balancing, System Calibration & Serial Numbers

* **Read Command:** `[Addr, 0x78, 0x1C, 0x00, 0x1C, 0xA0, 0x00, 0x00, CRC_L, CRC_H]`
* **Expected Response Length:** $\ge$ 140 bytes. Offsets below relative to **Byte Index 8**.

| Relative Offset | Parameter | Type / Scale | Description |
| :--- | :--- | :--- | :--- |
| `+0` | `balance_mode` | `UInt16` | `0`: Static Balance, `1`: Charge Balance, `2`: Always Balance |
| `+4` | `bal_act_volt` | `UInt16` (mV) | Voltage above which cell balancing can trigger |
| `+6` | `bal_act_volt_diff`| `UInt16` (mV) | Minimum cell delta required to trigger balancing |
| `+12` | `full_chg_cal_volt`| `UInt16 / 100` (V) | Full charge calibration pack voltage |
| `+14` | `full_chg_cal_curr`| `UInt16` (mA) | Full charge calibration current threshold |
| `+16` | `bms_sn_code` | ASCII (26 bytes) | BMS hardware serial number string |
| `+46` | `bms_mfg` | `3 x UInt16` | BMS manufacturing date (Year - Month - Day) |
| `+52` | `sn_code` | ASCII (28 bytes) | Battery pack serial number |
| `+82` | `pack_mfg` | `3 x UInt16` | Pack manufacturing date (Year - Month - Day) |
| `+88` | `bar_code` | ASCII (30 bytes) | Factory barcode |
| `+118` | `stand_sleep_volt` | `UInt16` (mV) | Standby auto-sleep voltage threshold |
| `+120` | `stand_sleep_delay`| `UInt16` (min) | Standby auto-sleep delay |
| `+130` | `stand_sleep_curr` | `UInt16 / 10` (A) | Current threshold considered "idle/standby" |

---

### 3.4 Block `0x20`: Inverter Integration Demands & Capacity

* **Read Command:** `[Addr, 0x78, 0x20, 0x00, 0x20, 0x50, 0x00, 0x00, CRC_L, CRC_H]`
* **Expected Response Length:** $\ge$ 60 bytes. Offsets below relative to **Byte Index 8**.

| Relative Offset | Parameter | Type / Scale | Description |
| :--- | :--- | :--- | :--- |
| `+0` | `nominal_cap` | `UInt16 / 100` (Ah) | Rated battery capacity |
| `+2` | `full_cap` | `UInt16 / 100` (Ah) | Learned full capacity |
| `+4` | `remain_cap` | `UInt16 / 100` (Ah) | Current remaining capacity |
| `+6` | `soc` | `UInt16 / 100` (%) | State of Charge |
| `+8` | `soh` | `UInt16` (%) | State of Health |
| `+10` | `cycle_cnt` | `UInt16` | Cycle count |
| `+42` | `chg_demand_volt` | `UInt16 / 10` (V) | Target Charge Voltage sent to inverter |
| `+44` | `chg_demand_curr` | `UInt16 / 10` (A) | Max Charge Current limit sent to inverter |
| `+46` | `dsg_demand_volt` | `UInt16 / 10` (V) | Cutoff Discharge Voltage sent to inverter |
| `+48` | `dsg_demand_curr` | `UInt16 / 10` (A) | Max Discharge Current limit sent to inverter |

---

## 4. Parameter Write Reference (`0x79`)

### 4.1 Balance Mode Selection
* **Target Block:** `0x1C`
* **Register Range:** Start: `0x7C`, End: `0x7E`
* **Values:**
  * `0x0000` = Static Balance
  * `0x0001` = Charge Balance (Recommended default)

**Example Frame (Battery 1, Charge Balance):**
```text
Hex: 01 79 1C 7C 1C 7E 00 06 18 4A 42 44 00 01 [CRC_L] [CRC_H]
```

### 4.2 Charge Demand Voltage
* **Target Block:** `0x20`
* **Register Range:** Start: `0x2A`, End: `0x2C`
* **Scale:** Volts $\times 10$ (e.g., $54.1\text{ V} \rightarrow 541 = \text{0x021D}$)

**Example Frame (Battery 1, Set 54.1V):**
```text
Hex: 01 79 20 2A 20 2C 00 06 18 4A 42 44 02 1D [CRC_L] [CRC_H]
```
