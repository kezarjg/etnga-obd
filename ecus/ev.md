---
name: "EV ECU (Toyota official: Hybrid Vehicle Control ECU, F45)"
diagnostic_request_id: 0x7D2
diagnostic_response_id: 0x7DA
toyota_name: Hybrid Vehicle Control ECU
em_reference: F45
physical_bus: P-CAN-FD          # per EM39J0U/system/MPX_P_*.pdf
isotp: standard            # standard ISO-TP, no extended-addressing prefix
sessions_observed:
  - default                # 0x01
  - extended               # 0x03 — Techstream switches into this for 0x2C
techstream_parameter_count: 375     # what Techstream's UI reports as available
confidence: high
---

# EV ECU (Toyota: Hybrid Vehicle Control ECU)

Per the Toyota Electrical Manual (EM39J0U), this ECU is officially **F45 — Hybrid Vehicle Control ECU**, sitting on the **P-CAN-FD bus**. Despite the "Hybrid" name, this is the main vehicle supervisor on the all-electric Solterra/bZ4X — the bZ4X platform was derived from Toyota's hybrid architecture and retains the legacy ECU naming.

When "EV ECU" is opened in Techstream, traffic flows on the OBD-II `0x7D2/0x7DA` ID pair (proxied through the F38 Network Gateway from P-CAN-FD).

## Diagnostics

| Aspect | Value |
|---|---|
| Request ID | `0x7D2` (standard 11-bit) |
| Response ID | `0x7DA` (request + 8) |
| Transport | Standard ISO-TP — no extended-addressing prefix (unlike `0x750`) |
| Confirmed services | `0x10` SessionControl, `0x22` ReadDataByIdentifier, `0x2C` DynamicallyDefineDataIdentifier, `0x31` RoutineControl, `0x3E` TesterPresent |
| Sessions observed | default (`0x01`), extended (`0x03` — required for `0x2C`) |
| Session timing returned | P2 = 50 ms, P2* = 500 ms |
| Techstream UI parameter count | **375** Data List items available |

## Data List uses dynamic DIDs — read this first

Techstream does NOT poll the EV ECU's underlying telemetry DIDs directly. It uses ISO 14229-1 service `0x2C` (`DynamicallyDefineDataIdentifier`) to compose ad-hoc bundles into two slots — `0xF301` and `0xF302` — and then polls those slots via `0x22`.

When you change the Data List selection in Techstream, this exact sequence runs on the wire (every time):

```
3E 00            TesterPresent
10 01            DiagnosticSessionControl → defaultSession
10 03            DiagnosticSessionControl → extendedDiagnosticSession
31 03 10 02      RoutineControl → requestRoutineResults, routine 0x1002 (preparation, purpose unknown)
2C 03 F3 01      Clear dynamic slot F301
2C 03 F3 02      Clear dynamic slot F302
2C 01 F3 01 …    Define F301 (multi-frame; lists source DIDs to compose)
2C 01 F3 02 …    Define F302 (multi-frame; second batch)
22 F3 01         Begin polling slot 1
22 F3 02         Begin polling slot 2
```

The `2C 01 F3 0X` define payload is structured as `2C 01 [slot_high] [slot_low]` followed by repeated 4-byte specs `[src_did_high] [src_did_low] [position 1-indexed] [length]`. Each spec means "into the next position of the slot's response, copy `length` bytes starting at byte `position` of source DID `src_did`."

**Practical consequence:** when looking at a capture, you can't decode an `F301` poll response without also finding the most recent `2C 01 F3 01 …` define request — that define tells you which source DIDs and what offsets the response bytes correspond to.

To identify the source DID for a single Techstream Data List item: enable only that item, capture, and the resulting `2C 01 F3 01` define will list exactly one source spec.

## Source DIDs identified

| Techstream label / role | Source DID | Position | Length | Observed value | Notes |
|---|--:|--:|--:|---|---|
| **Vehicle Speed** | `0x1F0D` | 1 | 1 | `0x00` parked | **1 LSB = 1 km/h** (confirmed via ABRP, max 210 km/h). |
| Vehicle Speed when DC Quick Charging Connector Connect | `0x10E4` | 1 | 2 | `0x8000` | Sentinel "no recorded value" — car has presumably never been DC fast-charged while moving. |
| **Parked flag** (within gear/state register) | `0x1061` | byte 1 (B) | bit 9 of B | — | ABRP reads `bit 9 of byte B` as parked flag. Full P/R/N/D enum elsewhere in this DID's bits — TBD. |
| **HVAC power draw** | `0x106E` | 1 | 1 | — | ABRP equation `A / 20` → kW. Indicates climate-system instantaneous power. Useful as a heating/cooling activity proxy. |

