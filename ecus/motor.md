---
name: Motor Generator ECUs (Front + Rear inverters)
diagnostic_request_id_front: 0x724
diagnostic_response_id_front: 0x72C
diagnostic_request_id_rear: 0x705
diagnostic_response_id_rear: 0x70D
toyota_name_front: Motor Generator
toyota_name_rear: Rear Motor Generator
em_reference_front: D9 Front EV Motor Control Inverter
em_reference_rear: L4 Rear Transaxle with Motor & Inverter
physical_bus: P-CAN-FD
isotp: standard
sessions_observed:
  - default                # 0x01
sources:
  - 2026-05-08_1955_ecu-mapping-marathon
  - 2026-05-08 Motor Generator + Rear Motor Generator Data List snapshots (Techstream, Ready mode, parked)
confidence: high
---

# Motor Generator ECUs (Front + Rear)

Two physically separate ECUs, one per axle:
- **Front (`0x724/0x72C`)** = Toyota EM "D9 Front EV Motor Control Inverter"
- **Rear (`0x705/0x70D`)** = Toyota EM "L4 Rear Transaxle with Motor & Inverter" (AWD only — confirmed present on this car)

Each is the inverter ECU co-located with its motor — it reads the local resolver, stator temperature, transaxle oil temperature, and the three motor phase currents. The EV ECU (`0x7D2`) supervises both via cross-broadcast.

> **Note**: at idle in Ready mode, the **rear inverter is shut down** (`Rear Inverter Shut Down Status = ON`, `WOUT Control Limit Power = 0.00 kW`). Toyota saves energy by parking the rear inverter when no rear torque is requested. The front stays awake at all times.

## Diagnostics

| Aspect | Front | Rear |
|---|---|---|
| Request ID | `0x724` | `0x705` |
| Response ID | `0x72C` | `0x70D` |
| Transport | Standard ISO-TP | Standard ISO-TP |
| Confirmed services | `0x10`, `0x19`, `0x22`, `0x3E` | `0x10`, `0x19`, `0x22`, `0x3E` |

## Data List structure (both ECUs share an OBD-II compliance shell)

Both ECUs expose the **standard OBD-II emission-monitor framework** even though it's a BEV. Calculate Load, Coolant Temperature, Engine Speed, Intake Air Temperature, Throttle Position — all present, all sentinel zeros / 32 °F. The MIL/DTC/monitor framework is here because OBD-II Mode 01/06/09 compliance has to be served from somewhere. These ECUs answer for the powertrain.

Treat all emission-monitor parameters as **non-meaningful sentinels** — they exist for compliance, not for OVMS.

The real content is the motor/inverter telemetry below.

## Per-axle motor / inverter parameters (Techstream Data List inventory, 2026-05-08)

Same parameter shapes for front and rear, with `Rear` prefix on the rear-specific names.

### Motor / inverter electrical state

| Parameter | Front (sample) | Rear (sample) | Notes |
|---|---|---|---|
| Motor Revolution | 0 rpm | 0 rpm | mechanical RPM |
| Motor Torque | 0.000 Nm | 0.000 Nm | high precision |
| Motor Control Mode | 0 | 0 | enum: 0 = Sine Wave (per EV ECU) |
| Motor Carrier Frequency | 2 | 12 | PWM switching freq (kHz). Asymmetric front/rear. |
| Motor Inverter Shut Down Signal | 4 | 0 | front commanded "limited" mode; rear "no signal" |
| Motor Inverter Shut Down Status | OFF | ON | **rear inverter parked at idle** |
| Motor Inverter Temperature | 79 °F | 79 °F | matches EV ECU |
| Motor Inverter Temperature Duty | 50.0 % | 50.0 % | cooling capacity utilization estimate |
| Motor Inverter High Current (MFINV/RFINV) | OFF | OFF | over-current fault flag |
| Generate Request Torque | 0.000 Nm | 0.000 Nm | torque request (regen) |

### Motor stator + transaxle (NEW signals not exposed elsewhere)

These are **new** — the EV ECU doesn't expose them.

| Parameter | Front | Rear | Notes |
|---|---|---|---|
| **Motor Temperature** | 66 °F | 63 °F | **stator temperature** — direct OVMS `v.m.temp` candidate |
| Motor Temperature Sensor Voltage / Sensor AD Value | 3.63 V | 3.748 V | raw thermistor voltage |
| **Transaxle Oil Temperature** | 66 °F | 61 °F | **gearbox oil temp** — useful for transmission health |
| Transaxle Oil Temperature Sensor Voltage | 2.89 V | 2.97 V | raw |

### Phase currents (3 phase, both lower and higher resolution)

| Parameter | Front (sample) | Rear (sample) | Notes |
|---|---|---|---|
| Motor U Current (Lower Resolution) | 0.3 A | 0.7 A | per-phase instantaneous |
| Motor V Current (Lower Resolution) | -0.5 A | 2.6 A |  |
| Motor W Current (Lower Resolution) | 1.0 A | -1.0 A |  |
| Motor U Current Sensor AD Value (Lower / Higher Resolution) | 2.499 / 2.496 V | 2.502 / 2.498 V | raw ADC voltages |
| Motor V Current Sensor AD Value (Lower Resolution) | 2.504 V | 2.498 V |  |
| Motor W Current Sensor AD Value (Lower / Higher Resolution) | 2.499 / 2.491 V | 2.499 / 2.490 V |  |

The "Higher Resolution" / "Lower Resolution" pair likely corresponds to two different ADC sample modes (faster low-res for control loop, slower higher-res for diagnostics).

