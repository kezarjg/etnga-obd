---
name: Plug-in Charge Control / OBC + DC-DC Converter (Toyota EM "A33")
diagnostic_request_id: "0x745"
diagnostic_response_id: "0x74D"
toyota_name: Plug-in Charge Control
em_reference: A33 (Electric Converter Unit — OBC + DC-DC)
isotp: standard
sessions_observed:
  - default                # 0x01
confidence: high                           # Techstream-confirmed 2026-05-08
---

# Plug-in Charge Control (= OBC + DC-DC Converter, Toyota EM "A33")

This ECU **is the On-Board Charger (OBC) plus DC-DC Converter** — Toyota EM component code **A33 "Electric Converter Unit"**. The Techstream label "Plug-in Control" is misleading: it's not just a session manager, it's the entire OBC hardware ECU. The Data List inventory makes this unambiguous — it owns PFC, AC/DC inlet temperatures, charger I/O, and both main and sub DC/DC converters.

> **History notes**:
> - Until 2026-05-08, this ECU was mislabeled "BMS Monitor / Battery Manager" — corrected by the mapping marathon.
> - Through most of 2026-05-08, it was further mislabeled as "just the charge-session controller" with the OBC presumed to live elsewhere. **Re-corrected late on 2026-05-08** when the full Techstream Data List inventory revealed PFC / DC-DC / charger-I/O parameters that only the OBC hardware ECU could expose. There is no separate OBC ECU — `0x745` is it.

## Diagnostics

| Aspect | Value |
|---|---|
| Request ID | `0x745` (standard 11-bit) |
| Response ID | `0x74D` (request + 8) |
| Transport | Standard ISO-TP |
| Confirmed services | `0x10` SessionControl, `0x19` ReadDTCInformation, `0x22` ReadDataByIdentifier, `0x2C` DynamicallyDefineDataIdentifier, `0x31` RoutineControl, `0x3E` TesterPresent, `0xAB` (Toyota proprietary, observed in Health Check) |
| Stored DTCs | none observed |
| Data List polling | **confirmed 2026-05-09: dynamic DIDs `0xF301` (40 sources, 135-byte body) and `0xF302` (40 sources, 59-byte body)** — same idiom as EV ECU. Also 71 direct-poll DIDs read in parallel. |

## Functional roles (per Data List inventory, 2026-05-08 snapshot)

This ECU bundles several functions a typical Toyota EV would split across multiple modules:

1. **OBC (On-Board AC Charger)** — converts grid AC to HV DC during AC charging
   - PFC Boosting Circuit (with driver state, voltage, current, amplitude, temperature)
   - VDC voltage measurement
   - AC charging relay control (positive/negative/precharge)
   - AC inlet temperature monitoring (positive and negative inlets)
   - AC input voltage instantaneous samples (16 samples for waveform analysis)
   - AC input voltage / current / charging power measurements
   - Charging Current Duty from Charger (PWM duty cycle for J1772 control pilot)
   - Single Phase / Three Phase detection
2. **DC-DC Converter** — feeds the 12V auxiliary battery from HV
   - Main DC/DC Converter Operation Status, target current, temperature
   - Sub DC/DC Converter (separate, charging-specific): operation status, drive request, target voltage, output V/I, temperature, consumption
3. **DC Fast Charging interface** — handles CCS communication and DC inlet path
   - HLC Communication Sequence Status (HomePlug Green PHY for ISO 15118)
   - DC Charger Station present/min/max output V/I/P (CCS)
   - DC inlet temperatures × 2 sensors
   - DCR (DC Charge Relay) temperatures × 2, drive permission, stuck-closed diagnosis
   - DC operation mode, charge-stop reason flags (multiple)
4. **Charge session management**
   - Charging history information (last session result, e.g. "AC Charging Complete (Full Charge)")
   - Total Number of AC Charging (lifetime counter)
   - AC Charging Total Time (lifetime minutes)
   - Charging required time / elapsed time / state elapsed time
   - Target precharge voltage
   - Charge Amount Upper Limit Setting (user-configurable charge target SOC)
   - 6A Charging Mode Switching History