## Source DID inventory from the "all items" Data List

Reconstructed from the 164-byte multi-frame `2C 01 F3 01` define captured at session timestamp `1777852925.549`. This is the set of source DIDs that comprise dynamic slot F301 when Techstream's Data List has its full set of items enabled. (F302 contains a second batch — extracted similarly when reviewing the same window.)

40 source DIDs in F301, total 226 bytes when polled:

| Source DID | Position | Length | DID | Position | Length | DID | Position | Length | DID | Position | Length |
|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|
| `0x15EA` | 1 | 35 | `0x1410` | 1 | 23 | `0x15E8` | 1 | 17 | `0x1006` | 1 | 13 |
| `0x15E9` | 1 | 9  | `0x10A1` | 1 | 9  | `0x10A2` | 1 | 9  | `0x1007` | 1 | 8  |
| `0x1075` | 1 | 8  | `0x145D` | 1 | 5  | `0x1F9A` | 2 | 5  | `0x107D` | 1 | 4  |
| `0x1807` | 1 | 4  | `0x0103` | 1 | 4  | `0x110E` | 1 | 4  | `0x1069` | 1 | 4  |
| `0x106A` | 1 | 4  | `0x10B7` | 1 | 4  | `0x110B` | 1 | 4  | `0x10B8` | 1 | 4  |
| `0x1039` | 1 | 4  | `0x106C` | 1 | 3  | `0x10A6` | 1 | 3  | `0x1066` | 1 | 3  |
| `0x1462` | 1 | 3  | `0x15EF` | 1 | 3  | `0x1449` | 1 | 3  | `0x10A5` | 1 | 3  |
| `0x1F42` | 1 | 2  | `0x10B5` | 1 | 2  | `0x10B1` | 1 | 2  | `0x1F31` | 1 | 2  |
| `0x1067` | 1 | 2  | `0x10D6` | 1 | 2  | `0x111E` | 1 | 2  | `0x1F21` | 1 | 2  |
| `0x1078` | 1 | 2  | `0x1112` | 1 | 2  | `0x2813` | 1 | 2  | `0x1061` | 1 | 2  |

**Observation:** lengths are biggest at the top (35, 23, 17, 13 bytes) and shrink. The biggest source DIDs probably contain "panels" of related signals — e.g., `0x15EA` at 35 bytes might be a battery-pack-state struct holding voltage, current, temperatures, SOC, and several status flags, all packed contiguously.

## Heuristic DID range structure (subject to revision)

Observed groupings in the supported source DID space — useful for guessing what category an unknown DID belongs to:

| Range | Likely category (Toyota/Subaru convention; not yet confirmed) |
|---|---|
| `0x010X` | Vehicle / system status |
| `0x10XX–0x11XX` | Battery cell / pack electrical parameters (large structs in the high-counts part of the inventory live here) |
| `0x14XX–0x15XX` | Motor / inverter / temperatures / sensor groups |
| `0x16XX–0x1FXX` | Diagnostic info, freeze-frame, fault data, vehicle-state snapshots |
| `0x20XX–0x2BXX` | Various subsystems |
| `0x81XX, 0xB1XX, 0xDA01, 0xE801` | System / version / supplier info |
| `0xF1XX` | ISO 14229 standard (`F186` ActiveSession, `F190` VIN) |
| `0xF301, 0xF302` | **Dynamic DID slots — not stable signals** |

## Decoding plan going forward

1. **Per-item isolation captures**: enable one Data List item at a time in Techstream → capture the `2C 01 F3 01` define → record the source-DID + position + length triple → record the parameter label off the Techstream screen. Each session unlocks 1 item.
2. **Range bombing**: enumerate source DIDs by polling them directly (without going through Techstream). Doable from any tester host with `cansend` once active probing is acceptable.
3. **Big-payload decomposition**: source DIDs like `0x15EA` (35 bytes) probably hold many signals. With a full poll response captured, observed bytes can be correlated with values displayed in Techstream when those packed items are isolated.

## Functional roles (per Data List inventory, 2026-05-08 snapshot)

The full Data List was inspected on 2026-05-08 in Ready mode. The ECU is the **vehicle supervisor** — it owns or coordinates a much broader scope than just propulsion. Major functional groups:

### Propulsion (front + rear motors — AWD)

