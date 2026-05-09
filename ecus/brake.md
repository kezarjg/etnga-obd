---
name: Brake System (Brake/EPB ECU + Brake Booster Actuator)
diagnostic_request_id_main: 0x7B0
diagnostic_response_id_main: 0x7B8
diagnostic_request_id_booster: 0x750
diagnostic_response_id_booster: 0x758
booster_gateway_sub_target: 0x29
isotp: standard for 0x7B0; mixed-addressing for 0x750/0x29
sessions_observed:
  - default                # 0x01
sources:
  - 2026-05-08_1955_ecu-mapping-marathon
  - 2026-05-08 Brake/EPB Data List snapshot (Techstream, Ready mode)
  - 2026-05-08 Brake Booster Data List snapshot (Techstream, Ready mode)
confidence: high
---

# Brake System

Two related ECUs:

1. **Brake/EPB ECU (`0x7B0/0x7B8`)** — the main ABS/VSC/TRC/EPB controller. Owns wheel-speed sensing, dynamics sensors (yaw, G), all the active stability control, the Electric Parking Brake actuators (RH and LH), brake hold, and the regen-cooperation interface.
2. **Brake Booster (`0x750` sub-target `0x29`)** — gateway-routed; the **brake-by-wire pump unit**. EVs don't have engine vacuum, so a brushless-motor pump generates servo pressure for the calipers. Has its own inverter, thermistors, and capacitor-backup for failsafe.

These two work together: pedal stroke → Brake Booster generates servo pressure → Brake/EPB ECU modulates per-wheel via ABS solenoids and coordinates with regenerative braking from the EV ECU.

## Brake/EPB ECU (`0x7B0`)

### Diagnostics

| Aspect | Value |
|---|---|
| Request ID | `0x7B0` (standard 11-bit) |
| Response ID | `0x7B8` (request + 8) |
| Transport | Standard ISO-TP |
| Confirmed services | `0x10`, `0x19`, `0x22`, `0x3E` |

### Functional content (per Data List inventory, 2026-05-08)

#### Wheel speeds + per-wheel accelerations

| Parameter | Sample (parked) | Notes |
|---|---|---|
| FR/FL/RR/RL Wheel Speed | 0.0 MPH each | per-wheel speed sensors |
| FR/FL/RR/RL Wheel Acceleration | 0.000 m/s² each | derived from wheel speed |
| FR/FL/RR/RL Wheel direction | "Forward" each | rotation direction sensing |
| Per-wheel speed-sensor open / signal status | Normal | diagnostic flags |

#### Vehicle dynamics — high-resolution sensors (with zero-point calibration)

| Parameter | Sample | Notes |
|---|---|---|
| Master Cylinder Sensor 1 | -0.02 MPa | brake pedal hydraulic pressure |
| Zero Point of M/C | -2.4 MPa | learned offset |
| M/C Sensor Grade | 0 MPa/s | rate of change |
| Master Cylinder Sensor Temperature | 73 °F | sensor self-temp |
| Lateral G | 0.000 m/s² | also at "Higher Resolution" via Yaw Rate Sensor 1 / 2 |
| Forward and Rearward G | 0.000 m/s² |  |
| Yaw Rate Sensor Value | 0 deg/s |  |
| Yaw Rate Sensor 1/2 Higher Resolution Signal | -0.11/-0.12 deg/sec | redundant sensor pair |
| GL1 GX / GL2 GY Sensor Higher Resolution | 3 / 5 mG | redundant accelerometer pair |
| Steering Angle Value | 19.4 deg | steering wheel angle (cross-broadcast) |
| Zero Point of Steering Angle | 3.0 deg | learned offset |
| Stroke Sensor2 | 4.0 V | pedal stroke sensor #2 |
| Quantity of Brake Pedal Stroke | 0 inch | derived stroke distance |
| Brake Pedal Stroke Change Speed | 0 mm/s | derived rate |

#### Active stability and traction control

| Parameter | Sample | Notes |
|---|---|---|
| TRC(TRAC)/VSC OFF Mode | Normal mode (TRC ON/VSC ON) | user setting |
| Brake Hold Control Mode | Out of control mode | not engaged |
| TRC Control / Engine Control / Brake Control | Out of controlling | currently inactive |
| FR/FL/RR/RL Wheel VSC Ctrl Status | Out of controlling | per-wheel |
| FR/FL/RR/RL Wheel ABS Ctrl Status | Out of controlling | per-wheel |
| BA / PBA Ctrl Status | OFF / OFF | brake assist statuses |
| FR/FL/RR/RL Target Oil Pressure | 0.0 MPa each | requested per-corner |

#### Per-wheel ABS solenoids (8 solenoids: SR/SF × LR/LH × RR/RH)