5. **Charge connector and lock**
   - Charging Lid switch / lamp / open-close state
   - Charging Connector Connect Status / voltage
   - Charging Connector Lock Pin Status, motor lock/unlock direction request currents
   - Connector Unlock History during Charging
6. **Cabin / climate gating during charging**
   - A/C Useable Power, A/C Consumption Power
   - Remote Air Control System status, history
   - My Room Operation status, history (V2L cabin power)
   - Water Heater Power Consumption / Command Prohibition Factor
7. **Battery thermal coordination during charging**
   - Battery Temperature when Charging Start (logged value)
   - Battery Max/Min Temperature during Charging (logged values)
   - Battery Charging/Power Feeding Permission with Thermal Keep
   - Battery Control Status on Thermal Keeping and Charging
   - Battery Temperature Rising History / Cooling History flags
8. **Solar option** (bZ4X had a solar roof option; not equipped on the test vehicle, but parameters exist)
   - Solar Available Information / Specification / Switching

This breadth confirms `0x745` = the integrated A33 ECU.

## DIDs (high-confidence — from public ABRP bZ4X/Solterra OBD config)

| DID | Meaning | Encoding | Notes |
|---|---|---|---|
| `0x1739` | **State of Charge (SOC)** | uint8, 0–100 % | The charge-controller's view of SOC. May be the same value as `0x747` `0x1F5B` byte 1, or biased (e.g. the "DC Charger Display SOC" was 95 % vs EV Battery's 89 %). |
| `0x10D1` | **Charging-state enum** | uint8 enum | ABRP: `A == 3` → "actively charging". Other enum values not yet pinned. |
| `0x1668` | **Charge-mode / DCFC-state enum** | uint8 enum | ABRP: `A == 5` → "DC fast charging". Other enum values not yet pinned. |

## Notable Data List values from the 2026-05-08 snapshot (idle, Ready ON, not charging)