| Aspect | Front | Rear |
|---|---|---|
| Motor Revolution | ✅ | ✅ |
| Target Motor Torque | ✅ | ✅ |
| Motor Torque (actual) | ✅ | ✅ |
| Request Motor Regenerative Brake Torque | ✅ | ✅ |
| Motor Regenerate Brake Execution Torque | ✅ | ✅ |
| Motor Inverter Temperature | ✅ | ✅ |
| Motor Inverter Temperature just after IG ON | ✅ | ✅ |
| Motor Inverter Maximum Temperature | ✅ | ✅ |
| V Phase / W Phase Motor Current | ✅ | ✅ |
| Motor Carrier Frequency | ✅ | ✅ |
| Motor Control Mode (e.g. "Sine Wave") | ✅ | ✅ |
| Motor Inverter Operation Request | ✅ | ✅ |
| Motor Inverter Shutdown Status | ✅ | ✅ |
| Motor Emergency Shutdown (Sub/Main CPU) | ✅ | ✅ |

Notable AWD coordination behavior at idle: **`Rear Motor Inverter Operation Request = Shutdown`** and **`Rear Motor Inverter Shutdown Status = Shutdown`** — the rear inverter is parked off when not under load, conserving energy. Front inverter stays "Output Torque / Awake".

Also includes:
- "Rear Motor Torque Ratio" — front/rear torque distribution in real time
- "VL-Voltage before Boosting" / "VH-Voltage after Boosting" — DC bus monitoring (no actual boost converter; values match each other on Solterra)
- "Voltage Deviation between before Boosting and after Boosting during SMR Precharge" — diagnostic for the SMR pre-charge sequence

### Vehicle dynamics

- Vehicle Speed (and `SP1 Vehicle Speed` separately)
- Wheel Speeds × 4 (FR, FL, RR, RL) — useful for speed validation across sources
- Steering Angle (deg)
- Forward and Rearward G, Lateral G (m/s²)
- Yaw Rate Value (deg/s)
- Accelerator Position (%) plus the two raw sensor voltages (No.1 / No.2 — dual-redundant Hall sensors)
- Master Cylinder Control Torque, Brake Cancel Switch
- Stop Light Switch, Door Open Switch Status
- Grille Shutter Position (%) + Stuck/Slipping/Control Mode flags — active aero control

### Drive mode and chassis selection

- Shift Position + Shift Position (Meter), N/P Switch Status, Sports Shift Position
- 1 Pedal Mode, 1 Pedal Switch
- AWD Input Switch, AWD Mode Status (NORMAL / ...)
- Off-road Speed Up/Down Switch
- Drive Mode (e.g. "HV Mode" — note: Toyota retains the legacy enum even on BEV)
- Drive Mode Select Status, Powertrain Drive Mode Switch
- VSC/TRC OFF Switch, AC100V Accessory Outlet Switch, TC Terminal

### Power coordination — HV state

- IGB Signal Status, IGB Keeping Status, IG2 Signal Status, MRL2 Signal Status
- IGR, IGP Signal Status, IGR Signal Status
- HV/EV Activate Condition (Normal / ...)
- MG Activate Condition
- SMRG/SMRB/SMRP Status + Control Status (the contactor state machine — same parameters as on the EV Battery ECU at `0x747`, suggests cross-broadcast or duplicate read)
- WIN/WOUT Control Limit Power (cross-checked with EV Battery ECU)

### DC/DC Converter — main HV→12V (distinct from the OBC's charging-side converter)

This ECU runs a **main DC/DC converter** that's separate from the "Sub DC/DC Converter (for Charging)" on `0x745`. While Ready, this is what supplies the 12V loads. Parameters:

- DC/DC Converter Output Voltage Control Mode (e.g. "Normal")
- DC/DC Converter Drive Request, Activate Condition, Operation Status Notification
- Target DC/DC Converter Voltage (e.g. 13.4 V), Output Voltage (Low Side / High Side), Output Current
- Over Temperature Protection, Stopping, Drooping, Unavailable Status flags
- Voltage Sensor (High Voltage Side) Unavailable Status
- DC/DC Converter Diagnosis Status, CAN Unreceivable Status

Confirmed at idle: Output Current 25.0 A, Output Voltage Low 13.35 V, High 393.31 V, target 13.4 V. **This ECU is the authoritative source for 12V auxiliary voltage and current — more authoritative than the cluster's `+B Voltage`.**

### Inverter / motor cooling subsystem (separate loop from battery cooling)

