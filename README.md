# Infiniti Q50 CAN Bus Decoding

This is an ongoing effort to reverse engineer the CAN bus on my 2019 Infiniti Q50, decoding as many messages and signals as possible. Everything here was worked out empirically: logging traffic, triggering things in the car (doors, lights, pedals, buttons), and watching which bytes change.

As far as I am aware, the CAN platform from years 2014-2024 should be similar enough to where you can work off of this data but please confirm on your own vehicle before using the DBC file. Some of it may also be shared with previous year Nissan and Infiniti vehicles, but this should only be used as a reference.

> [!WARNING]
> **This is unofficial and incomplete.** None of it comes from Nissan/Infiniti documentation. Some signals are educated guesses, some bit widths and scaling factors are approximate, and some of it may be wrong. Verify anything before relying on it, and **don't use this data for anything safety-critical.**

## How to read

- Everything below is generated from [`q50_main.dbc`](q50_main.dbc). Each section is one message (frame) from the DBC.
- Each message has a header (CAN ID, description, DLC, interval, sender) and a byte/bit grid. Bit 7 is the most significant bit of each byte. Signals are big-endian (Motorola) unless the Position column says little-endian.
- Signals that span multiple bytes are merged across the grid. Where a signal wraps into part of the next byte, the continuation is marked with ↑ or ↓.
- The table under each grid lists the DBC signals. Physical value = raw × Factor + Offset.
- Status says how sure the decode is:
  - **confirmed**: reproduced on the bench (driving the cluster) or on the car, with a visible, repeatable effect.
  - **probable**: strong correlation in real-car captures, but not proven on the bench.
  - **documented**: carried over from earlier notes and not yet re-verified.
  - **candidate**: a guess or a weak correlation. These get a `?` after the name in the grid.
- Expand **Signal notes** under a table for the evidence behind each signal (captures, bench tests, open questions).
- Blank cells haven't been decoded yet. They may be unused, or I just haven't figured them out.
- Enum tables list the values defined for multi-bit signals. Values missing from a table haven't been seen.

Feel free to let me know of any corrections or additions that you may have found on your own vehicle.

