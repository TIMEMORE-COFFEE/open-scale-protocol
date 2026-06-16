# Timemore Black Mirror Scale BLE Protocol

> **Product:** Timemore Black Mirror scale family  
> **Version:** v1.0.3  

## Revision history

| Version | Changes | Date |
| --- | --- | --- |
| V1.0.0 | Updated overload rules for weighing command (`0x01`) | 2026.1.9 | 
| v1.0.1 | Updated weighing command (`0x01`) | 2026.2 |
| v1.0.2 | Fixed brew-ratio status values; clarified brew-ratio protocol | 2026.2.13 |
| v1.0.3 | Optimize the protocol description, remove unnecessary content | 2026.6.16 |

## 1. Background

**Transport:** BLE transparent transmission (UART-style pass-through).

**Frame format:** Binary — fixed header + opcode + command byte + payload + CRC.

## 2. BLE

### 2.1 Advertising and scan response

Manufacturer Data in the advertising packet contains broadcast type and device type:

- **Broadcast type:** `01` — whitelist advertising; `02` — general advertising

**Device type codes:**

- `01` DOT TES017

**Scan response:**

Includes device name and related fields.

### 2.2 GATT services

| Service Name | Service UUID | Characteristic Name | Characteristic UUID | Properties |
| --- | --- | --- | --- | --- |
| custom service | 0xFFF0 | Uplink (Notify) | 0xFFF1 | Notify |
| custom service | 0xFFF0 | Downlink (Write) | 0xFFF2 | Write without response |
| Device Information | 0x180A | Model Number String | 0x2A24 | Read |
| Device Information | 0x180A | Firmware Revision String | 0x2A26 | Read |
| Device Information | 0x180A | Manufacturer Name String | 0x2A29 | Read |

### 2.3 Device Information Service

Standard GATT Device Information Service. All characteristics are **Read-only** (UTF-8 strings without null terminator).

| Characteristic | Value (DOT) |
| --- | --- |
| Model Number | `TES017` |
| Firmware Revision | `v1.0.4` |
| Manufacturer Name | `TIMEMORE` |


## 3. Frame format

| Field | Header<br>2 B | Opcode<br>1 B | Cmd<br>1 B | Length<br>2 B | Data₁ | Data₂ | ... | Dataₙ | CRC<br>2 B |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Example | 0xA55A | Notify / Read / Write | Cmd | Payload length |  |  |  |  | CRC over preceding bytes<br>(CRC field reserved) |

The **length** field counts payload bytes only. **CRC-16/IBM** (polynomial `0x8005`, init `0xFFFF`) covers all bytes except the CRC field.

Opcode names are listed in §4 and the command table below.

Multi-byte fields are **big-endian** (MSB first). Example: length 8 → `00 08`.

## 4. Command reference

Opcodes: **Notify** (`0x01`), **Read** (`0x02`), **Write** (`0x03`).

- `0x01` **Notify** — scale pushes updates to the app.

- `0x02` **Read** — app queries scale state; scale responds.

- `0x03` **Write** — app changes settings; scale returns success/failure.

| Flag | Meaning |
| --- | --- |
| **N** | Notify |
| **R** | Read |
| **W** | Write |
| **R/W** | Read and Write |
| **R/W/N** | Read, Write, and Notify |
| **√** | Supported on this product |


| Feature | Cmd | Description | Ops | dot |
| --- | --- | --- | --- | --- |
| Weight, flow, time | 0x01 | Weight, flow, time | N | √ |
| Timer | 0x02 | Timer state: running, paused, reset | R/W/N | √ |
| Battery | 0x05 | Bar count and percentage | R/N | √ |
| Weight unit | 0x06 | g, oz | R/W/N | √ |
| Power off | 0x0B | — | W | √ |
| Device name | 0x0C | UTF-8 string (1–16 bytes) | R/W | √ |
| Tare | 0x0D | — | W | √ |
| Firmware version | 0x10 | ASCII string | R | √ |
| Factory reset | 0x19 | Clear all settings | W | √ |
| Forget device | 0x1A | Clear pairing data | W | √ |

## 5. Protocol Details

### 5.1 Weight, flow rate, timer (`0x01`)

Reported every **100 ms** while the scale is active.

| Field | Header | Opcode | Cmd | Length | Weight | Flow rate | Timer | Overload | CRC |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Example | 0xA55A | 0x01 | 0x01 | 0x0009 | 4 byte<br>Int32 | 2 byte<br>Int16 | 2 byte<br>uInt16 | 1 byte<br>uint8 | 2byte |

**Flow rate:** 0–999, unit 100 mg/s (in oz mode, minimum step 0.01).

**Timer:** 0–3599 seconds.

**Overload flag:** `1` when positive or negative limit exceeded; `0` otherwise.

### 5.2 Timer (`0x02`)

Timer command supports Notify / Read / Write:

- `0x01` — start
- `0x02` — pause
- `0x03` — reset

#### 5.2.1 Notify

Sent once when the timer state changes.

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x01 | 0x02 | 0x0001 | 1byte<br>uint8 | 2byte |