- Inverter Coolant Water Temperature (separate from battery coolant loop)
- Inverter Water Pump (status, duty ratio %, revolution rpm)
- Radiator Fan (% duty)
- Coolant Distribution Valve target/actual position (deg) — confirms Solterra has a routing valve between the inverter and battery loops
- Coolant Distribution Valve protection flags (voltage out of range, temperature out of range)
- Coolant Distribution Valve Initialize Status
- Hybrid/EV Battery Water Pump Speed/Duty/Drive Status — duplicated here from the EV Battery ECU (cross-broadcast)

### Battery state — cross-broadcast from EV Battery ECU

This ECU sees and exposes EV Battery data via `0x1F` range source DIDs that read shared values. Includes:

- Hybrid/EV Battery SOC (matches `0x747` `0x1F5B` value), SOC Min/Max/Just-after-IG-ON
- HV Battery Voltage (matches `0x747` `0x1F9A` bytes 3-4), Current (bytes 5-6), Current for Driving Control
- Battery Max/Min Temperature
- Battery Cooling Necessity before Charging (BMS internal flag — directly relevant to preconditioning)
- Battery Charging and Discharging Permission Status with Hybrid/EV Battery Thermal Keep
- Insulation Resistance Division Check Completion (multiple sub-flags by component)

### **12V auxiliary battery — comprehensive lifetime telemetry**

This is the single biggest novel chunk vs. what the cluster exposes. The EV ECU runs a full 12V battery health analytics suite:

| Parameter | Sample value (idle, Ready) | Use |
|---|---|---|
| Auxiliary Battery Voltage | 13.58 V | 12V aux voltage (more authoritative than cluster) |
| Auxiliary Battery Current | 0.61 A | 12V aux current |
| Smoothed Value of Auxiliary Battery Temperature | 64.9 °F | 12V aux temperature |
| Auxiliary Battery Voltage just before SMR Precharge | 10.99 V | precharge dip — useful health proxy |
| Auxiliary Battery Charging Integrated Current | 2128.1 Ah | lifetime In counter |
| Auxiliary Battery Discharging Integrated Current | 121.4 Ah | lifetime Out counter |
| Auxiliary Battery Capacity after IG ON / OFF | 1 Ah / 1 Ah | per-cycle delta |
| Auxiliary Battery Status of Full Charge | 29.0 Ah | derived 12V capacity (≈ rated capacity) — **direct SoH proxy** |
| Auxiliary Battery Charging Rate Accuracy | "High" | meta-confidence in SoC computation |
| Auxiliary Battery Voltage Low Times | 23 | lifetime low-voltage event count |
| Auxiliary Battery Voltage at Low Voltage Checking Initiation | 9.94 V | latest event voltage |
| Integrated Ready ON Time | 126 hour | drive-on lifetime |
| Number of Long Term Leaving with IG OFF | 0 | sleep events counted |
| Auxiliary Battery Integrated Thermal Load | 914760 | thermal stress accumulator (proprietary unit) |
| Auxiliary Battery Average Current during IG OFF (1st…5th trip before) | -0.022 to -0.125 A | sleep-current per recent trip |
| Total Distance Up to (1st…5th) Trip before | 24850/24843/24843/24840/24836 | rolling odometer history (synthetic — real readings redacted) |
| IG ON Time / Ready ON Time (1st…5th trip) | 16/272/6/8/28 min | per-trip duration history |

**This is comprehensive 12V battery health data already computed by the car** — any client can expose it directly without re-deriving. Good "battery health" UI page material.

### Gear Shift Control Module (GSCM) — sub-ECU integrated here

The EV ECU exposes a complete GSCM interface (the shift-by-wire actuator subsystem), including:

- Gear Shift Control Module Power Supply Voltage, B-CPU Temperature
- IGP/IG Status (GSCM A and B), WAKE Signal Status
- Backup Power Supply Type (Capacitor Type), Backup Signal Status, Backup Request
- Fail Safe Status (multiple variants)
- Shift Sensor 1/2/3 Status (H/L), Absolute Angle Sensor Value 1/2
- Gear Shift Actuator Power Supply Voltages (MA1, MA2), Motor Angle Sensor Value, Motor Speed
- Parking Lock Motor U/V/W Phase Current-Carrying Status + Terminal Currents
- ACT Relay status, ACT Position Status (e.g. "Shift in P"), ACT Operation Status, ACT Function Informing
- Not P Position Learning Value (Output Side / Motor Side)