- [0x002 Steering Angle](#0x002-steering-angle)
- [0x02A SCM Status](#0x02a-scm-status)
- [0x160 Accelerator Pedal](#0x160-accelerator-pedal)
- [0x174 Transmission (gear / shift)](#0x174-transmission-gear--shift)
- [0x176 Transmission Speeds](#0x176-transmission-speeds)
- [0x177 Transmission Status](#0x177-transmission-status)
- [0x180 Engine RPM](#0x180-engine-rpm)
- [0x182 Engine 2](#0x182-engine-2)
- [0x1A5 AWD](#0x1a5-awd)
- [0x1C3 Unresolved 0x1C3](#0x1c3-unresolved-0x1c3)
- [0x1F9 Engine RPM (copy)](#0x1f9-engine-rpm-copy)
- [0x216 BCM Lights](#0x216-bcm-lights)
- [0x239 Pedal / Cruise Switches](#0x239-pedal--cruise-switches)
- [0x245 Unresolved 0x245](#0x245-unresolved-0x245)
- [0x280 Cluster Status 1 (meter)](#0x280-cluster-status-1-meter)
- [0x284 Front Wheel Speeds](#0x284-front-wheel-speeds)
- [0x285 Rear Wheel Speeds](#0x285-rear-wheel-speeds)
- [0x286 Unresolved 0x286](#0x286-unresolved-0x286)
- [0x292 Forces](#0x292-forces)
- [0x2AA Unresolved 0x2AA](#0x2aa-unresolved-0x2aa)
- [0x2AB Chassis (yaw rate)](#0x2ab-chassis-yaw-rate)
- [0x2B0 Unresolved 0x2B0](#0x2b0-unresolved-0x2b0)
- [0x2B1 ADAS](#0x2b1-adas)
- [0x2DE Cluster Status 2 (meter)](#0x2de-cluster-status-2-meter)
- [0x300 Power Steering Assist](#0x300-power-steering-assist)
- [0x351 Intelligent Key](#0x351-intelligent-key)
- [0x354 Chassis Control (ABS)](#0x354-chassis-control-abs)
- [0x355 Cluster Status 3 (meter)](#0x355-cluster-status-3-meter)
- [0x358 Body / Lamps](#0x358-body--lamps)
- [0x35D Brake Signal](#0x35d-brake-signal)
- [0x385 TPMS](#0x385-tpms)
- [0x3AF Hazard Switch](#0x3af-hazard-switch)
- [0x3EC Engine RPM (meter tach source)](#0x3ec-engine-rpm-meter-tach-source)
- [0x421 Automatic Transmission](#0x421-automatic-transmission)
- [0x4CC ADAS Errors](#0x4cc-adas-errors)
- [0x4DF](#0x4df)
- [0x4F6 Page Slot Prerequisite B](#0x4f6-page-slot-prerequisite-b)
- [0x519 Drive Mode Switch / Startup (raw)](#0x519-drive-mode-switch--startup-raw)
- [0x51B Service Distance (wrong ID?)](#0x51b-service-distance-wrong-id)
- [0x51C Page Slot Prerequisite A](#0x51c-page-slot-prerequisite-a)
- [0x544 Vehicle Speed](#0x544-vehicle-speed)
- [0x54A HVAC 1](#0x54a-hvac-1)
- [0x54B HVAC 2](#0x54b-hvac-2)
- [0x54C HVAC 3 (outside temperature)](#0x54c-hvac-3-outside-temperature)
- [0x54D HVAC 4 (seat heat)](#0x54d-hvac-4-seat-heat)
- [0x551 Engine (coolant / MIL / cruise)](#0x551-engine-coolant--mil--cruise)
- [0x553 Oil Service](#0x553-oil-service)
- [0x56C Chassis Control / Drive Mode](#0x56c-chassis-control--drive-mode)
- [0x56E Telematics](#0x56e-telematics)
- [0x580 Engine 3](#0x580-engine-3)
- [0x5B0 Drive Mode](#0x5b0-drive-mode)
- [0x5C5 Cluster Status 4 (meter)](#0x5c5-cluster-status-4-meter)
- [0x5E4 Power Steering](#0x5e4-power-steering)
- [0x60D BCM](#0x60d-bcm)
- [0x625 BCM / IPDM](#0x625-bcm--ipdm)
- [0x6E2 Unresolved 0x6E2 (tach gate)](#0x6e2-unresolved-0x6e2-tach-gate)

---

## 0x002 Steering Angle

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x002</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Steering Angle</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">10 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" rowspan="2" align="center">STEERING_<wbr>ANGLE (×-0.1 deg)</td>
  </tr>
  <tr>
    <th>1</th>
  </tr>
  <tr>
    <th>2</th>
    <td colspan="8" align="center">STEERING_<wbr>RATE (×4 deg/s)</td>
  </tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>4</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td colspan="4" align="center">COUNTER</td>
  </tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>7</th>
    <td colspan="8" align="center">CHECKSUM</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `STEERING_ANGLE` | Bytes 0–1, little-endian | 16 (signed) | -0.1 | 0 | deg | documented |
| `STEERING_RATE` | Byte 2 | 8 | 4 | 0 | deg/s | probable |
| `COUNTER` | Byte 4 [3:0] | 4 | 1 | 0 |  | probable |
| `CHECKSUM` | Byte 7 | 8 | 1 | 0 |  | probable |

<details>
<summary>Signal notes</summary>

- `STEERING_ANGLE`: signed little-endian, factor -0.1 deg
- `STEERING_RATE`: absolute rate, factor ~4 deg/s
- `CHECKSUM`: observed: B7 = XOR of all nibbles of B0..B6 

</details>

---

## 0x02A SCM Status

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x02A</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">SCM Status</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">6</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>4</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">SCM_<wbr>SYSTEM_<wbr>ERROR</td>
  </tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `SCM_SYSTEM_ERROR` | Byte 4 [0] | 1 | 1 | 0 |  | confirmed |

<details>
<summary>Signal notes</summary>

- don't know what the SCM system is

</details>

---

## 0x160 Accelerator Pedal

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x160</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Accelerator Pedal</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td></td>
    <td align="center">PEDAL_<wbr>FULLY_<wbr>PRESSED?</td>
    <td></td>
    <td></td>
    <td align="center">ENGINE_<wbr>STOPPED</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>3</th>
    <td colspan="8" align="center">ACCEL_<wbr>POSITION (×0.5 %)</td>
  </tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>5</th>
    <td colspan="8" align="center">ACCEL_<wbr>POSITION_<wbr>INV</td>
  </tr>
  <tr>
    <th>6</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">ENGINE_<wbr>RUNNING</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `PEDAL_FULLY_PRESSED` | Byte 0 [6] | 1 | 1 | 0 |  | candidate |
| `ENGINE_STOPPED` | Byte 0 [3] | 1 | 1 | 0 |  | probable |
| `ENGINE_RUNNING` | Byte 6 [3] | 1 | 1 | 0 |  | probable |
| `ACCEL_POSITION` | Byte 3 | 8 | 0.5 | 0 | % | documented |
| `ACCEL_POSITION_INV` | Byte 5 | 8 | 1 | 0 |  | documented |

<details>
<summary>Signal notes</summary>

- `PEDAL_FULLY_PRESSED`: bench 2026-10-12: no visible change
- `ENGINE_STOPPED`: engine-state flag (all real-car captures incl. 60 s IG-on/engine-off: 1 off, 0 running with the engine, one change at start and stop)
- `ENGINE_RUNNING`: engine-state flag (all real-car captures incl. 60 s IG-on/engine-off: 0 off, 1 running with the engine, one change at start and stop)
- `ACCEL_POSITION_INV`: 0xFF - byte 3

</details>

---

## 0x174 Transmission (gear / shift)

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x174</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Transmission (gear / shift)</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" align="center">SHIFT_<wbr>TORQUE_<wbr>CAND?</td>
  </tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>3</th>
    <td colspan="4" align="center">GEAR_<wbr>ACTUAL</td>
    <td colspan="4" align="center">GEAR_<wbr>TARGET</td>
  </tr>
  <tr>
    <th>4</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td colspan="4" align="center">COUNTER</td>
  </tr>
  <tr>
    <th>5</th>
    <td align="center">LOCKUP_<wbr>CAND?</td>
    <td></td>
    <td></td>
    <td></td>
    <td colspan="2" align="center">SHIFT_<wbr>MODE</td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `SHIFT_TORQUE_CAND` | Byte 0 | 8 | 1 | 0 |  | candidate |
| `GEAR_ACTUAL` | Byte 3 [7:4] | 4 | 1 | 0 |  | probable |
| `GEAR_TARGET` | Byte 3 [3:0] | 4 | 1 | 0 |  | probable |
| `COUNTER` | Byte 4 [3:0] | 4 | 1 | 0 |  | probable |
| `SHIFT_MODE` | Byte 5 [3:2] | 2 | 1 | 0 |  | probable |
| `LOCKUP_CAND` | Byte 5 [7] | 1 | 1 | 0 |  | candidate |

<details>
<summary>Signal notes</summary>

- `SHIFT_TORQUE_CAND`: captures: 0xFF except during gear changes (100% of activity within 0.5 s of a shift); torque reduction request?
- `GEAR_ACTUAL`: captures: P/N -> 0xA, R -> 0x9, D -> 1..7 (7-speed AT), M2..M5 -> 2..5
- `GEAR_TARGET`: captures: differs from GEAR_ACTUAL only during a shift (e.g. 0x34 = 3 -> 4)
- `SHIFT_MODE`: captures: follows the 0x56C drive mode (Personal shows its configured mode)
- `LOCKUP_CAND`: captures: set mostly when engine rpm and 0x176 turbine speed are within 30 rpm, not confirmed

</details>

### 0x174 · GEAR_ACTUAL

| Value | Meaning |
|-------|---------|
| 0b0001 (1) | 1 |
| 0b0010 (2) | 2 |
| 0b0011 (3) | 3 |
| 0b0100 (4) | 4 |
| 0b0101 (5) | 5 |
| 0b0110 (6) | 6 |
| 0b0111 (7) | 7 |
| 0b1001 (9) | R |
| 0b1010 (10) | P/N |

### 0x174 · GEAR_TARGET

| Value | Meaning |
|-------|---------|
| 0b0001 (1) | 1 |
| 0b0010 (2) | 2 |
| 0b0011 (3) | 3 |
| 0b0100 (4) | 4 |
| 0b0101 (5) | 5 |
| 0b0110 (6) | 6 |
| 0b0111 (7) | 7 |
| 0b1001 (9) | R |
| 0b1010 (10) | P/N |

### 0x174 · SHIFT_MODE

| Value | Meaning |
|-------|---------|
| 0b00 (0) | Standard |
| 0b01 (1) | Eco |
| 0b10 (2) | Sport |
| 0b11 (3) | Snow |

---

## 0x176 Transmission Speeds

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x176</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Transmission Speeds</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">7</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td colspan="4" align="center">OUTPUT_<wbr>SPEED ↓</td>
  </tr>
  <tr>
    <th>1</th>
    <td colspan="8" align="center">OUTPUT_<wbr>SPEED</td>
  </tr>
  <tr>
    <th>2</th>
    <td colspan="8" rowspan="2" align="center">ENGINE_<wbr>RPM (rpm)</td>
  </tr>
  <tr>
    <th>3</th>
  </tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>6</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td colspan="4" align="center">COUNTER</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `OUTPUT_SPEED` | Byte 0 [3:0] + Byte 1 | 12 | 1 | 0 |  | probable |
| `ENGINE_RPM` | Bytes 2–3 | 16 | 1 | 0 | rpm | probable |
| `COUNTER` | Byte 6 [3:0] | 4 | 1 | 0 |  | probable |

<details>
<summary>Signal notes</summary>

- `OUTPUT_SPEED`: captures: 23.77 x vehicle km/h (r 1.000), i.e. ~driveshaft rpm; 12-bit
- `ENGINE_RPM`: tracks 0x180 through engine start in mainbus_startup; changing it alone did not move the tach. Linked to 0x180. Driving captures: reads 0 while stopped in gear at idle and below engine rpm at low speed, so it behaves as the transmission input (turbine) speed.

</details>

---

## 0x177 Transmission Status

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x177</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Transmission Status</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>1</th>
    <td align="center">MANUAL_<wbr>MODE</td>
    <td></td>
    <td></td>
    <td align="center">PARK</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>4</th>
    <td colspan="8" align="center">SHIFT_<wbr>ACTIVE_<wbr>CAND?</td>
  </tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `PARK` | Byte 1 [4] | 1 | 1 | 0 |  | probable |
| `MANUAL_MODE` | Byte 1 [7] | 1 | 1 | 0 |  | probable |
| `SHIFT_ACTIVE_CAND` | Byte 4 | 8 | 1 | 0 |  | candidate |

<details>
<summary>Signal notes</summary>

- `PARK`: captures: 1 in P, 0 in R/D/M
- `MANUAL_MODE`: captures: 1 exactly in M1..M7
- `SHIFT_ACTIVE_CAND`: captures: 0 except during gear changes (100% within 0.5 s of a shift)

</details>

---

## 0x180 Engine RPM

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x180</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Engine RPM</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" rowspan="2" align="center">ENGINE_<wbr>RPM (×0.125 rpm)</td>
  </tr>
  <tr>
    <th>1</th>
  </tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>5</th>
    <td colspan="8" align="center">ACCEL_<wbr>POSITION (×0.5 %)</td>
  </tr>
  <tr>
    <th>6</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td colspan="4" align="center">COUNTER</td>
  </tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `ENGINE_RPM` | Bytes 0–1 | 16 | 0.125 | 0 | rpm | confirmed |
| `ACCEL_POSITION` | Byte 5 | 8 | 0.5 | 0 | % | documented |
| `COUNTER` | Byte 6 [3:0] | 4 | 1 | 0 |  | documented |

<details>
<summary>Signal notes</summary>

- `ENGINE_RPM`: raw = rpm * 8, big-endian. Setting it also sets the other RPM copies (0x1F9, 0x3EC, 0x176); 0x3EC wins on the meter when present.
- `COUNTER`: README: byte 6 counter; only the low nibble rolls in captures (upper nibble kept).

</details>

---

## 0x182 Engine 2

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x182</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Engine 2</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>1</th>
    <td align="center">COAST_<wbr>CAND?</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>4</th>
    <td colspan="8" align="center">THROTTLE_<wbr>CAND?</td>
  </tr>
  <tr>
    <th>5</th>
    <td></td>
    <td></td>
    <td align="center">ENGINE_<wbr>STOPPED</td>
    <td></td>
    <td colspan="4" align="center">COUNTER</td>
  </tr>
  <tr>
    <th>6</th>
    <td></td>
    <td align="center">BRAKE_<wbr>PRESSED</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `COAST_CAND` | Byte 1 [7] | 1 | 1 | 0 |  | candidate |
| `THROTTLE_CAND` | Byte 4 | 8 | 1 | 0 |  | candidate |
| `COUNTER` | Byte 5 [3:0] | 4 | 1 | 0 |  | documented |
| `ENGINE_STOPPED` | Byte 5 [5] | 1 | 1 | 0 |  | probable |
| `BRAKE_PRESSED` | Byte 6 [6] | 1 | 1 | 0 |  | documented |

<details>
<summary>Signal notes</summary>

- `COAST_CAND`: captures: set only while moving with the accelerator released (2931 of 2983 samples moving); deceleration fuel cut?
- `THROTTLE_CAND`: captures: follows the accelerator (r 0.94 vs 0x180 B5), 0xFF at full throttle; not calibrated
- `ENGINE_STOPPED`: engine-state flag (all real-car captures incl. 60 s IG-on/engine-off: 1 off, 0 running with the engine, one change at start and stop)

</details>

---

## 0x1A5 AWD

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x1A5</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">AWD</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

> **Notes**
>
> - Adding then removing this message triggers AWD error unless B+ is cycled (car forgets it is AWD)

---

## 0x1C3 Unresolved 0x1C3

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x1C3</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Unresolved 0x1C3</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>5</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td colspan="2" align="center">COUNTER</td>
  </tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>7</th>
    <td colspan="8" align="center">CHECKSUM</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `COUNTER` | Byte 5 [1:0] | 2 | 1 | 0 |  | probable |
| `CHECKSUM` | Byte 7 | 8 | 1 | 0 |  | probable |

<details>
<summary>Signal notes</summary>

- `CHECKSUM`: captures: B7 = (sum(B0..B6) + 0xC4) & 0xFF (100%)

</details>

---

## 0x1F9 Engine RPM (copy)

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x1F9</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Engine RPM (copy)</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td></td>
    <td></td>
    <td colspan="3" align="center">ENGINE_<wbr>STATE</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>2</th>
    <td colspan="8" rowspan="2" align="center">ENGINE_<wbr>RPM (×0.125 rpm)</td>
  </tr>
  <tr>
    <th>3</th>
  </tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `ENGINE_STATE` | Byte 0 [5:3] | 3 | 1 | 0 |  | probable |
| `ENGINE_RPM` | Bytes 2–3 | 16 | 0.125 | 0 | rpm | documented |

<details>
<summary>Signal notes</summary>

- `ENGINE_STATE`: captures: same states as 0x551 B5[1:0]; 6 matches START_STATE cranking
- `ENGINE_RPM`: same raw as 0x180 bytes 0-1 frame for frame in captures. Linked to 0x180.

</details>

### 0x1F9 · ENGINE_STATE

| Value | Meaning |
|-------|---------|
| 0b000 (0) | Off (IG off) |
| 0b010 (2) | IG on, stopped |
| 0b100 (4) | Running |
| 0b101 (5) | Running (settled) |
| 0b110 (6) | Cranking |

---

## 0x216 BCM Lights

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x216</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">BCM Lights</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">3</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">20 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="2" align="center">WIPER_<wbr>STATE?</td>
    <td align="center">HOOD_<wbr>OPEN?</td>
    <td></td>
    <td></td>
    <td colspan="3" align="center">OP_<wbr>STATE?</td>
  </tr>
  <tr>
    <th>1</th>
    <td></td>
    <td align="center">DRL_<wbr>ON</td>
    <td align="center">HEADLIGHTS</td>
    <td></td>
    <td align="center">FOG_<wbr>LIGHTS?</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <th>2</th>
    <td colspan="8" align="center">CHECKSUM?</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `WIPER_STATE` | Byte 0 [7:6] | 2 | 1 | 0 |  | candidate |
| `HOOD_OPEN` | Byte 0 [5] | 1 | 1 | 0 |  | candidate |
| `OP_STATE` | Byte 0 [2:0] | 3 | 1 | 0 |  | candidate |
| `DRL_ON` | Byte 1 [6] | 1 | 1 | 0 |  | documented |
| `HEADLIGHTS` | Byte 1 [5] | 1 | 1 | 0 |  | documented |
| `FOG_LIGHTS` | Byte 1 [3] | 1 | 1 | 0 |  | candidate |
| `CHECKSUM` | Byte 2 | 8 | 1 | 0 |  | candidate |

<details>
<summary>Signal notes</summary>

- `WIPER_STATE`: was B0[7:5]; bit 5 is the hood (captures). Real car: bit 6 always 1, bit 7 always 0; bench 2026-10-12: no visible change
- `OP_STATE`: bench 2026-10-12: no visible change
- `DRL_ON`: bench 2026-10-12: meter shows the green parking-light telltale
- `HEADLIGHTS`: bench 2026-10-12: no telltale (lamps follow 0x60D)
- `FOG_LIGHTS`: README: fog lights. Real-car captures: 1 in P and 0 in R/D/M in all 42 captures with P, so on the car it behaves like a P/N position signal; bench 2026-10-12: no telltale (lamps follow 0x60D)
- `CHECKSUM`: observed: last byte = (B0+B1+0x72) & 0xFF in idle capture

</details>

---

## 0x239 Pedal / Cruise Switches

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x239</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Pedal / Cruise Switches</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" align="center">PEDAL?</td>
  </tr>
  <tr>
    <th>1</th>
    <td colspan="8" align="center">PEDAL_<wbr>INV?</td>
  </tr>
  <tr>
    <th>2</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">CRUISE_<wbr>MASTER</td>
    <td align="center">PEDAL_<wbr>RELEASED?</td>
  </tr>
  <tr>
    <th>3</th>
    <td colspan="8" align="center">CRUISE_<wbr>SWITCH?</td>
  </tr>
  <tr>
    <th>4</th>
    <td></td>
    <td align="center">IG_<wbr>READY?</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td colspan="2" align="center">COUNTER</td>
  </tr>
  <tr>
    <th>5</th>
    <td colspan="8" align="center">ACCEL_<wbr>POSITION (×0.5 %)</td>
  </tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `PEDAL` | Byte 0 | 8 | 1 | 0 |  | candidate |
| `PEDAL_INV` | Byte 1 | 8 | 1 | 0 |  | candidate |
| `PEDAL_RELEASED` | Byte 2 [0] | 1 | 1 | 0 |  | candidate |
| `CRUISE_MASTER` | Byte 2 [1] | 1 | 1 | 0 |  | probable |
| `CRUISE_SWITCH` | Byte 3 | 8 | 1 | 0 |  | candidate |
| `COUNTER` | Byte 4 [1:0] | 2 | 1 | 0 |  | documented |
| `IG_READY` | Byte 4 [6] | 1 | 1 | 0 |  | candidate |
| `ACCEL_POSITION` | Byte 5 | 8 | 0.5 | 0 | % | probable |

<details>
<summary>Signal notes</summary>

- `PEDAL`: ~100*raw/255 %, not calibrated
- `PEDAL_INV`: ~0xFF - byte 0
- `PEDAL_RELEASED`: 0x81 released, 0x80 pressed
- `CRUISE_MASTER`: correlates with cruise enabled on meter
- `CRUISE_SWITCH`: idle 0x20; 0x08/0x10 during SET-/RES+ presses
- `IG_READY`: follows the ignition, not the engine: on for the whole IG-on/engine-off capture, cleared right after IG off; already set in remote-start captures
- `ACCEL_POSITION`: captures: equals 0x180 B5 raw (r 0.9997)

</details>

---

## 0x245 Unresolved 0x245

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x245</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Unresolved 0x245</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>3</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">TCS_<wbr>OFF</td>
  </tr>
  <tr>
    <th>4</th>
    <td colspan="8" align="center">COUNTER</td>
  </tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>7</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td colspan="2" align="center">COUNTER2</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `TCS_OFF` | Byte 3 [0] | 1 | 1 | 0 |  | probable |
| `COUNTER` | Byte 4 | 8 | 1 | 0 |  | probable |
| `COUNTER2` | Byte 7 [1:0] | 2 | 1 | 0 |  | probable |

<details>
<summary>Signal notes</summary>

- `TCS_OFF`: captures: 1 exactly while 0x354 TCS_STATE = 2 (TCS OFF), 5 captures; no visible meter effect

</details>

---

## 0x280 Cluster Status 1 (meter)

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x280</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Cluster Status 1 (meter)</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">20 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Meter (cluster)</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">DRIVER_<wbr>UNBUCKLED</td>
    <td></td>
  </tr>
  <tr>
    <th>1</th>
    <td colspan="8" align="center">FUEL_<wbr>LEVEL_<wbr>NORM</td>
  </tr>
  <tr>
    <th>2</th>
    <td colspan="2" align="center">↑ FUEL_<wbr>LEVEL_<wbr>NORM</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>4</th>
    <td colspan="8" rowspan="2" align="center">SPEED_<wbr>PROCESSED (×0.01 km/h)</td>
  </tr>
  <tr>
    <th>5</th>
  </tr>
  <tr>
    <th>6</th>
    <td colspan="8" align="center">FUEL_<wbr>ADC</td>
  </tr>
  <tr>
    <th>7</th>
    <td colspan="2" align="center">↑ FUEL_<wbr>ADC</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `DRIVER_UNBUCKLED` | Byte 0 [1] | 1 | 1 | 0 |  | documented |
| `FUEL_LEVEL_NORM` | Byte 1 + Byte 2 [7:6] | 10 | 1 | 0 |  | documented |
| `SPEED_PROCESSED` | Bytes 4–5 | 16 | 0.01 | 0 | km/h | documented |
| `FUEL_ADC` | Byte 6 + Byte 7 [7:6] | 10 | 1 | 0 |  | documented |

<details>
<summary>Signal notes</summary>

- `FUEL_LEVEL_NORM`: normalized 10-bit ADC input from the fuel level sender (independent of supply voltage)
- `FUEL_ADC`: raw 10-bit fuel level from ADC, changes with voltage

</details>

---

## 0x284 Front Wheel Speeds

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x284</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Front Wheel Speeds</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" rowspan="2" align="center">FR_<wbr>WHEEL_<wbr>SPEED (×0.005 km/h)</td>
  </tr>
  <tr>
    <th>1</th>
  </tr>
  <tr>
    <th>2</th>
    <td colspan="8" rowspan="2" align="center">FL_<wbr>WHEEL_<wbr>SPEED (×0.005 km/h)</td>
  </tr>
  <tr>
    <th>3</th>
  </tr>
  <tr>
    <th>4</th>
    <td colspan="8" rowspan="2" align="center">VEHICLE_<wbr>SPEED? (×0.01 km/h)</td>
  </tr>
  <tr>
    <th>5</th>
  </tr>
  <tr>
    <th>6</th>
    <td colspan="8" align="center">COUNTER</td>
  </tr>
  <tr>
    <th>7</th>
    <td colspan="8" align="center">CHECKSUM</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `FR_WHEEL_SPEED` | Bytes 0–1 | 16 | 0.005 | 0 | km/h | confirmed |
| `FL_WHEEL_SPEED` | Bytes 2–3 | 16 | 0.005 | 0 | km/h | documented |
| `VEHICLE_SPEED` | Bytes 4–5 | 16 | 0.01 | 0 | km/h | candidate |
| `COUNTER` | Byte 6 | 8 | 1 | 0 |  | documented |
| `CHECKSUM` | Byte 7 | 8 | 1 | 0 |  | probable |

<details>
<summary>Signal notes</summary>
- `VEHICLE_SPEED`: observed in driving capture: equals 0x354 vehicle speed (km/h x 100)
- `CHECKSUM`: observed: B7 = (sum(B0..B6) + 0x86) & 0xFF

</details>

---

## 0x285 Rear Wheel Speeds

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x285</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Rear Wheel Speeds</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" rowspan="2" align="center">RR_<wbr>WHEEL_<wbr>SPEED (×0.005 km/h)</td>
  </tr>
  <tr>
    <th>1</th>
  </tr>
  <tr>
    <th>2</th>
    <td colspan="8" rowspan="2" align="center">RL_<wbr>WHEEL_<wbr>SPEED (×0.005 km/h)</td>
  </tr>
  <tr>
    <th>3</th>
  </tr>
  <tr>
    <th>4</th>
    <td colspan="8" align="center">VEHICLE_<wbr>SPEED_<wbr>KMH (km/h)</td>
  </tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>6</th>
    <td colspan="8" align="center">COUNTER</td>
  </tr>
  <tr>
    <th>7</th>
    <td colspan="8" align="center">CHECKSUM</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `RR_WHEEL_SPEED` | Bytes 0–1 | 16 | 0.005 | 0 | km/h | documented |
| `RL_WHEEL_SPEED` | Bytes 2–3 | 16 | 0.005 | 0 | km/h | documented |
| `VEHICLE_SPEED_KMH` | Byte 4 | 8 | 1 | 0 | km/h | probable |
| `COUNTER` | Byte 6 | 8 | 1 | 0 |  | documented |
| `CHECKSUM` | Byte 7 | 8 | 1 | 0 |  | probable |

<details>
<summary>Signal notes</summary>

- `VEHICLE_SPEED_KMH`: captures: floor of 0x354 vehicle speed (r 0.9999, 94% exact)
- `CHECKSUM`: observed: B7 = (sum(B0..B6) + 0x87) & 0xFF

</details>

---

## 0x286 Unresolved 0x286

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x286</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Unresolved 0x286</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">2</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" align="center">SPEED_<wbr>DEPENDENT_<wbr>CAND?</td>
  </tr>
  <tr>
    <th>1</th>
    <td colspan="2" align="center">COUNTER</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `SPEED_DEPENDENT_CAND` | Byte 0 | 8 | 1 | 0 |  | candidate |
| `COUNTER` | Byte 1 [7:6] | 2 | 1 | 0 |  | probable |

<details>
<summary>Signal notes</summary>

- `SPEED_DEPENDENT_CAND`: captures: rises with speed (~3 x km/h + 32) and saturates at ~0xED above ~65 km/h

</details>

---

## 0x292 Forces

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x292</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Forces</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">20 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" align="center">LONG_<wbr>ACCEL (×0.001 -2.048 g)</td>
  </tr>
  <tr>
    <th>1</th>
    <td colspan="4" align="center">↑ LONG_<wbr>ACCEL</td>
    <td colspan="4" align="center">LAT_<wbr>ACCEL ↓</td>
  </tr>
  <tr>
    <th>2</th>
    <td colspan="8" align="center">LAT_<wbr>ACCEL (×0.001 -2.048 g)</td>
  </tr>
  <tr>
    <th>3</th>
    <td colspan="8" align="center">YAW_<wbr>RATE? (×0.1 -204.8 deg/s)</td>
  </tr>
  <tr>
    <th>4</th>
    <td colspan="4" align="center">↑ YAW_<wbr>RATE?</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>6</th>
    <td colspan="8" align="center">BRAKE_<wbr>PRESSURE</td>
  </tr>
  <tr>
    <th>7</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td colspan="2" align="center">COUNTER</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `LONG_ACCEL` | Byte 0 + Byte 1 [7:4] | 12 | 0.001 | -2.048 | g | probable |
| `LAT_ACCEL` | Byte 1 [3:0] + Byte 2 | 12 | 0.001 | -2.048 | g | probable |
| `YAW_RATE` | Byte 3 + Byte 4 [7:4] | 12 | 0.1 | -204.8 | deg/s | candidate |
| `BRAKE_PRESSURE` | Byte 6 | 8 | 1 | 0 |  | documented |
| `COUNTER` | Byte 7 [1:0] | 2 | 1 | 0 |  | documented |

<details>
<summary>Signal notes</summary>

- `LONG_ACCEL`: captures: offset binary (0x800 = 0), ~0.001 g/bit (fit vs wheel-speed derivative r 0.93); + = accelerating; 0xFFF before the sensor is ready
- `LAT_ACCEL`: captures: offset binary (0x800 = 0), ~0.001 g/bit (fit vs speed x yaw r 0.92); + = turning left
- `YAW_RATE`: captures: offset binary (0x800 = 0), ~0.1 deg/s/bit, + = left; noisier than 0x2AB YAW_RATE (r 0.74 vs rear-wheel yaw)
- `BRAKE_PRESSURE`: raw, not calibrated

</details>

---

## 0x2AA Unresolved 0x2AA

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x2AA</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Unresolved 0x2AA</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>6</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td colspan="4" align="center">COUNTER</td>
  </tr>
  <tr>
    <th>7</th>
    <td colspan="8" align="center">CHECKSUM</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `COUNTER` | Byte 6 [3:0] | 4 | 1 | 0 |  | probable |
| `CHECKSUM` | Byte 7 | 8 | 1 | 0 |  | probable |

<details>
<summary>Signal notes</summary>

- `CHECKSUM`: captures: B7 = (sum(B0..B6) + 0xAC) & 0xFF (100%)

</details>

---

## 0x2AB Chassis (yaw rate)

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x2AB</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Chassis (yaw rate)</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" align="center">YAW_<wbr>RATE (×-0.0191 deg/s)</td>
  </tr>
  <tr>
    <th>1</th>
    <td colspan="4" align="center">↑ YAW_<wbr>RATE</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <th>2</th>
    <td colspan="8" rowspan="2" align="center">WHEEL_<wbr>ANGLE_<wbr>CAND?</td>
  </tr>
  <tr>
    <th>3</th>
  </tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `YAW_RATE` | Byte 0 + Byte 1 [7:4] | 12 (signed) | -0.0191 | 0 | deg/s | probable |
| `WHEEL_ANGLE_CAND` | Bytes 2–3 | 16 | 1 | 0 |  | candidate |

<details>
<summary>Signal notes</summary>

- `YAW_RATE`: captures: signed 12-bit, r -0.990 vs yaw from rear wheel speeds (track 1.55 m assumed, so factor +-5%); + = turning left
- `WHEEL_ANGLE_CAND`: suggested wheel angle

</details>

---

## 0x2B0 Unresolved 0x2B0

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x2B0</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Unresolved 0x2B0</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>3</th>
    <td align="center">BRAKE_<wbr>SW_<wbr>OFF</td>
    <td align="center">BRAKE_<wbr>SW_<wbr>ON</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>6</th>
    <td colspan="4" align="center">COUNTER</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `BRAKE_SW_ON` | Byte 3 [6] | 1 | 1 | 0 |  | probable |
| `BRAKE_SW_OFF` | Byte 3 [7] | 1 | 1 | 0 |  | probable |
| `COUNTER` | Byte 6 [7:4] | 4 | 1 | 0 |  | probable |

<details>
<summary>Signal notes</summary>

- `BRAKE_SW_ON`: captures: equals 0x182 BRAKE_PRESSED; no visible meter effect
- `BRAKE_SW_OFF`: captures: inverse of BRAKE_SW_ON (two-switch brake input; both 0 or 1 briefly at edges)

</details>

---

## 0x2B1 ADAS

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x2B1</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">ADAS</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">6</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">20 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="2" align="center">CRUISE_<wbr>CONTROL</td>
    <td align="center">CC_<wbr>ENABLED</td>
    <td align="center">CC_<wbr>BIT4?</td>
    <td colspan="2" align="center">DISTANCE_<wbr>BARS</td>
    <td align="center">DISPLAY_<wbr>MPH</td>
    <td align="center">ADAPTIVE_<wbr>CC</td>
  </tr>
  <tr>
    <th>1</th>
    <td colspan="2" align="center">CAR_<wbr>AHEAD</td>
    <td align="center">LANE_<wbr>WARN_<wbr>LEFT</td>
    <td align="center">LANE_<wbr>WARN_<wbr>RIGHT</td>
    <td align="center">BSW_<wbr>LEFT_<wbr>BLINK</td>
    <td align="center">BSW_<wbr>RIGHT_<wbr>BLINK</td>
    <td align="center">EMERGENCY_<wbr>BRAKE_<wbr>WARNING</td>
    <td align="center">WINDING_<wbr>ROAD</td>
  </tr>
  <tr>
    <th>2</th>
    <td colspan="2" align="center">EMG_<wbr>BRAKE</td>
    <td colspan="2" align="center">LANE_<wbr>ASSIST</td>
    <td colspan="2" align="center">BLIND_<wbr>SPOT</td>
    <td align="center">FEB_<wbr>LIGHT_<wbr>BLINK</td>
    <td align="center">FEB_<wbr>LIGHT</td>
  </tr>
  <tr>
    <th>3</th>
    <td colspan="2" align="center">EMG_<wbr>BRAKE_<wbr>2</td>
    <td colspan="2" align="center">LANE_<wbr>ASSIST_<wbr>2</td>
    <td colspan="2" align="center">BLIND_<wbr>SPOT_<wbr>2</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <th>4</th>
    <td colspan="8" align="center">CC_<wbr>SPEED (-1)</td>
  </tr>
  <tr>
    <th>5</th>
    <td colspan="3" align="center">TONES</td>
    <td align="center">SWITCH_<wbr>PAGE</td>
    <td align="center">BLINK_<wbr>SPEED</td>
    <td align="center">PROMPT_<wbr>PAGE</td>
    <td></td>
    <td></td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `CRUISE_CONTROL` | Byte 0 [7:6] | 2 | 1 | 0 |  | confirmed |
| `CC_ENABLED` | Byte 0 [5] | 1 | 1 | 0 |  | documented |
| `CC_BIT4` | Byte 0 [4] | 1 | 1 | 0 |  | candidate |
| `DISTANCE_BARS` | Byte 0 [3:2] | 2 | 1 | 0 |  | documented |
| `DISPLAY_MPH` | Byte 0 [1] | 1 | 1 | 0 |  | documented |
| `ADAPTIVE_CC` | Byte 0 [0] | 1 | 1 | 0 |  | documented |
| `CAR_AHEAD` | Byte 1 [7:6] | 2 | 1 | 0 |  | confirmed |
| `LANE_WARN_LEFT` | Byte 1 [5] | 1 | 1 | 0 |  | documented |
| `LANE_WARN_RIGHT` | Byte 1 [4] | 1 | 1 | 0 |  | documented |
| `BSW_LEFT_BLINK` | Byte 1 [3] | 1 | 1 | 0 |  | documented |
| `BSW_RIGHT_BLINK` | Byte 1 [2] | 1 | 1 | 0 |  | documented |
| `EMERGENCY_BRAKE_WARNING` | Byte 1 [1] | 1 | 1 | 0 |  | confirmed |
| `WINDING_ROAD` | Byte 1 [0] | 1 | 1 | 0 |  | confirmed |
| `EMG_BRAKE` | Byte 2 [7:6] | 2 | 1 | 0 |  | confirmed |
| `LANE_ASSIST` | Byte 2 [5:4] | 2 | 1 | 0 |  | confirmed |
| `BLIND_SPOT` | Byte 2 [3:2] | 2 | 1 | 0 |  | confirmed |
| `FEB_LIGHT_BLINK` | Byte 2 [1] | 1 | 1 | 0 |  | documented |
| `FEB_LIGHT` | Byte 2 [0] | 1 | 1 | 0 |  | documented |
| `EMG_BRAKE_2` | Byte 3 [7:6] | 2 | 1 | 0 |  | confirmed |
| `LANE_ASSIST_2` | Byte 3 [5:4] | 2 | 1 | 0 |  | confirmed |
| `BLIND_SPOT_2` | Byte 3 [3:2] | 2 | 1 | 0 |  | documented |
| `CC_SPEED` | Byte 4 | 8 | 1 | -1 |  | documented |
| `TONES` | Byte 5 [7:5] | 3 | 1 | 0 |  | documented |
| `SWITCH_PAGE` | Byte 5 [4] | 1 | 1 | 0 |  | documented |
| `BLINK_SPEED` | Byte 5 [3] | 1 | 1 | 0 |  | documented |
| `PROMPT_PAGE` | Byte 5 [2] | 1 | 1 | 0 |  | confirmed |

<details>
<summary>Signal notes</summary>

- `CRUISE_CONTROL`: bench 2026-10-12: 3 turns the cruise icon orange
- `CC_BIT4`: appears in older enum tables; do not treat as a confirmed icon bit; bench 2026-10-12: no visible change
- `CAR_AHEAD`: bench 2026-10-12: 2 orange car bar; 3 red crash-alert banner
- `LANE_WARN_LEFT`: bench 2026-10-12: lane icon blinks orange (side not distinguishable on camera)
- `LANE_WARN_RIGHT`: bench 2026-10-12: lane icon blinks orange (side not distinguishable on camera)
- `BSW_RIGHT_BLINK`: bench 2026-10-12: orange marks blink at the rear of the car icon
- `EMERGENCY_BRAKE_WARNING`: bench 2026-10-12: orange forward zone on Driving Aids page, blinking arc in top bar
- `WINDING_ROAD`: bench 2026-10-12: shown only while CAR_AHEAD is 1 (white) or 2 (yellow) (owner, bench)
- `EMG_BRAKE`: bench 2026-10-12: 1 white, 2 warning triangle on Driving Aids page, 3 orange, 0 hidden
- `LANE_ASSIST`: bench 2026-10-12: 1 white, 2 warning triangle on Driving Aids page, 3 orange, 0 hidden
- `BLIND_SPOT`: bench 2026-10-12: 2 warning triangle on Driving Aids page, 3 orange rear zones
- `FEB_LIGHT`: bench 2026-10-12: amber FEB telltale: two cars with '!' between (owner photo)
- `EMG_BRAKE_2`: bench 2026-10-12: 1/2 green, 3 orange, 0 hidden
- `LANE_ASSIST_2`: bench 2026-10-12: 1/2 green, 3 orange, 0 hidden
- `BLIND_SPOT_2`: bench 2026-10-12: shown in the top bar
- `CC_SPEED`: 0x00 blank, 0x01-0xFE = speed + 1 (shown as raw-1), 0xFF '- - -'
- `SWITCH_PAGE`: sometimes switches page without beep; retrigger rules unresolved
- `PROMPT_PAGE`: prompts ADAS page + beep; bench 2026-10-12: shows the Driving Aids page, which reverts after a few seconds

</details>

### 0x2B1 · CRUISE_CONTROL

| Value | Meaning |
|-------|---------|
| 0b00 (0) | None |
| 0b01 (1) | CC icon |
| 0b10 (2) | CC blink |
| 0b11 (3) | Yellow CC (warning) |

### 0x2B1 · DISTANCE_BARS

| Value | Meaning |
|-------|---------|
| 0b00 (0) | No bars |
| 0b01 (1) | 1 bar |
| 0b10 (2) | 2 bars |
| 0b11 (3) | 3 bars |

### 0x2B1 · CAR_AHEAD

| Value | Meaning |
|-------|---------|
| 0b00 (0) | None |
| 0b01 (1) | White car ahead |
| 0b10 (2) | Yellow blinking car |
| 0b11 (3) | Crash alert |

### 0x2B1 · EMG_BRAKE

| Value | Meaning |
|-------|---------|
| 0b00 (0) | Disabled |
| 0b01 (1) | Enabled |
| 0b10 (2) | White blink (warning icon) |
| 0b11 (3) | Yellow solid (warning icon) |

### 0x2B1 · LANE_ASSIST

| Value | Meaning |
|-------|---------|
| 0b00 (0) | Disabled |
| 0b01 (1) | Enabled |
| 0b10 (2) | White blink (warning icon) |
| 0b11 (3) | Yellow solid (warning icon) |

### 0x2B1 · BLIND_SPOT

| Value | Meaning |
|-------|---------|
| 0b00 (0) | Disabled |
| 0b01 (1) | Enabled |
| 0b10 (2) | White blink (warning icon) |
| 0b11 (3) | Yellow solid (warning icon) |

### 0x2B1 · EMG_BRAKE_2

| Value | Meaning |
|-------|---------|
| 0b00 (0) | Disabled |
| 0b01 (1) | Green enabled |
| 0b10 (2) | Green blink (warning icon) |
| 0b11 (3) | Yellow solid (warning icon) |

### 0x2B1 · LANE_ASSIST_2

| Value | Meaning |
|-------|---------|
| 0b00 (0) | Disabled |
| 0b01 (1) | Green enabled |
| 0b10 (2) | Green blink (warning icon) |
| 0b11 (3) | Yellow solid (warning icon) |

### 0x2B1 · BLIND_SPOT_2

| Value | Meaning |
|-------|---------|
| 0b00 (0) | Disabled |
| 0b01 (1) | Green enabled |
| 0b10 (2) | Green blink (warning icon) |
| 0b11 (3) | Yellow solid (warning icon) |

### 0x2B1 · CC_SPEED

| Value | Meaning |
|-------|---------|
| 0x00 | Blank |
| 0x01–0xFE | Speed + 1 (shown as raw − 1) |
| 0xFF | - - - |

### 0x2B1 · TONES

| Value | Meaning |
|-------|---------|
| 0b000 (0) | None |
| 0b001 (1) | Constant beep |
| 0b010 (2) | Quick beep |
| 0b011 (3) | Triple fast beep |
| 0b100 (4) | Fast single beep |
| 0b101 (5) | Quad beep |
| 0b110 (6) | Single beep |
| 0b111 (7) | Double beep |

---

## 0x2DE Cluster Status 2 (meter)

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x2DE</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Cluster Status 2 (meter)</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Meter (cluster)</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td></td>
    <td></td>
    <td align="center">PADDLE_<wbr>DOWN_<wbr>REQ</td>
    <td align="center">PADDLE_<wbr>UP_<wbr>REQ</td>
    <td align="center">STICK_<wbr>DOWN_<wbr>REQ</td>
    <td align="center">STICK_<wbr>UP_<wbr>REQ</td>
    <td align="center">AUTO_<wbr>MODE?</td>
    <td align="center">MANUAL_<wbr>MODE</td>
  </tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>3</th>
    <td colspan="8" align="center">AVG_<wbr>FUEL_<wbr>ECONOMY (×0.01 km/L)</td>
  </tr>
  <tr>
    <th>4</th>
    <td colspan="4" align="center">↑ AVG_<wbr>FUEL_<wbr>ECONOMY</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>6</th>
    <td colspan="8" rowspan="2" align="center">DISTANCE_<wbr>TO_<wbr>EMPTY (×0.1 km)</td>
  </tr>
  <tr>
    <th>7</th>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `PADDLE_DOWN_REQ` | Byte 0 [5] | 1 | 1 | 0 |  | documented |
| `PADDLE_UP_REQ` | Byte 0 [4] | 1 | 1 | 0 |  | documented |
| `STICK_DOWN_REQ` | Byte 0 [3] | 1 | 1 | 0 |  | documented |
| `STICK_UP_REQ` | Byte 0 [2] | 1 | 1 | 0 |  | documented |
| `AUTO_MODE` | Byte 0 [1] | 1 | 1 | 0 |  | candidate |
| `MANUAL_MODE` | Byte 0 [0] | 1 | 1 | 0 |  | documented |
| `AVG_FUEL_ECONOMY` | Byte 3 + Byte 4 [7:4] | 12 | 0.01 | 0 | km/L | documented |
| `DISTANCE_TO_EMPTY` | Bytes 6–7 | 16 | 0.1 | 0 | km | documented |

<details>
<summary>Signal notes</summary>

- `AVG_FUEL_ECONOMY`: x 2.3521458 = US mpg; low nibble of B4 not part of it

</details>

---

## 0x300 Power Steering Assist

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x300</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Power Steering Assist</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">1</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" align="center">STEER_<wbr>EFFORT_<wbr>CAND</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `STEER_EFFORT_CAND` | Byte 0 | 8 | 1 | 0 |  | probable |

<details>
<summary>Signal notes</summary>

- `STEER_EFFORT_CAND`: captures 2026-10-12: unsigned power-steering assist magnitude, uncalibrated. Rises while the wheel turns (~120 at full lock parked), back near 0 when it stops, same sign left and right; falls with speed (mean 46 parked, 16 at 2-10 km/h, 7 at 10-40, ~3 above 40 km/h while steering). 0-1 with the engine off, ~2-3 at idle. Bit 1 was formerly ENGINE_RUNNING: it only looked like a flag in idle captures (while driving 0 in 28940 samples, 1 in 17547).

</details>

---

## 0x351 Intelligent Key

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x351</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Intelligent Key</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>1</th>
    <td align="center">UNLOCK</td>
    <td align="center">LOCK</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>5</th>
    <td align="center">STEERING_<wbr>LOCK_<wbr>REG_<wbr>COMPLETE</td>
    <td align="center">KEY_<wbr>SYSTEM_<wbr>ERROR</td>
    <td align="center">DOUBLE_<wbr>BEEP</td>
    <td align="center">FAST_<wbr>BEEP</td>
    <td colspan="4" align="center">KEY_<wbr>PROMPT</td>
  </tr>
  <tr>
    <th>6</th>
    <td align="center">BEEP</td>
    <td></td>
    <td></td>
    <td align="center">AUTO_<wbr>HIGH_<wbr>BEAM</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <th>7</th>
    <td colspan="8" align="center">CHECKSUM?</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `UNLOCK` | Byte 1 [7] | 1 | 1 | 0 |  | documented |
| `LOCK` | Byte 1 [6] | 1 | 1 | 0 |  | documented |
| `STEERING_LOCK_REG_COMPLETE` | Byte 5 [7] | 1 | 1 | 0 |  | confirmed |
| `KEY_SYSTEM_ERROR` | Byte 5 [6] | 1 | 1 | 0 |  | confirmed |
| `DOUBLE_BEEP` | Byte 5 [5] | 1 | 1 | 0 |  | documented |
| `FAST_BEEP` | Byte 5 [4] | 1 | 1 | 0 |  | documented |
| `KEY_PROMPT` | Byte 5 [3:0] | 4 | 1 | 0 |  | confirmed |
| `BEEP` | Byte 6 [7] | 1 | 1 | 0 |  | documented |
| `AUTO_HIGH_BEAM` | Byte 6 [4] | 1 | 1 | 0 |  | confirmed |
| `CHECKSUM` | Byte 7 | 8 | 1 | 0 |  | candidate |

<details>
<summary>Signal notes</summary>
- `STEERING_LOCK_REG_COMPLETE`: bench 2026-10-12: 'Steering Lock Registration Complete'
- `KEY_SYSTEM_ERROR`: bench 2026-10-12: 'Key System Error / See Owner's Manual'
- `KEY_PROMPT`: high bits of byte 5 show priority/suppression when combined; bench 2026-10-12: all 15 codes shown as listed; 2/12/13 show nothing; 15 also turns the gauge backlight off
- `AUTO_HIGH_BEAM`: bench 2026-10-12: green auto-high-beam telltale
- `CHECKSUM`: observed: B7 = sum(B0..B6) & 0xFF in idle capture

</details>

### 0x351 · KEY_PROMPT

| Value | Meaning |
|-------|---------|
| 0b0000 (0) | None |
| 0b0001 (1) | Brake + Push Start |
| 0b0010 (2) | No visible response (0x2) |
| 0b0011 (3) | Key ID Incorrect |
| 0b0100 (4) | Turn Steering Wheel + Push Button |
| 0b0101 (5) | Shift to Park |
| 0b0110 (6) | Power turned off to save battery |
| 0b0111 (7) | Key Battery Low |
| 0b1000 (8) | No key detected |
| 0b1001 (9) | Power will turn off to save battery |
| 0b1010 (10) | Push Ignition to OFF |
| 0b1011 (11) | Clutch + Push Start (shared firmware) |
| 0b1100 (12) | No visible response (0xC) |
| 0b1101 (13) | No visible response (0xD) |
| 0b1110 (14) | Touch Key to Start Button |
| 0b1111 (15) | Push Brake and Start Button to Drive |

---

## 0x354 Chassis Control (ABS)

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x354</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Chassis Control (ABS)</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">50 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" rowspan="2" align="center">VEHICLE_<wbr>SPEED (×0.01 km/h)</td>
  </tr>
  <tr>
    <th>1</th>
  </tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>4</th>
    <td colspan="3" align="center">TCS_<wbr>STATE</td>
    <td align="center">ABS</td>
    <td align="center">ABS_<wbr>LATCH</td>
    <td align="center">BRAKE</td>
    <td align="center">BRAKE_<wbr>LATCH</td>
    <td></td>
  </tr>
  <tr>
    <th>5</th>
    <td></td>
    <td></td>
    <td></td>
    <td colspan="2" align="center">COUNTER</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <th>6</th>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">BRAKE_<wbr>PRESSED</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `VEHICLE_SPEED` | Bytes 0–1 | 16 | 0.01 | 0 | km/h | documented |
| `TCS_STATE` | Byte 4 [7:5] | 3 | 1 | 0 |  | confirmed |
| `ABS` | Byte 4 [4] | 1 | 1 | 0 |  | confirmed |
| `ABS_LATCH` | Byte 4 [3] | 1 | 1 | 0 |  | confirmed |
| `BRAKE` | Byte 4 [2] | 1 | 1 | 0 |  | confirmed |
| `BRAKE_LATCH` | Byte 4 [1] | 1 | 1 | 0 |  | confirmed |
| `COUNTER` | Byte 5 [4:3] | 2 | 1 | 0 |  | documented |
| `BRAKE_PRESSED` | Byte 6 [4] | 1 | 1 | 0 |  | documented |

<details>
<summary>Signal notes</summary>

- `TCS_STATE`: bench 2026-10-12: 1, 2, 4-7 as listed (4: slip blinks; 6/7: TCS OFF solid + slip blinks); 3 showed nothing
- `ABS`: freezes the ABS lamp in its current state while set; never lights it by itself
- `ABS_LATCH`: lights ABS while set (only when bit 4 is clear; with bit 4 set the lamp keeps its state)
- `BRAKE`: freezes the BRAKE lamp in its current state while set; never lights it by itself
- `BRAKE_LATCH`: lights BRAKE while set (only when bit 2 is clear). BRAKE is shared with parking brake and low brake fluid inputs
- `COUNTER`: 2-bit counter (00,08,10,18)

</details>

### 0x354 · TCS_STATE

| Value | Meaning |
|-------|---------|
| 0b000 (0) | None |
| 0b001 (1) | TCS OFF solid + slip solid |
| 0b010 (2) | TCS OFF solid |
| 0b011 (3) | Slip blinking (not seen on bench) |
| 0b100 (4) | Slip blinking |
| 0b101 (5) | Slip solid |
| 0b110 (6) | TCS OFF + slip blinking |
| 0b111 (7) | TCS OFF + slip blinking |

---

## 0x355 Cluster Status 3 (meter)

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x355</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Cluster Status 3 (meter)</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Meter (cluster)</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" rowspan="2" align="center">SPEED_<wbr>PROCESSED (×0.01 km/h)</td>
  </tr>
  <tr>
    <th>1</th>
  </tr>
  <tr>
    <th>2</th>
    <td colspan="8" rowspan="2" align="center">SPEED_<wbr>DISPLAYED (×0.01 km/h)</td>
  </tr>
  <tr>
    <th>3</th>
  </tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `SPEED_PROCESSED` | Bytes 0–1 | 16 | 0.01 | 0 | km/h | documented |
| `SPEED_DISPLAYED` | Bytes 2–3 | 16 | 0.01 | 0 | km/h | documented |

---

## 0x358 Body / Lamps

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x358</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Body / Lamps</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>1</th>
    <td align="center">DIM_<wbr>BACKLIGHT</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">BULB_<wbr>CHECK_<wbr>CAND?</td>
  </tr>
  <tr>
    <th>2</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">RIGHT_<wbr>BLINKER_<wbr>REQ?</td>
    <td align="center">LEFT_<wbr>BLINKER_<wbr>REQ?</td>
    <td align="center">TRUNK_<wbr>OPEN</td>
  </tr>
  <tr>
    <th>3</th>
    <td align="center">UNLOCK_<wbr>EVENT_<wbr>CAND?</td>
    <td></td>
    <td align="center">LOW_<wbr>OIL_<wbr>PRESSURE</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <th>4</th>
    <td align="center">TPMS_<wbr>LIGHT</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <th>5</th>
    <td colspan="3" align="center">REMOTE_<wbr>START_<wbr>STATE?</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `DIM_BACKLIGHT` | Byte 1 [7] | 1 | 1 | 0 |  | confirmed |
| `RIGHT_BLINKER_REQ` | Byte 2 [2] | 1 | 1 | 0 |  | candidate |
| `LEFT_BLINKER_REQ` | Byte 2 [1] | 1 | 1 | 0 |  | candidate |
| `TRUNK_OPEN` | Byte 2 [0] | 1 | 1 | 0 |  | confirmed |
| `LOW_OIL_PRESSURE` | Byte 3 [5] | 1 | 1 | 0 |  | confirmed |
| `TPMS_LIGHT` | Byte 4 [7] | 1 | 1 | 0 |  | documented |
| `BULB_CHECK_CAND` | Byte 1 [0] | 1 | 1 | 0 |  | candidate |
| `UNLOCK_EVENT_CAND` | Byte 3 [7] | 1 | 1 | 0 |  | candidate |
| `REMOTE_START_STATE` | Byte 5 [7:5] | 3 | 1 | 0 |  | candidate |

<details>
<summary>Signal notes</summary>

- `DIM_BACKLIGHT`: bench 2026-10-12: whole meter dims
- `RIGHT_BLINKER_REQ`: no visible effect on the bench; bench 2026-10-12: no visible change
- `LEFT_BLINKER_REQ`: no visible effect on the bench; bench 2026-10-12: no visible change
- `TRUNK_OPEN`: bench 2026-10-12: Warning + trunk shown open
- `LOW_OIL_PRESSURE`: lights red oil lamp + 'Low Oil Pressure / Stop Vehicle' (bench 2026-10-12, 3000 rpm); off at RPM 0
- `BULB_CHECK_CAND`: captures: 1 s pulse at IG on together with TPMS_LIGHT; no visible meter effect (bench)
- `UNLOCK_EVENT_CAND`: captures: 0.3 s pulse at each unlock (with 0x351 UNLOCK and the hazard flash); no visible effect
- `REMOTE_START_STATE`: captures: 0 in normal use; 1-4 only in the two remote-start captures (4 -> 3 at remote start, 0 when the driver takes over); no visible meter effect

</details>

---

## 0x35D Brake Signal

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x35D</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Brake Signal</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td align="center">IG_<wbr>ON</td>
    <td align="center">CRANKING</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>4</th>
    <td></td>
    <td align="center">UNKNOWN_<wbr>B4_<wbr>6?</td>
    <td></td>
    <td align="center">BRAKE_<wbr>PRESSED</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>7</th>
    <td colspan="4" align="center">REMOTE_<wbr>START_<wbr>CAND?</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `IG_ON` | Byte 0 [7] | 1 | 1 | 0 |  | probable |
| `CRANKING` | Byte 0 [6] | 1 | 1 | 0 |  | probable |
| `UNKNOWN_B4_6` | Byte 4 [6] | 1 | 1 | 0 |  | candidate |
| `BRAKE_PRESSED` | Byte 4 [4] | 1 | 1 | 0 |  | documented |
| `REMOTE_START_CAND` | Byte 7 [7:4] | 4 | 1 | 0 |  | candidate |

<details>
<summary>Signal notes</summary>

- `IG_ON`: captures: equals 0x60D IG_RUN (2 mismatches in ~55k samples)
- `CRANKING`: captures: set exactly while 0x625 START_STATE = 3 (cranking); no visible meter effect
- `UNKNOWN_B4_6`: bench 2026-10-12: no visible change
- `REMOTE_START_CAND`: captures: 5 normally, 0xA while the engine runs on remote start

</details>

---

## 0x385 TPMS

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x385</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">TPMS</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">PRESSURES_<wbr>WHILE_<wbr>DRIVING</td>
    <td></td>
  </tr>
  <tr>
    <th>1</th>
    <td align="center">FR_<wbr>FLAT</td>
    <td align="center">FL_<wbr>FLAT</td>
    <td align="center">RR_<wbr>FLAT</td>
    <td align="center">RL_<wbr>FLAT</td>
    <td align="center">FR_<wbr>WARNING</td>
    <td align="center">FL_<wbr>WARNING</td>
    <td align="center">RR_<wbr>WARNING</td>
    <td align="center">RL_<wbr>WARNING</td>
  </tr>
  <tr>
    <th>2</th>
    <td colspan="8" align="center">FR_<wbr>PRESSURE (×0.25 psi)</td>
  </tr>
  <tr>
    <th>3</th>
    <td colspan="8" align="center">FL_<wbr>PRESSURE (×0.25 psi)</td>
  </tr>
  <tr>
    <th>4</th>
    <td colspan="8" align="center">RR_<wbr>PRESSURE (×0.25 psi)</td>
  </tr>
  <tr>
    <th>5</th>
    <td colspan="8" align="center">RL_<wbr>PRESSURE (×0.25 psi)</td>
  </tr>
  <tr>
    <th>6</th>
    <td align="center">FR_<wbr>VALID</td>
    <td align="center">FL_<wbr>VALID</td>
    <td align="center">RR_<wbr>VALID</td>
    <td align="center">RL_<wbr>VALID</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <th>7</th>
    <td colspan="4" align="center">TARGET_<wbr>FRONT (×10 +190 kPa)</td>
    <td colspan="4" align="center">TARGET_<wbr>REAR (×10 +190 kPa)</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `PRESSURES_WHILE_DRIVING` | Byte 0 [1] | 1 | 1 | 0 |  | confirmed |
| `FR_FLAT` | Byte 1 [7] | 1 | 1 | 0 |  | documented |
| `FR_WARNING` | Byte 1 [3] | 1 | 1 | 0 |  | documented |
| `FR_PRESSURE` | Byte 2 | 8 | 0.25 | 0 | psi | documented |
| `FR_VALID` | Byte 6 [7] | 1 | 1 | 0 |  | documented |
| `FL_FLAT` | Byte 1 [6] | 1 | 1 | 0 |  | documented |
| `FL_WARNING` | Byte 1 [2] | 1 | 1 | 0 |  | documented |
| `FL_PRESSURE` | Byte 3 | 8 | 0.25 | 0 | psi | documented |
| `FL_VALID` | Byte 6 [6] | 1 | 1 | 0 |  | documented |
| `RR_FLAT` | Byte 1 [5] | 1 | 1 | 0 |  | documented |
| `RR_WARNING` | Byte 1 [1] | 1 | 1 | 0 |  | documented |
| `RR_PRESSURE` | Byte 4 | 8 | 0.25 | 0 | psi | documented |
| `RR_VALID` | Byte 6 [5] | 1 | 1 | 0 |  | documented |
| `RL_FLAT` | Byte 1 [4] | 1 | 1 | 0 |  | documented |
| `RL_WARNING` | Byte 1 [0] | 1 | 1 | 0 |  | documented |
| `RL_PRESSURE` | Byte 5 | 8 | 0.25 | 0 | psi | documented |
| `RL_VALID` | Byte 6 [4] | 1 | 1 | 0 |  | documented |
| `TARGET_FRONT` | Byte 7 [7:4] | 4 | 10 | 190 | kPa | confirmed |
| `TARGET_REAR` | Byte 7 [3:0] | 4 | 10 | 190 | kPa | confirmed |

<details>
<summary>Signal notes</summary>

- `PRESSURES_WHILE_DRIVING`: with every tire invalid: 0 shows 'Tire pressures appear while driving.', 1 shows dashes
- `TARGET_FRONT`: kPa = 190 + 10 x nibble; nibble 0 hides the target (bench: 1 -> 29 psi, 15 -> 49 psi)
- `TARGET_REAR`: kPa = 190 + 10 x nibble; nibble 0 hides the target (bench: 10 -> 42 psi)

</details>

---

## 0x3AF Hazard Switch

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x3AF</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Hazard Switch</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">1</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td align="center">HAZARD_<wbr>BUTTON</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `HAZARD_BUTTON` | Byte 0 [7] | 1 | 1 | 0 |  | documented |

---

## 0x3EC Engine RPM (meter tach source)

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x3EC</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Engine RPM (meter tach source)</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">2</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" rowspan="2" align="center">ENGINE_<wbr>RPM (×0.125 rpm)</td>
  </tr>
  <tr>
    <th>1</th>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `ENGINE_RPM` | Bytes 0–1 | 16 | 0.125 | 0 | rpm | confirmed |

<details>
<summary>Signal notes</summary>

- `ENGINE_RPM`: raw = rpm * 8. Bench 2026-10-12: while 0x3EC is sent the tach follows it, not 0x180; removing it leaves the tach at 0 until a B+ cycle. Linked to 0x180.

</details>

---

## 0x421 Automatic Transmission

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x421</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Automatic Transmission</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">3</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">60 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="5" align="center">GEAR</td>
    <td align="center">AT_<wbr>ERROR_<wbr>A</td>
    <td></td>
    <td align="center">AT_<wbr>ERROR_<wbr>B</td>
  </tr>
  <tr>
    <th>1</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">FLASH_<wbr>GEAR</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <th>2</th>
    <td></td>
    <td align="center">BEEP</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `GEAR` | Byte 0 [7:3] | 5 | 1 | 0 |  | confirmed |
| `AT_ERROR_A` | Byte 0 [2] | 1 | 1 | 0 |  | confirmed |
| `AT_ERROR_B` | Byte 0 [0] | 1 | 1 | 0 |  | confirmed |
| `FLASH_GEAR` | Byte 1 [2] | 1 | 1 | 0 |  | confirmed |
| `BEEP` | Byte 2 [6] | 1 | 1 | 0 |  | documented |

<details>
<summary>Signal notes</summary>

- `GEAR`: bench 2026-10-12: every listed code shown (N boxed, Ds, L, 1-6, M1-M8 as nM)
- `AT_ERROR_A`: bench 2026-10-12: 'AT Error' like AT_ERROR_B, but only while RPM > 0 (owner)
- `AT_ERROR_B`: bench 2026-10-12: 'AT Error / See Owner's Manual' + master warning; needs RPM > 0 (owner)
- `FLASH_GEAR`: bench 2026-10-12: gear digit blinks

</details>

### 0x421 · GEAR

| Value | Meaning |
|-------|---------|
| 0b00000 (0) | None |
| 0b00001 (1) | P |
| 0b00010 (2) | R |
| 0b00011 (3) | N |
| 0b00100 (4) | D |
| 0b00101 (5) | D (S) |
| 0b00110 (6) | L |
| 0b01000 (8) | 1 |
| 0b01001 (9) | 2 |
| 0b01010 (10) | 3 |
| 0b01011 (11) | 4 |
| 0b01100 (12) | 5 |
| 0b01101 (13) | 6 |
| 0b10000 (16) | M1 |
| 0b10001 (17) | M2 |
| 0b10010 (18) | M3 |
| 0b10011 (19) | M4 |
| 0b10100 (20) | M5 |
| 0b10101 (21) | M6 |
| 0b10110 (22) | M7 |
| 0b10111 (23) | M8 |

---

## 0x4CC ADAS Errors

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x4CC</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">ADAS Errors</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">5</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">500 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">UNAVAIL_<wbr>HIGH_<wbr>ACCEL_<wbr>TEMP_<wbr>DCA</td>
    <td align="center">UNAVAIL_<wbr>HIGH_<wbr>ACCEL_<wbr>TEMP_<wbr>BCI</td>
    <td align="center">UNAVAIL_<wbr>HIGH_<wbr>ACCEL_<wbr>TEMP_<wbr>FEB</td>
  </tr>
  <tr>
    <th>1</th>
    <td align="center">UNAVAIL_<wbr>CLEAN_<wbr>REAR_<wbr>CAM_<wbr>LDW</td>
    <td align="center">UNAVAIL_<wbr>CLEAN_<wbr>REAR_<wbr>CAM_<wbr>BSW</td>
    <td align="center">MALFUNCTION_<wbr>PFCW</td>
    <td align="center">MALFUNCTION_<wbr>LDW</td>
    <td align="center">MALFUNCTION_<wbr>BSW</td>
    <td align="center">UNAVAIL_<wbr>HIGH_<wbr>CABIN_<wbr>TEMP_<wbr>LDW</td>
    <td align="center">UNAVAIL_<wbr>HIGH_<wbr>CABIN_<wbr>TEMP_<wbr>LDP</td>
    <td align="center">UNAVAIL_<wbr>HIGH_<wbr>CABIN_<wbr>TEMP_<wbr>BSI</td>
  </tr>
  <tr>
    <th>2</th>
    <td align="center">UNAVAIL_<wbr>FRONT_<wbr>RADAR_<wbr>ICC</td>
    <td align="center">UNAVAIL_<wbr>FRONT_<wbr>RADAR_<wbr>PFCW</td>
    <td align="center">UNAVAIL_<wbr>FRONT_<wbr>RADAR_<wbr>DCA</td>
    <td align="center">UNAVAIL_<wbr>FRONT_<wbr>RADAR_<wbr>FEB</td>
    <td align="center">UNAVAIL_<wbr>SELECT_<wbr>DRIVING_<wbr>AIDS_<wbr>A</td>
    <td align="center">UNAVAIL_<wbr>SELECT_<wbr>DRIVING_<wbr>AIDS_<wbr>B</td>
    <td align="center">CURRENTLY_<wbr>UNAVAIL_<wbr>ICC_<wbr>A</td>
    <td align="center">CURRENTLY_<wbr>UNAVAIL_<wbr>ICC_<wbr>B</td>
  </tr>
  <tr>
    <th>3</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">UNAVAIL_<wbr>SIDE_<wbr>RADAR_<wbr>BSW</td>
    <td align="center">UNAVAIL_<wbr>SIDE_<wbr>RADAR_<wbr>BSI</td>
    <td align="center">UNAVAIL_<wbr>SIDE_<wbr>RADAR_<wbr>BCI</td>
    <td></td>
  </tr>
  <tr>
    <th>4</th>
    <td align="center">MALFUNCTION_<wbr>DCA</td>
    <td align="center">MALFUNCTION_<wbr>LDP</td>
    <td align="center">MALFUNCTION_<wbr>BSI</td>
    <td align="center">MALFUNCTION_<wbr>FEB</td>
    <td align="center">MALFUNCTION_<wbr>BCI</td>
    <td align="center">SYSTEM_<wbr>OFF_<wbr>BCI</td>
    <td></td>
    <td></td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `UNAVAIL_HIGH_ACCEL_TEMP_DCA` | Byte 0 [2] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_HIGH_ACCEL_TEMP_BCI` | Byte 0 [1] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_HIGH_ACCEL_TEMP_FEB` | Byte 0 [0] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_CLEAN_REAR_CAM_LDW` | Byte 1 [7] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_CLEAN_REAR_CAM_BSW` | Byte 1 [6] | 1 | 1 | 0 |  | documented |
| `MALFUNCTION_PFCW` | Byte 1 [5] | 1 | 1 | 0 |  | documented |
| `MALFUNCTION_LDW` | Byte 1 [4] | 1 | 1 | 0 |  | documented |
| `MALFUNCTION_BSW` | Byte 1 [3] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_HIGH_CABIN_TEMP_LDW` | Byte 1 [2] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_HIGH_CABIN_TEMP_LDP` | Byte 1 [1] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_HIGH_CABIN_TEMP_BSI` | Byte 1 [0] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_FRONT_RADAR_ICC` | Byte 2 [7] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_FRONT_RADAR_PFCW` | Byte 2 [6] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_FRONT_RADAR_DCA` | Byte 2 [5] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_FRONT_RADAR_FEB` | Byte 2 [4] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_SELECT_DRIVING_AIDS_A` | Byte 2 [3] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_SELECT_DRIVING_AIDS_B` | Byte 2 [2] | 1 | 1 | 0 |  | documented |
| `CURRENTLY_UNAVAIL_ICC_A` | Byte 2 [1] | 1 | 1 | 0 |  | documented |
| `CURRENTLY_UNAVAIL_ICC_B` | Byte 2 [0] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_SIDE_RADAR_BSW` | Byte 3 [3] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_SIDE_RADAR_BSI` | Byte 3 [2] | 1 | 1 | 0 |  | documented |
| `UNAVAIL_SIDE_RADAR_BCI` | Byte 3 [1] | 1 | 1 | 0 |  | documented |
| `MALFUNCTION_DCA` | Byte 4 [7] | 1 | 1 | 0 |  | documented |
| `MALFUNCTION_LDP` | Byte 4 [6] | 1 | 1 | 0 |  | documented |
| `MALFUNCTION_BSI` | Byte 4 [5] | 1 | 1 | 0 |  | documented |
| `MALFUNCTION_FEB` | Byte 4 [4] | 1 | 1 | 0 |  | documented |
| `MALFUNCTION_BCI` | Byte 4 [3] | 1 | 1 | 0 |  | documented |
| `SYSTEM_OFF_BCI` | Byte 4 [2] | 1 | 1 | 0 |  | documented |

---

## 0x4DF

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x4DF</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7"></td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

---

## 0x4F6 Page Slot Prerequisite B

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x4F6</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Page Slot Prerequisite B</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

---

## 0x519 Drive Mode Switch / Startup (raw)

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x519</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Drive Mode Switch / Startup (raw)</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" align="center">STATE?</td>
  </tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `STATE` | Byte 0 | 8 | 1 | 0 |  | candidate |

<details>
<summary>Signal notes</summary>

- `STATE`: startup E0 -> 80 -> 00; switch sequences 14/06/04/16; bench 2026-10-12: no visible change (0xFF)

</details>

---

## 0x51B Service Distance (wrong ID?)

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x51B</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Service Distance (wrong ID?)</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">6</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>4</th>
    <td></td>
    <td></td>
    <td colspan="6" align="center">SERVICE_<wbr>DISTANCE? ↓</td>
  </tr>
  <tr>
    <th>5</th>
    <td colspan="8" align="center">SERVICE_<wbr>DISTANCE? (×5 km)</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `SERVICE_DISTANCE` | Byte 4 [5:0] + Byte 5 | 14 | 5 | 0 | km | candidate |

<details>
<summary>Signal notes</summary>

- `SERVICE_DISTANCE`: candidate: raw*5 km; captured message displayed 4060 mi. Bench 2026-10-12: sending 0x51B changed nothing; the same layout in 0x553 drives the Engine Oil page

</details>

---

## 0x51C Page Slot Prerequisite A

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x51C</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Page Slot Prerequisite A</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

---

## 0x544 Vehicle Speed

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x544</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Vehicle Speed</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">4</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" align="center">VEHICLE_<wbr>SPEED_<wbr>KMH (km/h)</td>
  </tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `VEHICLE_SPEED_KMH` | Byte 0 | 8 | 1 | 0 | km/h | probable |

<details>
<summary>Signal notes</summary>

- `VEHICLE_SPEED_KMH`: captures: tracks 0x354 speed in whole km/h; 0xFF for the first ~10 s (not available)

</details>

---

## 0x54A HVAC 1

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x54A</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">HVAC 1</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>4</th>
    <td colspan="8" align="center">DRIVER_<wbr>TEMP</td>
  </tr>
  <tr>
    <th>5</th>
    <td colspan="8" align="center">PASSENGER_<wbr>TEMP</td>
  </tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `DRIVER_TEMP` | Byte 4 | 8 | 1 | 0 |  | documented |
| `PASSENGER_TEMP` | Byte 5 | 8 | 1 | 0 |  | documented |

---

## 0x54B HVAC 2

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x54B</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">HVAC 2</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>2</th>
    <td></td>
    <td></td>
    <td colspan="3" align="center">AIR_<wbr>DIRECTION</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <th>3</th>
    <td></td>
    <td align="center">DUAL</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">RECIRCULATION</td>
    <td></td>
  </tr>
  <tr>
    <th>4</th>
    <td></td>
    <td></td>
    <td colspan="4" align="center">BLOWER_<wbr>SPEED</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <th>5</th>
    <td></td>
    <td align="center">STEERING_<wbr>HEATER</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `AIR_DIRECTION` | Byte 2 [5:3] | 3 | 1 | 0 |  | documented |
| `DUAL` | Byte 3 [6] | 1 | 1 | 0 |  | documented |
| `RECIRCULATION` | Byte 3 [1] | 1 | 1 | 0 |  | documented |
| `BLOWER_SPEED` | Byte 4 [5:2] | 4 | 1 | 0 |  | documented |
| `STEERING_HEATER` | Byte 5 [6] | 1 | 1 | 0 |  | documented |

<details>
<summary>Signal notes</summary>

- `BLOWER_SPEED`: README masks 0x0C..0x3C are whole-byte; enum values here are the extracted field

</details>

### 0x54B · AIR_DIRECTION

| Value | Meaning |
|-------|---------|
| 0b001 (1) | Upper |
| 0b010 (2) | Upper + Lower |
| 0b011 (3) | Lower |
| 0b100 (4) | Defogger + Lower |
| 0b101 (5) | Defogger |

### 0x54B · BLOWER_SPEED

| Value | Meaning |
|-------|---------|
| 0b0011 (3) | Level 1 |
| 0b0101 (5) | Level 2 |
| 0b0111 (7) | Level 3 |
| 0b1001 (9) | Level 4 |
| 0b1011 (11) | Level 5 |
| 0b1101 (13) | Level 6 |
| 0b1111 (15) | Level 7 |

---

## 0x54C HVAC 3 (outside temperature)

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x54C</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">HVAC 3 (outside temperature)</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" align="center">TEMP_<wbr>B0_<wbr>CAND?</td>
  </tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>5</th>
    <td></td>
    <td colspan="4" align="center">BLOWER_<wbr>OUTPUT</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <th>6</th>
    <td colspan="8" align="center">OUTSIDE_<wbr>TEMP (×0.5 -41 degC)</td>
  </tr>
  <tr>
    <th>7</th>
    <td colspan="8" align="center">TEMP_<wbr>B7_<wbr>CAND?</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `TEMP_B0_CAND` | Byte 0 | 8 | 1 | 0 |  | candidate |
| `BLOWER_OUTPUT` | Byte 5 [6:3] | 4 | 1 | 0 |  | probable |
| `TEMP_B7_CAND` | Byte 7 | 8 | 1 | 0 |  | candidate |
| `OUTSIDE_TEMP` | Byte 6 | 8 | 0.5 | -41 | degC | confirmed |

<details>
<summary>Signal notes</summary>

- `TEMP_B0_CAND`: captures: 0xFF at start, settles ~100 with the A/C running, higher after hot soak; scale unknown
- `BLOWER_OUTPUT`: captures: ramps in 0.5 s steps and rises with 0x54B BLOWER_SPEED (L1 7, L2 8, L3 10-12, L4 13, off 0)
- `TEMP_B7_CAND`: captures: 0xFE until ready, ~180 at idle, drops to ~95-120 driving with A/C; scale unknown
- `OUTSIDE_TEMP`: raw = (degC + 41) * 2. Bench 2026-10-12 (camera): B1 -> 118 degF (idle baseline value), 20 degC -> 68 degF, -10 degC -> 14 degF, 0xFF -> '--- degF' (not available; also the first frame after engine start in engineonq50). B1-B4 undecoded. Owner 2026-10-12: the meter filters the reading - a decrease shows almost at once, an increase lags (does not follow in real time); rate/delay not measured.

</details>

---

## 0x54D HVAC 4 (seat heat)

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x54D</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">HVAC 4 (seat heat)</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>1</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">DRIVER_<wbr>SEAT_<wbr>AUTO</td>
    <td align="center">PASSENGER_<wbr>SEAT_<wbr>AUTO</td>
  </tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>5</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">DRIVER_<wbr>SEAT_<wbr>OFF</td>
    <td align="center">DRIVER_<wbr>SEAT_<wbr>LOW</td>
    <td align="center">DRIVER_<wbr>SEAT_<wbr>MID</td>
  </tr>
  <tr>
    <th>6</th>
    <td align="center">DRIVER_<wbr>SEAT_<wbr>HIGH</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">PASSENGER_<wbr>SEAT_<wbr>OFF</td>
    <td align="center">PASSENGER_<wbr>SEAT_<wbr>LOW</td>
    <td align="center">PASSENGER_<wbr>SEAT_<wbr>MID</td>
  </tr>
  <tr>
    <th>7</th>
    <td align="center">PASSENGER_<wbr>SEAT_<wbr>HIGH</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `DRIVER_SEAT_AUTO` | Byte 1 [1] | 1 | 1 | 0 |  | documented |
| `PASSENGER_SEAT_AUTO` | Byte 1 [0] | 1 | 1 | 0 |  | documented |
| `DRIVER_SEAT_OFF` | Byte 5 [2] | 1 | 1 | 0 |  | documented |
| `DRIVER_SEAT_LOW` | Byte 5 [1] | 1 | 1 | 0 |  | documented |
| `DRIVER_SEAT_MID` | Byte 5 [0] | 1 | 1 | 0 |  | documented |
| `DRIVER_SEAT_HIGH` | Byte 6 [7] | 1 | 1 | 0 |  | documented |
| `PASSENGER_SEAT_OFF` | Byte 6 [2] | 1 | 1 | 0 |  | documented |
| `PASSENGER_SEAT_LOW` | Byte 6 [1] | 1 | 1 | 0 |  | documented |
| `PASSENGER_SEAT_MID` | Byte 6 [0] | 1 | 1 | 0 |  | documented |
| `PASSENGER_SEAT_HIGH` | Byte 7 [7] | 1 | 1 | 0 |  | documented |

---

## 0x551 Engine (coolant / MIL / cruise)

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x551</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Engine (coolant / MIL / cruise)</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" align="center">COOLANT_<wbr>TEMP (-40 degC)</td>
  </tr>
  <tr>
    <th>1</th>
    <td colspan="8" align="center">FUEL_<wbr>USAGE_<wbr>COUNTER</td>
  </tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>3</th>
    <td align="center">IG_<wbr>READY</td>
    <td align="center">MIL_<wbr>FLASH</td>
    <td align="center">MIL_<wbr>SOLID</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <th>4</th>
    <td colspan="8" align="center">CRUISE_<wbr>SET_<wbr>SPEED (mph)</td>
  </tr>
  <tr>
    <th>5</th>
    <td colspan="4" align="center">CRUISE_<wbr>ICON</td>
    <td></td>
    <td></td>
    <td colspan="2" align="center">ENGINE_<wbr>STATE</td>
  </tr>
  <tr>
    <th>6</th>
    <td></td>
    <td align="center">ECO_<wbr>MODE</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `COOLANT_TEMP` | Byte 0 | 8 | 1 | -40 | degC | probable |
| `FUEL_USAGE_COUNTER` | Byte 1 | 8 | 1 | 0 |  | confirmed |
| `MIL_SOLID` | Byte 3 [5] | 1 | 1 | 0 |  | documented |
| `MIL_FLASH` | Byte 3 [6] | 1 | 1 | 0 |  | confirmed |
| `IG_READY` | Byte 3 [7] | 1 | 1 | 0 |  | probable |
| `CRUISE_SET_SPEED` | Byte 4 | 8 | 1 | 0 | mph | documented |
| `CRUISE_ICON` | Byte 5 [7:4] | 4 | 1 | 0 |  | confirmed |
| `ENGINE_STATE` | Byte 5 [1:0] | 2 | 1 | 0 |  | probable |
| `ECO_MODE` | Byte 6 [6] | 1 | 1 | 0 |  | probable |

<details>
<summary>Signal notes</summary>

- `COOLANT_TEMP`: bench encoding raw = degC + 40
- `FUEL_USAGE_COUNTER`: cumulative fuel use; omitting 0x551 while moving spikes economy; bench 2026-10-12: instantaneous MPG = 13.1 x mph / counts per s: 30 mph at 19.65 and 9.83 counts/s showed 20 and 40 mpg (q50tb/sim/fuel_economy.py)
- `MIL_FLASH`: bench 2026-10-12: MIL blinks
- `IG_READY`: ignition on/ready, not engine running: set the whole IG-on/engine-off capture (BENCHMETER_MAINBUS_STARTUP, RPM 0 for 60 s); cleared 0.1 s after IG off (engineonq50)
- `CRUISE_SET_SPEED`: 0x33 displayed 51 mph
- `CRUISE_ICON`: upper-nibble whole-byte masks; interaction with 0x2B1 not isolated; bench 2026-10-12: 1-6, 9, 12 shown as listed with CRUISE_SET_SPEED
- `ENGINE_STATE`: captures: same states as 0x1F9 B0[5:3]
- `ECO_MODE`: captures: 1 in Eco (and Personal set to Eco), with 0x174 SHIFT_MODE = Eco; no visible meter effect

</details>

### 0x551 · CRUISE_ICON

| Value | Meaning |
|-------|---------|
| 0b0000 (0) | None |
| 0b0001 (1) | Green up-arrow CC |
| 0b0010 (2) | White up-arrow CC |
| 0b0011 (3) | Green up-arrow blink / no speed |
| 0b0100 (4) | Green speed-set CC |
| 0b0101 (5) | White speed-set CC |
| 0b0110 (6) | Green speed-set blink / no speed |
| 0b1001 (9) | Green up-arrow, numeric speed blinking |
| 0b1100 (12) | Green speed-set, numeric speed blinking |

### 0x551 · ENGINE_STATE

| Value | Meaning |
|-------|---------|
| 0b00 (0) | Off |
| 0b01 (1) | IG on, stopped |
| 0b10 (2) | Running |
| 0b11 (3) | Cranking |

---

## 0x553 Oil Service

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x553</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Oil Service</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">6</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>4</th>
    <td></td>
    <td></td>
    <td colspan="6" align="center">SERVICE_<wbr>DISTANCE ↓</td>
  </tr>
  <tr>
    <th>5</th>
    <td colspan="8" align="center">SERVICE_<wbr>DISTANCE (×5 km)</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `SERVICE_DISTANCE` | Byte 4 [5:0] + Byte 5 | 14 | 5 | 0 | km | confirmed |

<details>
<summary>Signal notes</summary>

- `SERVICE_DISTANCE`: Engine Oil page 'Service due in': raw*5 km (bench 2026-10-12: 0x0535 -> 4141 mi, 0x0100 -> 795 mi, 0x0010 -> 49 mi). Byte 4 bits 7:6 not exercised.

</details>

---

## 0x56C Chassis Control / Drive Mode

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x56C</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Chassis Control / Drive Mode</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">6</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td></td>
    <td align="center">TRACE_<wbr>BLUE_<wbr>SQUARES</td>
    <td colspan="2" align="center">TRACE_<wbr>SQUARE_<wbr>LINES</td>
    <td></td>
    <td></td>
    <td align="center">CHASSIS_<wbr>ERROR</td>
    <td></td>
  </tr>
  <tr>
    <th>1</th>
    <td></td>
    <td align="center">TRACE_<wbr>SADDLE_<wbr>CURVE</td>
    <td colspan="3" align="center">DRIVE_<wbr>MODE</td>
    <td align="center">QUAD_<wbr>BEEP</td>
    <td align="center">CONTINUOUS_<wbr>BEEP</td>
    <td></td>
  </tr>
  <tr>
    <th>2</th>
    <td></td>
    <td align="center">FL_<wbr>WHEEL_<wbr>HIGHLIGHT</td>
    <td></td>
    <td align="center">FR_<wbr>WHEEL_<wbr>HIGHLIGHT</td>
    <td></td>
    <td align="center">RL_<wbr>WHEEL_<wbr>HIGHLIGHT</td>
    <td></td>
    <td align="center">RR_<wbr>WHEEL_<wbr>HIGHLIGHT</td>
  </tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>5</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td colspan="2" align="center">COUNTER</td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `TRACE_BLUE_SQUARES` | Byte 0 [6] | 1 | 1 | 0 |  | confirmed |
| `TRACE_SQUARE_LINES` | Byte 0 [5:4] | 2 | 1 | 0 |  | confirmed |
| `CHASSIS_ERROR` | Byte 0 [1] | 1 | 1 | 0 |  | confirmed |
| `TRACE_SADDLE_CURVE` | Byte 1 [6] | 1 | 1 | 0 |  | confirmed |
| `DRIVE_MODE` | Byte 1 [5:3] | 3 | 1 | 0 |  | confirmed |
| `QUAD_BEEP` | Byte 1 [2] | 1 | 1 | 0 |  | documented |
| `CONTINUOUS_BEEP` | Byte 1 [1] | 1 | 1 | 0 |  | documented |
| `FL_WHEEL_HIGHLIGHT` | Byte 2 [6] | 1 | 1 | 0 |  | confirmed |
| `FR_WHEEL_HIGHLIGHT` | Byte 2 [4] | 1 | 1 | 0 |  | confirmed |
| `RL_WHEEL_HIGHLIGHT` | Byte 2 [2] | 1 | 1 | 0 |  | confirmed |
| `RR_WHEEL_HIGHLIGHT` | Byte 2 [0] | 1 | 1 | 0 |  | confirmed |
| `COUNTER` | Byte 5 [1:0] | 2 | 1 | 0 |  | probable |

<details>
<summary>Signal notes</summary>

- `TRACE_BLUE_SQUARES`: bench 2026-10-12: Chassis Control page grid graphic
- `TRACE_SQUARE_LINES`: number of blue-square lines beside the car (bench 2026-10-12: 10 -> 1, 20 -> 2, 30 -> 3); formerly two flags TRACE_BLUE_LINES_WHEELS (bit 4) and TRACE_2_SQUARES_WHEELS (bit 5)
- `CHASSIS_ERROR`: mask 0x02 associated with chassis-system error; bench 2026-10-12: 'Chassis Control System Error'; clears when the bit clears
- `TRACE_SADDLE_CURVE`: bench 2026-10-12: Chassis Control page grid graphic
- `DRIVE_MODE`: bench 2026-10-12: all six modes shown (ECO green, SPORT/SPORT+ red)
- `FL_WHEEL_HIGHLIGHT`: bench 2026-10-12: wheel highlighted on Chassis Control page
- `FR_WHEEL_HIGHLIGHT`: bench 2026-10-12: wheel highlighted on Chassis Control page
- `RL_WHEEL_HIGHLIGHT`: bench 2026-10-12: wheel highlighted on Chassis Control page
- `RR_WHEEL_HIGHLIGHT`: bench 2026-10-12: wheel highlighted on Chassis Control page

</details>

### 0x56C · TRACE_SQUARE_LINES

| Value | Meaning |
|-------|---------|
| 0b00 (0) | None |
| 0b01 (1) | 1 line |
| 0b10 (2) | 2 lines |
| 0b11 (3) | 3 lines |

### 0x56C · DRIVE_MODE

| Value | Meaning |
|-------|---------|
| 0b000 (0) | Standard |
| 0b001 (1) | Sport |
| 0b010 (2) | Eco |
| 0b011 (3) | Snow |
| 0b100 (4) | Personal |
| 0b101 (5) | Sport Plus |

> **Notes**
>
> - need to verify DLC

---

## 0x56E Telematics

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x56E</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Telematics</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">4</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td align="center">UNKNOWN_<wbr>B0_<wbr>7?</td>
    <td align="center">UNKNOWN_<wbr>B0_<wbr>6?</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">UNLOCK_<wbr>CMD</td>
  </tr>
  <tr>
    <th>1</th>
    <td align="center">LOCK_<wbr>CMD</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>3</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `UNKNOWN_B0_7` | Byte 0 [7] | 1 | 1 | 0 |  | candidate |
| `UNKNOWN_B0_6` | Byte 0 [6] | 1 | 1 | 0 |  | candidate |
| `UNLOCK_CMD` | Byte 0 [0] | 1 | 1 | 0 |  | documented |
| `LOCK_CMD` | Byte 1 [7] | 1 | 1 | 0 |  | documented |

<details>
<summary>Signal notes</summary>

- `UNKNOWN_B0_7`: bench 2026-10-12: no visible change
- `UNKNOWN_B0_6`: bench 2026-10-12: no visible change
- `UNLOCK_CMD`: bench 2026-10-12: no visible change
- `LOCK_CMD`: bench 2026-10-12: no visible change

</details>

> **Notes**
>
> - Baseline: `86 00 00 00`.

---

## 0x580 Engine 3

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x580</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Engine 3</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" rowspan="2" align="center">ENGINE_<wbr>TORQUE (×0.0625 Nm)</td>
  </tr>
  <tr>
    <th>1</th>
  </tr>
  <tr>
    <th>2</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">LOW_<wbr>OIL_<wbr>PRESSURE</td>
  </tr>
  <tr>
    <th>3</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">LOOSE_<wbr>FUEL_<wbr>CAP</td>
  </tr>
  <tr>
    <th>4</th>
    <td colspan="8" align="center">OIL_<wbr>TEMP? (-40 degC)</td>
  </tr>
  <tr>
    <th>5</th>
    <td></td>
    <td align="center">ECO_<wbr>LIGHT</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>7</th>
    <td></td>
    <td></td>
    <td align="center">ENGINE_<wbr>RUNNING?</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `ENGINE_TORQUE` | Bytes 0–1 | 16 | 0.0625 | 0 | Nm | documented |
| `LOW_OIL_PRESSURE` | Byte 2 [0] | 1 | 1 | 0 |  | documented |
| `LOOSE_FUEL_CAP` | Byte 3 [0] | 1 | 1 | 0 |  | documented |
| `OIL_TEMP` | Byte 4 | 8 | 1 | -40 | degC | candidate |
| `ECO_LIGHT` | Byte 5 [6] | 1 | 1 | 0 |  | documented |
| `ENGINE_RUNNING` | Byte 7 [5] | 1 | 1 | 0 |  | candidate |

<details>
<summary>Signal notes</summary>

- `LOW_OIL_PRESSURE`: owner: lights the oil warning while RPM > 0 and goes off at RPM 0 (one camera test at 3000 rpm on 2026-10-12 did not catch it)
- `OIL_TEMP`: README 'raw degC + 40' is ambiguous (+40 vs -40); bench 2026-10-12: no visible change (150 degC)
- `ENGINE_RUNNING`: engine-state flag (all real-car captures incl. 60 s IG-on/engine-off: 0 off, 1 running with the engine, one change at start and stop). Driving captures: B7 is a numeric, load-like value (~0x19 engine off, ~0x3C idle, 0x4B-0x50 cruising, ~0x76 full throttle), so bit 5 only tracks the engine at idle; 0 most of the time while driving

</details>

---

## 0x5B0 Drive Mode

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x5B0</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Drive Mode</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">2</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td colspan="8" align="center">DRIVE_<wbr>MODE_<wbr>FLAGS</td>
  </tr>
  <tr>
    <th>1</th>
    <td colspan="2" align="center">COUNTER</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `DRIVE_MODE_FLAGS` | Byte 0 | 8 | 1 | 0 |  | probable |
| `COUNTER` | Byte 1 [7:6] | 2 | 1 | 0 |  | probable |

<details>
<summary>Signal notes</summary>

- `DRIVE_MODE_FLAGS`: captures: one-hot, follows the 0x56C drive mode (Personal shows its configured mode); no visible meter effect

</details>

### 0x5B0 · DRIVE_MODE_FLAGS

| Value | Meaning |
|-------|---------|
| 0x01 (1) | Sport |
| 0x04 (4) | Standard |
| 0x10 (16) | Eco |
| 0x40 (64) | Snow |

---

## 0x5C5 Cluster Status 4 (meter)

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x5C5</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Cluster Status 4 (meter)</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Meter (cluster)</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td align="center">METER_<wbr>OFF</td>
    <td align="center">METER_<wbr>ON</td>
    <td align="center">GAS_<wbr>LIGHT</td>
    <td></td>
    <td align="center">LOW_<wbr>BRAKE_<wbr>FLUID</td>
    <td align="center">PARKING_<wbr>BRAKE</td>
    <td align="center">RUNNING?</td>
    <td></td>
  </tr>
  <tr>
    <th>1</th>
    <td colspan="8" rowspan="3" align="center">ODOMETER_<wbr>RAW</td>
  </tr>
  <tr>
    <th>2</th>
  </tr>
  <tr>
    <th>3</th>
  </tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `METER_OFF` | Byte 0 [7] | 1 | 1 | 0 |  | documented |
| `METER_ON` | Byte 0 [6] | 1 | 1 | 0 |  | documented |
| `GAS_LIGHT` | Byte 0 [5] | 1 | 1 | 0 |  | documented |
| `LOW_BRAKE_FLUID` | Byte 0 [3] | 1 | 1 | 0 |  | documented |
| `PARKING_BRAKE` | Byte 0 [2] | 1 | 1 | 0 |  | documented |
| `RUNNING` | Byte 0 [1] | 1 | 1 | 0 |  | candidate |
| `ODOMETER_RAW` | Bytes 1–3 | 24 | 1 | 0 | mi | documented |

<details>
<summary>Signal notes</summary>

- `ODOMETER_RAW`: big-endian, hex value of odometer in miles
</details>

---

## 0x5E4 Power Steering

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x5E4</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Power Steering</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">3</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">POWER_<wbr>STEERING_<wbr>WARNING</td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `POWER_STEERING_WARNING` | Byte 0 [2] | 1 | 1 | 0 |  | confirmed |

<details>
<summary>Signal notes</summary>

- `POWER_STEERING_WARNING`: red steering-wheel telltale (bench 2026-10-12, byte 0 = 0x04); part of the startup lamp check

</details>

---

## 0x60D BCM

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x60D</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">BCM</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td align="center">TRUNK</td>
    <td align="center">REAR_<wbr>PASSENGER_<wbr>DOOR</td>
    <td align="center">REAR_<wbr>DRIVER_<wbr>DOOR</td>
    <td align="center">PASSENGER_<wbr>DOOR</td>
    <td align="center">DRIVER_<wbr>DOOR</td>
    <td align="center">PARKING_<wbr>LIGHTS</td>
    <td align="center">HEADLIGHTS</td>
    <td></td>
  </tr>
  <tr>
    <th>1</th>
    <td></td>
    <td align="center">RIGHT_<wbr>TURN</td>
    <td align="center">LEFT_<wbr>TURN</td>
    <td></td>
    <td align="center">HIGH_<wbr>BEAM</td>
    <td align="center">IG_<wbr>RUN</td>
    <td align="center">ACC</td>
    <td align="center">FOG_<wbr>LIGHTS</td>
  </tr>
  <tr>
    <th>2</th>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">ALL_<wbr>DOORS_<wbr>LOCKED</td>
    <td></td>
    <td align="center">REAR_<wbr>FOG</td>
    <td></td>
    <td></td>
  </tr>
  <tr>
    <th>3</th>
    <td></td>
    <td></td>
    <td colspan="5" align="center">CHIMES</td>
    <td></td>
  </tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>6</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `TRUNK` | Byte 0 [7] | 1 | 1 | 0 |  | documented |
| `REAR_PASSENGER_DOOR` | Byte 0 [6] | 1 | 1 | 0 |  | documented |
| `REAR_DRIVER_DOOR` | Byte 0 [5] | 1 | 1 | 0 |  | documented |
| `PASSENGER_DOOR` | Byte 0 [4] | 1 | 1 | 0 |  | documented |
| `DRIVER_DOOR` | Byte 0 [3] | 1 | 1 | 0 |  | documented |
| `PARKING_LIGHTS` | Byte 0 [2] | 1 | 1 | 0 |  | documented |
| `HEADLIGHTS` | Byte 0 [1] | 1 | 1 | 0 |  | documented |
| `RIGHT_TURN` | Byte 1 [6] | 1 | 1 | 0 |  | documented |
| `LEFT_TURN` | Byte 1 [5] | 1 | 1 | 0 |  | documented |
| `HIGH_BEAM` | Byte 1 [3] | 1 | 1 | 0 |  | documented |
| `IG_RUN` | Byte 1 [2] | 1 | 1 | 0 |  | documented |
| `ACC` | Byte 1 [1] | 1 | 1 | 0 |  | documented |
| `FOG_LIGHTS` | Byte 1 [0] | 1 | 1 | 0 |  | documented |
| `ALL_DOORS_LOCKED` | Byte 2 [4] | 1 | 1 | 0 |  | documented |
| `REAR_FOG` | Byte 2 [2] | 1 | 1 | 0 |  | documented |
| `CHIMES` | Byte 3 [5:1] | 5 | 1 | 0 |  | documented |

<details>
<summary>Signal notes</summary>

- `HEADLIGHTS`: with LH/RH headlight inputs off this latches 'Headlight System Error'
- `CHIMES`: startup byte 0x2A = field 0x15

</details>

### 0x60D · CHIMES

| Value | Meaning |
|-------|---------|
| 0b00000 (0) | None |
| 0b00001 (1) | Headlight reminder |
| 0b10000 (16) | Door open chime |
| 0b10010 (18) | Single short high-pitch beep |
| 0b10011 (19) | Periodic quad beep |
| 0b10101 (21) | Startup beep |
| 0b11000 (24) | Long constant beep |
| 0b11001 (25) | Long constant beep (0x19) |

---

## 0x625 BCM / IPDM

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x625</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">BCM / IPDM</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">6</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7">100 ms</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr>
    <th>0</th>
    <td></td>
    <td></td>
    <td colspan="2" align="center">START_<wbr>STATE</td>
    <td></td>
    <td align="center">WIPER_<wbr>LOW</td>
    <td align="center">WIPERS_<wbr>NOT_<wbr>AT_<wbr>TOP</td>
    <td></td>
  </tr>
  <tr>
    <th>1</th>
    <td align="center">AC_<wbr>COMPRESSOR_<wbr>OR_<wbr>FAN</td>
    <td align="center">PARKING_<wbr>LIGHTS</td>
    <td align="center">HEADLIGHT</td>
    <td align="center">HIGH_<wbr>BEAM</td>
    <td align="center">FOG_<wbr>LIGHTS</td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>2</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>3</th>
    <td></td>
    <td></td>
    <td></td>
    <td align="center">IG_<wbr>RUN</td>
    <td></td>
    <td></td>
    <td colspan="2" align="center">AWAKE_<wbr>STATE?</td>
  </tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `START_STATE` | Byte 0 [5:4] | 2 | 1 | 0 |  | documented |
| `WIPER_LOW` | Byte 0 [2] | 1 | 1 | 0 |  | documented |
| `WIPERS_NOT_AT_TOP` | Byte 0 [1] | 1 | 1 | 0 |  | documented |
| `AC_COMPRESSOR_OR_FAN` | Byte 1 [7] | 1 | 1 | 0 |  | documented |
| `PARKING_LIGHTS` | Byte 1 [6] | 1 | 1 | 0 |  | documented |
| `HEADLIGHT` | Byte 1 [5] | 1 | 1 | 0 |  | documented |
| `HIGH_BEAM` | Byte 1 [4] | 1 | 1 | 0 |  | documented |
| `FOG_LIGHTS` | Byte 1 [3] | 1 | 1 | 0 |  | documented |
| `IG_RUN` | Byte 3 [4] | 1 | 1 | 0 |  | documented |
| `AWAKE_STATE` | Byte 3 [1:0] | 2 | 1 | 0 |  | candidate |

<details>
<summary>Signal notes</summary>

- `START_STATE`: bench 2026-10-12: 1 and 2: no visible change
- `AWAKE_STATE`: captures: 1 while IG is on and for 4.9 s after each wake event (door, button, unlock), otherwise 2

</details>

### 0x625 · START_STATE

| Value | Meaning |
|-------|---------|
| 0b00 (0) | Normal |
| 0b01 (1) | ? (1) |
| 0b10 (2) | ? (2) |
| 0b11 (3) | Cranking |

### 0x625 · AWAKE_STATE

| Value | Meaning |
|-------|---------|
| 0b01 (1) | Awake |
| 0b10 (2) | Standby |

---

## 0x6E2 Unresolved 0x6E2 (tach gate)

<table>
  <tr>
    <th colspan="2" align="left">CAN ID</th>
    <td colspan="7">0x6E2</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Description</th>
    <td colspan="7">Unresolved 0x6E2 (tach gate)</td>
  </tr>
  <tr>
    <th colspan="2" align="left">DLC</th>
    <td colspan="7">8</td>
  </tr>
  <tr>
    <th colspan="2" align="left">Interval</th>
    <td colspan="7"></td>
  </tr>
  <tr>
    <th colspan="2" align="left">Sender</th>
    <td colspan="7">Vehicle</td>
  </tr>

  <tr><td colspan="9"></td></tr>   <!-- spacer -->

  <tr>
    <th>Byte</th>
    <th>Bit 7</th><th>Bit 6</th><th>Bit 5</th><th>Bit 4</th>
    <th>Bit 3</th><th>Bit 2</th><th>Bit 1</th><th>Bit 0</th>
  </tr>
  <tr><th>0</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>1</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>2</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td colspan="3" align="center">PAGE</td>
  </tr>
  <tr>
    <th>3</th>
    <td colspan="8" align="center">RPM_<wbr>DIV64_<wbr>CAND?</td>
  </tr>
  <tr><th>4</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr><th>5</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
  <tr>
    <th>6</th>
    <td></td>
    <td align="center">TACH_<wbr>ZERO_<wbr>FLAG</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
  <tr><th>7</th><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
</table>

| Signal | Position | Length | Factor | Offset | Unit | Status |
|--------|----------|--------|--------|--------|------|--------|
| `PAGE` | Byte 2 [2:0] | 3 | 1 | 0 |  | probable |
| `RPM_DIV64_CAND` | Byte 3 | 8 | 1 | 0 |  | candidate |
| `TACH_ZERO_FLAG` | Byte 6 [6] | 1 | 1 | 0 |  | probable |

<details>
<summary>Signal notes</summary>

- `PAGE`: captures: B2 = 0x38 + page; pages 0-3 with the engine off, 4-7 running; each page ~400 ms
- `RPM_DIV64_CAND`: captures: follows engine rpm (~0x0B idle, 0x43 at 4300 rpm) on pages 4-7
- `TACH_ZERO_FLAG`: bench sweep 2026-10-12 (camera): bit set -> tach needle drops from ~700 rpm idle to 0, returns when cleared (5 of 5); no telltale. Other bits of byte 6 do nothing. Meaning (engine-stopped / RPM-invalid?) not established.

</details>