#### 5.2.2 Read

App→scale

| Header | Opcode | Cmd | Length | CRC |
| --- | --- | --- | --- | --- |
| 0xA55A | 0x02 | 0x02 | 0x0000 | 2byte |

scale→App

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x02 | 0x02 | 0x0001 | 1byte<br>uint8 | 2byte |

#### 5.2.3 Write

App→scale

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x03 | 0x02 | 0x0001 | 1byte<br>uint8 | 2byte |

scale→App

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x03 | 0x02 | 0x0001 | `0x00` fail<br>`0x01` success | 2byte |

### 5.3 Battery (`0x05`)

Battery is reported as **bar count** (icon segments on the scale) and **percentage** (0–100).

#### 5.3.1 Notify

| Field | Header | Opcode | Cmd | Length | Bars | Percent | CRC |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Example | 0xA55A | 0x01 | 0x05 | 0x0002 | 1byte<br>uint8 | 1byte<br>uint8 | 2byte |

#### 5.3.2 Read

App→scale

| Header | Opcode | Cmd | Length | CRC |
| --- | --- | --- | --- | --- |
| 0xA55A | 0x02 | 0x05 | 0x0000 | 2byte |

scale→App

| Field | Header | Opcode | Cmd | Length | Bars | Percent | CRC |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Example | 0xA55A | 0x02 | 0x05 | 0x0002 | 1byte<br>uint8 | 1byte<br>uint8 | 2byte |

### 5.4 Weight unit (`0x06`)

**Unit encoding:** `0` — g; `1` — oz

#### 5.4.1 Notify

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x01 | 0x06 | 0x0001 | 1byte<br>uint8 | 2byte |

#### 5.4.2 Read

App→scale

| Header | Opcode | Cmd | Length | CRC |
| --- | --- | --- | --- | --- |
| 0xA55A | 0x02 | 0x06 | 0x0000 | 2byte |

scale→App

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x02 | 0x06 | 0x0001 | 1byte | 2byte |

#### 5.4.3 Write

App→scale

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x03 | 0x06 | 0x0001 | 1byte | 2byte |

scale→App

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x03 | 0x06 | 0x0001 | `0x00` fail<br>`0x01` success | 2byte |

### 5.5 Power off (`0x0B`)

Puts the scale into sleep / shutdown.

#### 5.5.1 Write

App→scale

| Header | Opcode | Cmd | Length | CRC |
| --- | --- | --- | --- | --- |
| 0xA55A | 0x03 | 0x0B | 0x0000 | 2byte |

scale→App

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x03 | 0x0B | 0x0001 | `0x00` fail<br>`0x01` success | 2byte |

### 5.6 Device name (`0x0C`)

Up to 16 bytes (UTF-8).

#### 5.6.1 Read

App→scale

| Header | Opcode | Cmd | Length | CRC |
| --- | --- | --- | --- | --- |
| 0xA55A | 0x02 | 0x0C | 0x0000 | 2byte |

scale→App

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x02 | 0x0C | n | Device name | 2byte |

#### 5.6.2 Write

App→scale

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x03 | 0x0C | n | Device name<br>max 16 B | 2byte |

scale→App

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x03 | 0x0C | 0x0001 | `0x00` fail<br>`0x01` success | 2byte |

### 5.7 Tare (`0x0D`)

#### 5.7.1 Write

App→scale

| Header | Opcode | Cmd | Length | CRC |
| --- | --- | --- | --- | --- |
| 0xA55A | 0x03 | 0x0D | 0x0000 | 2byte |

scale→App

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x03 | 0x0D | 0x0001 | `0x00` fail<br>`0x01` success | 2byte |

### 5.8 Firmware version (`0x10`)

Version string is ASCII characters.

#### 5.8.1 Read

App→scale

| Header | Opcode | Cmd | Length | CRC |
| --- | --- | --- | --- | --- |
| 0xA55A | 0x02 | 0x10 | 0x0000 | 2byte |

scale→App

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x02 | 0x10 | n | Firmware version | 2byte |

### 5.9 Factory reset (`0x19`)

#### 5.9.1 Write

App→scale

| Header | Opcode | Cmd | Length | CRC |
| --- | --- | --- | --- | --- |
| 0xA55A | 0x03 | 0x19 | 0x0000 | 2byte |

scale→App

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x03 | 0x19 | 0x0001 | `0x00` fail<br>`0x01` success | 2byte |

### 5.10 Forget paired device (`0x1A`)

Scale reboots automatically after receiving this command.

#### 5.10.1 Write

App→scale

| Header | Opcode | Cmd | Length | CRC |
| --- | --- | --- | --- | --- |
| 0xA55A | 0x03 | 0x1A | 0x0000 | 2byte |

scale→App

| Header | Opcode | Cmd | Length | Data | CRC |
| --- | --- | --- | --- | --- | --- |
| 0xA55A | 0x03 | 0x1A | 0x0001 | `0x00` fail<br>`0x01` success | 2byte |