### Anomaly counters (lifetime trigger counters for unusual driving events)

Toyota records ~30+ "Trigger Counters" for irregular shift/driving events. Examples:

- "Shift Operation without Depressing Brake from Shift Position P Trigger Counter" = 164
- "Shift Operation when Auxiliary Battery Voltage Low Trigger Counter" = 4
- "Shift P Operation when Auxiliary Battery Low Voltage Trigger Counter" = 18
- "Shift Operation during Ready Indicator Blinking Trigger Counter" = 8
- "Auto Change to Shift Position P when Driver Get Out Trigger Counter" = 5
- "Shift R/D Operation Rejection from Shift Position N during Accelerator Pedal Depress Trigger Counter" = 0

Useful as a fingerprint of driving style. Probably persistent across the vehicle's lifetime. Could feed a "driving habits" UI page.

### Charging coordination signals (forwarded from `0x745`)

The EV ECU sees and forwards AC/DC charging relay statuses, permission signals, and the charge-during-Ready interlock. Most overlap with `0x745` Plug-in Charge Control content.

### Vehicle specification info (factory-fitted options)

- Suspension Control Module / IGS / Advanced Park / Solar / Power Steering — each as "Specification Information Switching" (Supported/Not Supported) + "Specification Information" (existence flag) pairs. Lets a client discover which options the car has.

## Open questions

- **Range estimate** — searched for "Cruising Distance" / "Distance to Empty" / "Range" parameters; **none present in the EV ECU Data List** (also confirmed absent in Cluster `0x7C0` and EV Battery `0x747`). Range estimate is internal to the cluster and not exposed via diagnostic surface on any ECU. Must be derived externally or sniffed from a CAN broadcast frame during driving.
- **Why "Drive Mode = HV Mode" on a BEV** — Toyota's enum was carried over from the hybrid platform. The bZ4X is on eTNGA shared with PHEVs; the BEV variant probably reports `HV Mode` as a stand-in for "powered by the HV traction battery". Just a quirk to note.
- **Motor torque sign convention** — at idle, "Motor Torque = -0.13 Nm" (slightly negative). Likely sign convention is +ve = drive, −ve = regen. Confirm during a drive cycle.
- **Single vs dual DC/DC architecture** — this ECU has the main DC/DC; `0x745` has the "Sub DC/DC Converter for Charging". Solterra has two physical DC/DC modules? Or two driver paths to one module? Worth checking against Toyota EM wiring diagrams.
- **Anomaly counters as DID surface** — these are clearly stored persistently. Reading the underlying source DIDs may reveal a "factory reset" routine ID that resets them. Useful or dangerous (clearing drives diagnostics).

## Source DID inventory cross-reference

The 2026-05-08 EV Battery snapshot confirmed many of the source DIDs used by F302 (the second dynamic slot) overlap with EV ECU's `0x1F` range:

- `0x1F0D` VehicleSpeed (1 byte) — confirmed shared with EV Battery
- `0x1F31` (2 bytes), `0x1F42` (2 bytes), `0x1F21` (2 bytes) — likely vehicle-state shared registers, same content visible from both ECUs
- `0x1F9A` (2 bytes from position 2) — pack V/I cross-read

This is consistent with the **eTNGA cross-broadcast model**: `0x1FXX` is the shared "vehicle state" namespace, and any P-CAN-FD ECU can read most of those DIDs.

## Source DID layout — fully decoded 2026-05-09

The 2026-05-09 PID-mapping session captured the full F301/F302 dynamic-DID definitions and decoded most source DIDs by single-source isolation, drive-cycle cross-correlation, and physics regression. Authoritative table:

### F301 (40 sources, 226-byte response body)