| Parameter | Value | Significance for telemetry / preconditioning |
|---|---|---|
| **Hybrid/EV Battery Temperature when Charging Start** | **32 F (0 °C)** | Logged value from last charge — direct evidence of cold-weather DCFC happening on the test vehicle. Strong preconditioning argument. |
| Charging History Information | "AC Charging Complete (Full Charge)" | Last-charge state is queryable as an enum |
| Total Number of AC Charging | 66 | Lifetime AC charge counter |
| AC Charging Total Time | 18598 min (~310 h) | Lifetime AC charge time |
| Hybrid/EV Battery SOC (DC Charger Display) | 95 % | The biased SOC presented to DCFC stations (vs EV Battery's 89 % "real" SOC) |
| Hybrid/EV Battery SOC (Meter Display) | 95 % | What the dashboard cluster displays |
| Hybrid/EV Battery Control Status on Thermal Keeping and Charging | "Unoperated" | Confirms there *is* a "Thermal Keep" feature in Toyota's logic; currently inactive. May be related to scheduled charge / pre-departure thermal prep. |
| Charge Amount Upper Limit Setting | "Full" | The user-configurable charge-target SOC Customize Parameter — exposed on this ECU. DID not yet pinned. |
| HV/EV Battery Total Voltage | 392.0 V | Same as EV Battery's `0x1F9A` reading |
| Charging Voltage for Hybrid/EV Battery | 0.0 V | Idle (not charging) |
| Hybrid/EV Battery Charging Power | -0.94 kW | Negative = battery feeding aux loads at idle |
| AC Power Supply Rated Current | 262.14 A | Sensor sentinel / max — not real until plugged in |
| PFC Temperature, DC/DC Converter Temperature | -58 F | Sentinel "off" value — only valid when charging |
| DC Charging Inlet Temperature 1, 2 | 68 F | Real (ambient-equilibrated) |
| AC Charging Positive/Negative Inlet Temp | 68 F | Real |
| DCR Temperature 1, 2 | 68 / 70 F | Real |
| Plug-in Control Module System Voltage (Minus) | -51.0 V | Possibly a charge-cable PE/CP voltage measurement (or sensor sentinel) |
| Power Supply Voltage (SP12) | 12.74 V | This ECU's internal 12V supply |

## Observed-but-unidentified DIDs (from Health Check)

During a Toyota Health Check (Techstream's full bus fan-out), this ECU was polled for: `0x1C00` (supported-DID list per Toyota convention), `0x1CB4-B7`, `0x1CBA`, `0x1CC6/C8/C9/CD`, `0x1CED/F1/F3/F5/F6/F7`, `0x1D00`, `0x1D41-D44`, `0x1D6B/D/E`, `0xF181` (ApplicationSoftwareIdentification).

The 247-byte responses on `0x1D41-D44` likely correspond to charge-session log records or per-session history (the Data List has many "history" parameters like "Connector Unlock History during Charging", "AC Charging Input Minimum Voltage History", "Power Limit Operation History", etc.). These are good targets for a future direct DID dump.

## Decoded signal summary

| Signal | DID / Source | Status |
|---|---|---|
| Pack SOC (charge-controller view) | `0x745` `0x1739` (or `0x747` `0x1F5B` for the unbiased view) | Decoded |
| Charging-active flag | `0x745` `0x10D1` (`A == 3`) | Decoded |
| Charge type (DC vs AC) | `0x745` `0x1668` (`A == 5` = DCFC) | Partial — only DCFC enum value identified |
| Full charge-state enum | `0x745` `0x10D1` enum (full mapping needed) | Need to enumerate values |
| Charge target SOC (user setting) | TBD — readable on this ECU as "Charge Amount Upper Limit Setting" | Isolate via Data List to find DID |
| Charging voltage | TBD — "Charging Voltage for Hybrid/EV Battery" Data List item | Isolate to find DID |
| Charging power | TBD — "Hybrid/EV Battery Charging Power" Data List item | Isolate |
| Charging kWh delivered | TBD — derive from charge time + power, or look for "Charging Required Time" / "Charging Elapsed Time" related | Open |
| Charge efficiency | TBD — derive from "Charger Input Power" vs "Charger Output Power" | Open |
| Total AC charge count / time | TBD — DIDs underlying the lifetime counters | Useful for lifetime tracking |
| Last-charge logged temperatures | TBD — "Battery Temperature when Charging Start" / "Max/Min during Charging" | Strong signal for preconditioning trigger |

## Implications for preconditioning

This ECU has the **logged record of cold-weather charging** (32 F start temp from last charge). For preconditioning logic, this means:

1. Historic charging conditions can be **tracked** to know whether preconditioning is even worth the energy (consistently warm plug-in conditions = no need).
2. **"Thermal Keep" parameters** (currently "Unoperated") suggest Toyota has *some* preconditioning infrastructure already in firmware — worth investigating whether triggering it via a known UDS command would be cleaner than driving the heater/pump tests directly. May be an undiscovered routine ID.
3. The **biased "DC Charger Display SOC"** (95 % vs 89 % real) is what DCFC stations see for charging negotiation — important to understand for any future DCFC-related telemetry.

## Open questions

- **DID for the charge-target-SOC user setting** — isolate "Charge Amount Upper Limit Setting" alone in the Data List to find its source DID. Then writable via `0x2E` WriteDataByIdentifier.
- **DIDs for live charging V/I/P** — partially identified 2026-05-09; positions tentative — single-source isolation needed.
- **`0x1D41-D44` 247-byte payloads** — likely charge-session history. Trigger known charge events and re-read to decode the structure.
- **"Thermal Keep" routine** — is there a UDS routine ID that triggers Toyota's built-in preconditioning, parallel to `0x1124` Cooling? Search for `0x31 01 RR RR` patterns when initiating a scheduled charge or thermal prep. Could be cleaner than driving the heater/pump tests directly.

## Source DID layout — captured 2026-05-09 L2 plug-in session

The 2026-05-09 charge-cycle capture (IG-OFF → L2 plug-in → AC charge initiation, 471 sec window) caught both `2C 01` defines on the OBC and the entire active-charging dialog. Authoritative table:

### F301 (40 sources, 135-byte response body)

| Source DID | Bytes | Decoded purpose |
|---|--:|---|
| `0x160E` | 16 | **16 × AC Input Voltage Instantaneous Value (waveform monitoring)** — bias-128 signed bytes; all 0x03 (sentinel) before charge, modulating around 0x80 (= zero offset) during AC charging. ✓ |
| `0x1648` | 10 | Lifetime charging counters cluster: bytes 5-6 = **Total Number of AC Charging** u16 BE (=68 after this session, was 67 before — incremented during capture, conclusively identified) ✓; bytes 9-10 = **AC Charging Total Time** u16 BE in minutes (=18679, matches baseline) ✓ |
| `0x1632` | 6 | AC Inlet sensor cluster — bytes 1-2 = **AC Charging Positive Inlet Temp Sensor Voltage** u16 BE × 5/4096 V ✓; bytes 3-4 + 5-6 = paired temp/related (TBD) |
| `0x1705` | 6 | DC Charging Inlet sensor cluster (same shape as `0x1632`) |
| `0x1006` | 5 | constant during charging — config |
| `0x1619` | 5 | mostly sentinels (offset-binary zeros) |
| `0x1656` | 5 | small slow drift during charging — likely a counter |
| `0x16A5` | 4 | bytes 1-2 = **HV/EV Battery Total Voltage** u16 BE × 1 V/LSB (= 392 during charging, matches baseline) ✓ |
| `0x1699` | 4 | constant `00 09 3a 80` — config sentinel |
| `0x0103` | 4 | **Odometer + unit** (same as EV ECU; `0x02 00 61 B1` = unit `mile` + 25009 mile, synthetic) ✓ |
| `0x1612` | 4 | bytes 1-2 grew from 0 to 9960 during charging — **Charging Elapsed Time** candidate |
| `0x173F`, `0x1740` | 4 each | Coolant valve drive position pair (TBD scale) |
| `0x1F42` | 2 | (variable; on EV ECU = BATT V × 0.001 V; here had different range — TBD) |
| `0x10D7` | 2 | offset-binary zero (constant idle) |
| `0x10DB` | 2 | varied 32768 → 35148 during charging (offset +2380) — **AC Input Current** candidate |
| `0x16A1`, `0x16A2`, `0x1685`, `0x1671` | 2 each | varied during charge — TBD specific assignments |

### F302 (40 sources, 59-byte response body)

| Source DID | Notes |
|---|---|
| `0x10D1` | byte 1 = **Charging flag** (transitioned 1→0 during plug-in event) ✓ |
| `0x10D4`, `0x181D`, `0x1696` | offset-binary near zero (~32768) — likely related currents |
| `0x16A0`, `0x16A3`, `0x1723` | varied substantially during charging — TBD |
| `0x161C`, `0x106E`, `0x1650`, `0x1722`, `0x1718`, `0x1689`, `0x1660` | 1-byte status flags that transitioned during the session |
| `0x1F0D` | Vehicle Speed (cross-broadcast from EV ECU surface) |
| `0x1129`, `0x1738`, `0x1739` | constant during this session — config |

### Direct-poll DIDs catalogued (71 total, polled ~45-47 times each during 7-min capture)

High-confidence matches (from value uniqueness):

| DID | Length | Sample value | Identified parameter |
|---|--:|---|---|
| `0x1666` | 1 | `0xFF` | **HLC Communication Sequence Status = "Unconnected"** (14-state ISO 15118 enum) ✓ |
| `0x1688` | 1 | `0x21` | **Charging History Information = "AC Charging Complete (Full Charge)"** (25-entry enum) ✓ |
| `0x1668` | 1 | `0x00` | **Charger Power Supply Voltage Type = "None"** (0=None, 1=100V, 2=200V) ✓ |
| `0x16AA` | 4 | `ff 00 0e 00` | likely HLC-related (matches `0x1666` "Unconnected" prefix) |

Remaining ~60 direct-poll DIDs returned ambiguous values (`0x00`, `0x01`, `0xFF`) — need single-source isolation to identify.

### Authoritative parameter dictionary

A Techstream `.TSE` Live Data recording of an L2 charge cycle yields the full set of 237 OBC parameter names + units + 125 enum tables (Charger Operation Status, AC Charging Operation Status, Charging History Information, DC Charging Control Status (CCS), HLC Communication Sequence Status, Charging Lid Opening and Closing Status, Charging Connector Lock Pin Status, etc.). Capturing during a real charge session is the practical way to enumerate the charging state machines in their authoritative form.