### Resolver

| Parameter | Front | Rear |
|---|---|---|
| Motor Resolver Supply Voltage | 1.635 V | 1.652 V |
| Motor Resolver Offset Complete Status | ON | ON |

### Internal ECU power supplies (diagnostic only)

| Parameter | Front | Rear |
|---|---|---|
| Motor ECU Power Supply (For 31V) | 1.640 V | 1.660 V |
| Motor ECU Power Supply (For AD0) | 2.500 V | 2.500 V |
| Motor ECU Power Supply (For 2.5V) | 2.500 V | 2.500 V |

### Cooling — shared between both inverters

These parameters are visible from both ECUs with identical values, suggesting a **single shared inverter coolant loop and a single oil pump** controller (probably driven by the EV ECU `0x7D2`):

| Parameter | Both | Notes |
|---|---|---|
| Inverter Coolant Temperature | 70 °F | matches EV ECU's reading exactly |
| Inverter Water Pump Duty | 54 % | shared loop |
| Inverter Water Pump Revolution | 4100 rpm |  |
| Motor/Generator Cooling Oil Pump Motor Revolution | 0 rpm | shared oil pump (idle) |
| Target Motor/Generator Cooling Oil Pump Motor Duty | 10.0 % |  |
| Motor/Generator Cooling Oil Pump Motor Power Relay Request | OFF |  |

### DC bus voltage — slightly different per axle

| Parameter | ECU | Sample | Notes |
|---|---|---|---|
| VH Voltage | Front | 390.7 V | DC bus at front inverter |
| VLR Voltage | Rear | 390.6 V | DC bus at rear inverter |
| Hybrid/EV Battery System Voltage | Both | 392.0 V | cross-broadcast pack voltage |
| Hybrid/EV Battery System Current | Both | 2.3 A | cross-broadcast pack current |

The 1.3-1.4 V offset between pack voltage and bus voltage is consistent with HV cable resistance / sensor offset.

### Vehicle / power state (cross-broadcast)

Standard cross-broadcast values: Battery Voltage (12V aux), Ambient Temperature, Total Distance Traveled, Vehicle Speed, Accelerator Position, Shift Position, Ready ON Status, SMR Status, Inter Lock Connect Status, Short Wave Highest Value, WIN/WOUT Control Limit Power, Fail Safe Mode, Emergency Shutdown Signal.

## OVMS mappings

### Already covered by EV ECU (no need to re-poll these ECUs for these metrics)

`v.m.rpm`, `v.m.rpm.rear`, `v.m.torque`, `v.m.torque.rear`, `v.i.temp`, `v.i.temp.rear` — same source DIDs cross-broadcast on the EV ECU.

### Newly available from these ECUs (worth polling here)

| OVMS metric | ECU | Parameter | Status |
|---|---|---|---|
| `v.m.temp` | `0x724` (front) | Motor Temperature (stator, °F) | ⬜ DID isolation needed |
| `v.m.temp.rear` | `0x705` | Rear Motor Temperature | ⬜ |
| Custom `v.t.front.oil.temp` | `0x724` | Transaxle Oil Temperature | ⬜ |
| Custom `v.t.rear.oil.temp` | `0x705` | Transaxle Oil Temperature | ⬜ |
| Custom `v.m.front.bus.voltage` | `0x724` | VH Voltage | ⬜ |
| Custom `v.m.rear.bus.voltage` | `0x705` | VLR Voltage | ⬜ |
| Custom: rear-inverter-shutdown flag | `0x705` | Rear Inverter Shut Down Status | ⬜ — useful for AWD-mode awareness |

The **stator temperatures** are the highest-value addition — driver-relevant for "did I push the motor too hard during this drive?" indicators, and useful in long high-load events (e.g., towing or sustained climbing).

The transaxle oil temperatures round out an OVMS "powertrain health" page — useful baseline for fleet diagnostics.

## Open questions

- **Why is rear motor carrier frequency 12 (vs 2 for front)?** Even with the rear inverter shut down, this default value is interesting. Likely just a sleep-state default. May change during active driving.
- **Why does front "Motor Inverter Shut Down Signal = 4" while rear = 0**? Front is awake yet has a non-zero shutdown signal? Could be an enum where 4 = "limited mode" or "ready but not commanded".
- **Are oil pump and inverter coolant pump truly shared?** The values from both ECUs match exactly — suggests yes, both ECUs read from the same source. EV ECU `0x7D2` likely owns these and broadcasts.
- **Phase current sign convention** — at idle, currents are small but non-zero (0.3 / -0.5 / 1.0 A). Likely a balanced 3-phase modulation maintaining alignment without producing torque. Worth verifying during a drive cycle for OVMS phase-current display.

## Notes on the OBD-II compliance shell

Both ECUs are the OBD-II "Engine" target for compliance purposes (Mode 01/06/09). When an OBD-II tool queries the standard emissions PIDs / monitors, these ECUs answer with:

- All emission monitors marked "Not Avail" (no engine, no emissions)
- All monitor results "Complete" (because there's nothing to test, every test trivially passes)
- Engine Speed = 0, Coolant Temperature = 32 °F (cold sentinel), Intake Air Temperature = 32 °F, Throttle Position = 0 %, Calculate Load = 0 % — all sentinels

Don't read these as real values. The real Engine Run Time counter (9659 sec front / 9698 sec rear) does increment, so it could function as a "lifetime ECU on-time" counter — though that's not how the parameter is officially defined.