| Source DID | Bytes | Decoded purpose |
|---|--:|---|
| `0x15EA` | 35 | Aux Battery 5-trip history bundle: 5×{2-B avg current offset-binary ×0.001 A, 2-B distance, 1-B IG-OFF days, 1-B IG-ON ×2 min, 1-B Ready-ON ×2 min} |
| `0x1410` | 23 | unknown status block |
| `0x15E8` | 17 | Aux Battery cluster: charging integrated current u32×0.1 Ah (1-4), discharging integrated u32×0.1 Ah (5-8), capacity after IG-ON/OFF bias-128 Ah (9, 10), Integrated Ready ON Time u16 hours (11-12), Number of Long-Term Leaving u8 (13), Aux Bat Integrated Thermal Load u32 (14-17) |
| `0x1006` | 13 | constant during drive — config/calibration |
| `0x15E9` | 9 | 3 × {Total Distance after long-term leaving u16 mile, Time u8 day} |
| `0x10A1` | 9 | **Front motor block**: Revolution u16 BE offset-binary × 1 RPM/LSB (1-2), Target Torque u16 BE offset-binary (3-4), bytes 5-6 (state flags, motor inverter request/shutdown/fail), Actual/Regen Torque u16 BE offset-binary × ~0.125 Nm/LSB (7-8), status byte (9) |
| `0x10A2` | 9 | **Rear motor block**: same layout as `0x10A1` |
| `0x1007` | 8 | byte 2 = small enum (7-10), bytes 3-4 = fast counters; byte 1 + 5-8 constant zero |
| `0x1075` | 8 | **4 × wheel speeds (FR/FL/RR/RL)**: u16 BE offset-binary × 0.01 km/h ✓ |
| `0x10B7` | 4 | **Front motor V/W phase currents**: 2 × u16 BE offset-binary × 0.1 A |
| `0x10B8` | 4 | **Rear motor V/W phase currents**: same encoding |
| `0x1039` | 4 | bytes 3-4 = Inverter Water Pump Revolution u16 BE rpm (=4242 idle); bytes 1-2 = related (TBD) |
| `0x110E` | 4 | byte 2 = HV Battery Water Pump Speed u8 rpm |
| `0x0103` | 4 | **Odometer + unit**: byte 1 = unit (0x02 = mile), bytes 2-4 = u24 BE odometer |
| `0x1807` | 4 | u32 — likely cumulative timer/counter |
| `0x107D`, `0x1069`, `0x106A`, `0x110B`, `0x1F42` | 4, 4, 4, 4, 2 | mostly zeros / sentinels in the captured state. `0x1F42` = **BATT Voltage** u16 BE × 0.001 V |
| `0x10B5`, `0x10B1` | 2, 2 | **Accelerator Position Sensor No.1 / No.2 voltage %** (paired, varied during drive) |
| `0x1118` | (in F302) | **Steering Angle**: (raw − `0x8000`) × 1.25 deg ✓ |
| `0x1061` | 2 | **byte 1 = Shift Position** (P=0, R=2, N=4, D=6, B=8) ✓ ; byte 2 = related shift flag |
| `0x10D6` | 2 | candidate Aux Battery Current (snapshot-time value matched) |
| (15+ more sources) | various | some constants, some sentinels, some still-pending labels |

### F302 (40 sources, 72-byte response body)

| Source DID | Notes |
|---|---|
| `0x10E4`, `0x107F`, `0x181D`, `0x1609`, `0x1829`, `0x182A` | mostly 2-byte ADC channels; some show small drift (likely temperatures, currents) |
| `0x10CF` | varied during drive (likely a current/voltage reading) |
| `0x110F`, `0x1110` | always 4 — possibly mode-config registers |
| `0x15EE` | **Aux Battery Voltage**: u16 BE × 5/4096 V (≈ 1.221 mV/LSB) ✓ |
| `0x107E` | **Aux Battery Voltage just before SMR Precharge**: same scale × 5/4096 V ✓ |
| `0x15F7`, `0x15F8` | unknown — varied during drive |
| `0x1090`, `0x1091`, `0x1092`, `0x1093` | **G-sensor cluster**: Forward G, Lateral G, Yaw Rate, +1 dynamics signal — all u16 BE offset-binary; scales likely 0.001 m/s² and 0.01 deg/s |
| `0x110A`, `0x1118`, `0x1144`, `0x1091`, `0x140F`, `0x2818`, `0x1092` | mixed; `0x1118` = Steering Angle, `0x140F`/`0x1440` = additional shift-position-correlated flags |
| `0x10AE` | **byte 2 = Hybrid/EV Control System Control Mode** enum (0=None, 1=Driving, 2=External Power Supply, 3=Charging, 4=Other) ✓ |
| `0x104B` | likely related drive-mode enum |
| `0x1F65`, `0x1062`, `0x1032`, `0x1119`, `0x10AD`, `0x1F1C` | 1-byte status / enum bytes |

### Authoritative parameter dictionary

A Techstream `.TSE` Live Data recording of a drive cycle yields the full set of 397 EV-ECU parameter names + units + 182 enum tables. Capturing during a real drive is the practical way to enumerate the parameter dictionary; the per-tick block format inside `.TSE` files is parseable as a tick-stream of fixed-size frames separated by sync markers.