| Parameter | Sample |
|---|---|
| ABS Solenoid (SRLR/SRLH/SRRR/SRRH) | OFF each |
| ABS Solenoid (SFLR/SFLH/SFRR/SFRH) | OFF each |
| TRC/VSC Solenoid (SM1/SM2) | OFF each |
| ABS Motor Relay | OFF |
| Solenoid Relay | ON |

#### Electric Parking Brake (EPB) — independent RH and LH actuators

| Parameter | Sample (RH/LH) | Notes |
|---|---|---|
| Actuator Status | Park Applied / Park Applied | currently engaged (we're parked) |
| Motor Input Voltage | 13.6 V / 13.6 V | each actuator's supply |
| Motor +Terminal / -Terminal Voltage | 3.9 V / 3.9 V each | balanced (both terminals at the same voltage = motor not driven) |
| Motor Actual Current | -0.027 A / -0.027 A | quiescent |
| Motor Adjustment Current | -0.054 A / 0.108 A | calibration offset |
| Motor Current Differential | 0.000 A / -0.063 A | differential measurement |
| Motor Driver Operation Status | OFF / OFF | not actively moving |
| Motor Current High flag | OFF / OFF | over-current fault |
| Motor Relay 1/2/3/4 | OFF each | H-bridge relays for direction |
| EPB Switch | Neutral | console button position |
| EPB Warning Light | OFF | dash indicator |
| Auto Mode | ON | auto-engage on park |
| Auto Mode Request | "Auto mode request" | currently auto-engaging |
| EPB Lock Request | "Not requested" | no manual lock |
| Brake Hold Ready | "Not in stand-by mode" | brake hold is a separate feature |
| Dynamic PKB Mode | OFF | "dynamic" parking-brake function (used for moving-vehicle emergency hold) |
| EPB Control Cancel History | 7 | times cancelled |
| Counter of IG ON After EPB Control Cancel | 255 | rolling counter |
| RH/LH Actuator Current Status | Normal / Normal | health flag |
| Permission of Interlocking Shift / Brake / RH PKB Lock / RH PKB Release / RH Dynamic PKB / RH PKB Full Release / LH ... | "Available" each | per-action permission flags |

#### Regenerative cooperation (the EV ECU ↔ Brake handoff)

| Parameter | Sample | Notes |
|---|---|---|
| **Regenerative Cooperation** | OFF | currently parked, no regen |
| **FR Regenerative Request** | 0 Nm | torque the brake ECU is asking from regen |
| **FR Regenerative Operation** | 0 Nm | torque actually delivered |
| **Fade Status** | OFF | overheating brake fade flag (would be useful in towing/mountain driving) |

The Brake/EPB ECU is the **regen-blending coordinator** — when the driver presses the brake pedal, this ECU decides how much torque to request from regen vs. how much hydraulic pressure to apply.

#### Toyota Safety Sense (TSS) and Vehicle Motion Control (VMC) integration

| Parameter | Sample | Notes |
|---|---|---|
| Request Acceleration of Upper Limit from Toyota Safety Sense | 32.767 m/s² | sentinel "no limit" |
| Request Acceleration of Lower Limit from Toyota Safety Sense | -0.071 m/s² | very small braking request from TSS |
| Target Acceleration of Upper Limit from Vehicle Motion Control | 12.28 m/s² | from VMC arbitration |
| Target Acceleration of Lower Limit from Vehicle Motion Control | 20.40 m/s² | unusual sentinel — interpretation TBD |
| Target Driving Force of Upper / Lower Limit from VMC | 65534 / 0 N | sentinel max + zero |
| ADS Control EPS Pinion Angle2 | 0.40850 rad | from automatic-driving-support (steering angle target) |

#### Calibration learning statuses

| Parameter | Status |
|---|---|
| Zero Point of G Sensor Learning Status | Complete |
| Zero Point of Yaw Rate Sensor Learning Status | Complete |
| Linear Solenoid Valve Offset Learning Status | Complete |
| Zero Point of Stroke Sensor Learning Status | Complete |
| System Variant Learning Status | Complete |

#### Counters and history

| Parameter | Sample |
|---|---|
| Vehicle Stop Time from IG ON | 1275 s |
| Travel Distance from IG ON | 0 s |
| EPB Control Cancel History | 7 |

#### Open-circuit diagnostic flags (all "Normal" at idle)

Many "Open" flags for sensor health: M/C Pressure Sensor, Stroke, Yaw Rate, Steering, FR/FL/RR/RL Speed, Solenoid Power, Motor Power, A/C ECU Communication, Air Bag ECU Communication, HV Communication, Body ECU Communication, etc. All "Normal" — the ECU's diagnostic surface for sensor and bus health.

## Brake Booster (`0x750/0x29`)

### Diagnostics

| Aspect | Value |
|---|---|
| Request ID | `0x750` with sub-target byte `0x29` |
| Response ID | `0x758` with sub-target byte `0x29` echoed |
| Transport | ISO-TP mixed-addressing (strip first byte before applying ISO-TP rules) |
| Confirmed services | `0x10`, `0x19`, `0x22`, `0x3E` |

### Functional content (per Data List, 2026-05-08)

This is the **brake-by-wire pump assembly** — a brushless DC motor that pressurizes a hydraulic accumulator to provide servo pressure for the calipers. EVs don't have engine vacuum, so the brake booster is electric.

#### Hydraulic state

| Parameter | Sample | Notes |
|---|---|---|
| Servo Pressure | 0.00 MPa | currently at rest |
| Target Oil Pressure | 0.00 MPa | requested |
| Stroke Sensor | 0.9 V | brake pedal stroke sensor (raw V) |
| Voltage of Stroke Sensor | 0.901 V | same parameter, different formatting |
| Quantity of Brake Pedal Stroke | 0 inch | derived stroke distance |
| Brake Pedal Stroke Change Speed | 0 mm/s | derived rate |
| Gap Hold Chamber Oil Pressure | 0.00 MPa | secondary chamber |
| Gap Hold Chamber Oil Pressure Grade | 0 MPa/s |  |

#### Solenoids and relays

| Parameter | Sample |
|---|---|
| ECB Solenoid (SGH) | ON |
| ECB Solenoid (SSA) | ON |
| ECB Main Relay | ON |
| Solenoid Relay (downstream voltage) | 13.5 V |
| SGH Solenoid Current | 0.529 A |
| SSA Solenoid Current | 0.811 A |
| Linear Solenoid (SLM1/SLM2) Current | 0.000 A each |
| Stop Light Relay | OFF |

#### Brushless motor + inverter (the actual booster pump)

| Parameter | Sample | Notes |
|---|---|---|
| Brake Booster Motor | OFF | currently not running |
| Brushless Motor Required Rotation Speed | 0 rpm | request |
| Brushless Motor Actual Rotation Speed | 0 rpm | feedback |
| Brushless Motor Operation Status | Stop | enum |
| Brushless Motor Inverter end Voltage | 13.49 V | input voltage |
| **Thermistor1 Temperature for Inverter Circuit** | 99.68 °F | inverter PCB temp |
| **Thermistor2 Temperature for Inverter Circuit** | 103.53 °F | second sensor — slightly warmer |
| Zero point Calibrated Value of Phase U Current Monitor | 0.408 A | per-phase calibration |
| Zero point Calibrated Value of Phase V Current Monitor | 0.612 A |  |
| Zero point Calibrated Value of Phase W Current Monitor | -0.816 A |  |

#### Capacitor failsafe (the "save your brakes" backup)

The Brake Booster has a capacitor for failsafe operation if the 12V battery fails — it can power one or more brake actuations even with no electrical supply.

| Parameter | Sample | Notes |
|---|---|---|
| The Number of Capacitor Operation | 255 | rolling counter (likely capped at 255) |
| Capacitor Error Detail Code | 0 | health |
| CBKP Voltage | 12.5 V | capacitor voltage |
| BS / BM Voltage | 13.5 / 13.5 V | two main supply rails |
| IGR Voltage | 13.4 V | ignition power |

#### Diagnostic flags

| Parameter | Sample |
|---|---|
| Stop Switch Open | Normal |
| Stroke Open | Normal |
| Servo Pressure Sensor Open | Normal |
| Gap Hold Chamber Pressure Sensor Open | Normal |
| Gap Hold Chamber Pressure Sensor Open History | None |
| Reservoir Warning SW | OFF |

### Notable telemetry from the Brake Booster

Most parameters are subsystem-specific and only useful for diagnostics. Notable exceptions for higher-level vehicle telemetry:

| Metric | Source | Use |
|---|---|---|
| Brake booster inverter temperature | Thermistor1/2 | warning for sustained heavy regen-blending events |
| Brake pedal stroke | Stroke Sensor + Quantity of Brake Pedal Stroke | "is the driver braking" indicator |

## Cross-ECU coordination map

```
Driver brake pedal
      ↓
Stroke sensor (read by both Brake/EPB and Brake Booster)
      ↓
Brake Booster (0x750/0x29): brushless motor pump → servo pressure
      ↓
Brake/EPB ECU (0x7B0): ABS solenoids modulate per-wheel, requests regen torque from EV ECU
      ↓                                                  ↓
Hydraulic calipers (4)                              EV ECU (0x7D2): commands motor regen
                                                          ↓
                                                   Front + Rear inverters (0x724, 0x705) → motor regen torque
```

For the regen-cooperation handoff to work, all four ECUs are involved every brake event.
