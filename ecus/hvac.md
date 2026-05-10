---
name: HVAC / Air Conditioner (Toyota EM "F28 Air Conditioning Amplifier Assembly")
diagnostic_request_id: "0x7C4"
diagnostic_response_id: "0x7CC"
toyota_name: Air Conditioner
em_reference: F28 Air Conditioning Amplifier Assembly
physical_bus: B-CAN
isotp: standard
sessions_observed:
  - default                # 0x01
techstream_parameter_count: ~75
confidence: high
---

# HVAC / Air Conditioner

Toyota's "Air Conditioner" ECU per Techstream — officially the **F28 Air Conditioning Amplifier Assembly** on the B-CAN bus. Coordinates the entire thermal system: cabin climate, refrigerant cycle (compressor and electric expansion valves), cabin PTC heater, **and the battery chiller path**.

> **Important architectural finding (2026-05-08)**: the **chiller for battery cooling lives on this ECU**, not on the EV Battery ECU. When `0x747`'s `0x1124` Water Cooling routine engages the battery coolant pump, the BMS sends a request to this ECU to open the **Battery Electric Expansion Valve** and route refrigerant through the battery chiller. The +1.0 kW A/C Consumption Power jump observed during the cooling test was this ECU autonomously responding. The Battery ECU runs the pump; the HVAC ECU runs the refrigerant.

## Diagnostics

| Aspect | Value |
|---|---|
| Request ID | `0x7C4` (standard 11-bit) |
| Response ID | `0x7CC` (request + 8) |
| Transport | Standard ISO-TP |
| Confirmed services | `0x10`, `0x19`, `0x22`, `0x3E` |

## DIDs decoded (from prior ABRP integration)

| DID | Meaning | Encoding |
|---|---|---|
| `0x1001` | **Cabin temperature** | int16 BE / 100 → °C |
| `0x1002` | **Ambient temperature** | int16 BE / 100 → °C (this ECU's view; cluster's `0x1141` is a separate cross-broadcast) |
| `0x1036` | **HVAC setpoint** | piecewise: `if(A=0,0, if(A<28, A/2 + 15.5, if(A<55, (A-1)*5/9, if(A=55,100, if(A<100, A/2 - 34, (A-74)*5/9)))))` — encodes °C, °F, LO, HI in distinct value bands |

## Functional roles (per Data List inventory, 2026-05-08 snapshot — AC running)

The HVAC ECU's scope is much broader than just cabin climate. Functional groups:

### Cabin temperature sensing

| Parameter | Sample | Notes |
|---|---|---|
| Room Temperature Sensor | 72.00 °F | cabin temp — same as `0x1001` |
| Ambient Temperature Sensor | 63.28 °F | outside temp — same as `0x1002` |
| Ambient Temperature Adjustment Value | 58.37 °F | filtered/compensated ambient |
| Front Left Solar Sensor | 0.0 W/m² | sun load |
| Front Right Solar Sensor | 0.0 W/m² | dual sensor for left/right asymmetric sun loading |

### User climate settings (dual zone)

| Parameter | Sample |
|---|---|
| Front Right Set Temperature | 68 °F |
| Front Left Set Temperature | 68 °F |
| Blower Level | 4 (of presumably 7) |

### Air outlet sensing (the actual delivered air)

| Parameter | Sample | Notes |
|---|---|---|
| Front Right Air Outlet Temperature | 78.60 °F | post-mix-damper, post-heater-core |
| Front Left Air Outlet Temperature | 78.60 °F | independent for dual-zone |

Note: with cabin 72 °F, setpoint 68 °F, and air outlet 78.6 °F at the time of the snapshot, the air was warmer than the cabin even though the AC compressor was running. Implies hot coolant (82–84 °F at heater core) was passing through the cabin heater core, raising the air temp despite the AC running. Heat-pump-mode complexity — sometimes the system is rejecting heat from the battery loop into the cabin or other paths.

### Refrigerant cycle — sensors and pressures

The Solterra uses a **heat-pump system with HFO-1234yf refrigerant** (low-GWP). Multiple sensors trace the refrigerant loop:

| Parameter | Sample | Loop position |
|---|---|---|
| Refrigerant Outlet Temperature Sensor | 91.04 °F | compressor outlet (hot side) |
| Regulator Pressure Sensor | 104 psi | high-side pressure |
| Evaporator Refrigerant Temperature Sensor | 43.52 °F | post-evaporator (cold side) |
| Evaporator Fin Thermistor | 44.22 °F | evaporator fin temp |
| Evaporator Outlet Refrigerant Pressure Sensor | 42 psi | low-side pressure (cabin path) |
| Battery Outlet Refrigerant Temperature Sensor | 70.11 °F | post-battery-chiller |
| Battery Outlet Refrigerant Pressure Sensor | 42 psi | low-side pressure (battery path) |
| Refrigerant Gas Type | Hfo1234yf | static config |
| Refrigerant High Pressure History Count | 0 | lifetime over-pressure events |
| Refrigerant Low Pressure History Count | 0 | lifetime low-pressure events |

### Refrigerant cycle — control

The system has **3 electric expansion valves** (EEV) and a **3-way flow valve**, allowing dynamic routing of refrigerant between cabin cooling, cabin heating (heat-pump mode), and battery cooling:

| Parameter | Sample | Role |
|---|---|---|
| Battery Electric Expansion Valve Target Position | 0 % | battery chilling currently OFF (no battery cooling routing) |
| Battery Electric Expansion Valve Current Position | 0 % | actual position |
| Battery EEV Voltage / Temperature Protection Status | OFF | fault flags |
| A/C Cooling Electric Expansion Valve (Heat Management Driver) Target Position | 13 % | cabin AC active, modest valve opening |
| A/C Cooling EEV Current Position | 13 % | matches target |
| A/C Cooling EEV Voltage / Temp Protection Status | OFF | fault flags |
| Cooling Electric Expansion Valve | 13 % | duplicate of A/C cooling EEV |
| Three-way Flow Adjustment Valve Target Position | 90 % | refrigerant routing valve (cabin/battery/heat-pump path selector) |
| Three-way Flow Adjustment Valve Current Position | 90 % | matches target |

### Coolant loop sensors (low-temperature loop, separate from inverter loop)

| Parameter | Sample | Notes |
|---|---|---|
| HV Electric Heater Inlet Coolant Temperature Sensor | 82.40 °F | warm coolant entering the cabin PTC heater |
| HV Electric Heater Outlet Coolant Temperature Sensor | 84.20 °F | warm coolant leaving — heater is OFF currently, so why warm? Likely residual heat or heat-pump rejection |
| Air Conditioning Coolant Temperature Sensor | 83.28 °F | average coolant in the AC loop |
| Chiller Outlet Coolant Temperature Sensor | 68.09 °F | post-chiller, when chiller path is active |
| Radiator Outlet Coolant Temperature Sensor (LT Coolant Circuit) | 68.09 °F | low-temperature loop radiator outlet |

### Compressor and water pump control

| Parameter | Sample | Notes |
|---|---|---|
| Compressor Target Speed | 2194 rpm | AC compressor set point |
| Compressor Actual Speed | 2194 rpm | matches target — running stably |
| Electric Water Pump Target Duty | 10000 % | likely 100.00 % (×100 internal scale) |
| Electric Water Pump Actual Speed | 5732 (units?) | running |
| Water Pump | ON | boolean status |

### Cabin PTC heater (HV Electric Heater) — the cabin heat source distinct from the battery coolant heater

| Parameter | Sample | Notes |
|---|---|---|
| HV Electric Heater Overheat Detection Status | Not Detected | |
| HV Electric Heater Target Electric Power | 0 W | requested heater power |
| HV Electric Heater Consumption Electric Power | 0 W | actual heater consumption |

**Two distinct heaters in the system**:
1. **Battery Coolant Heater Assembly (A36)** — owned by Battery ECU `0x747`, uses Heater Relay (test routine `0x2F 28 06`), heats battery coolant only.
2. **HV Electric Heater (here)** — cabin PTC heater, controlled by HVAC ECU.

For preconditioning the cabin (separate from the battery), this ECU's HV Electric Heater is the lever.

### Damper control servos (target/actual pulse positions for each damper)

Five servo motors, each with target and actual pulse counts:

| Servo | Sample (target / actual) | Function |
|---|---|---|
| Front Left Air Mix Damper | 187 / 187 | LH zone hot/cold mix |
| Front Right Air Mix Damper | 325 / 325 | RH zone hot/cold mix (dual-zone) |
| Air Inlet Damper | 196 / 196 | fresh vs. recirculated air |
| Front Air Outlet Damper | 256 / 256 | face/floor/defrost direction |
| Front Control Rear Air Outlet Damper | 261 / 261 | rear-cabin airflow |

Plus per-servo "Initialization History Count" (all 0 — never had to relearn position).

### Accessory heaters and switches

| Parameter | Sample | Notes |
|---|---|---|
| Steering Heater | OFF | steering wheel heater |
| Front Right Seat Heater | OFF | |
| Front Left Seat Heater | OFF | |
| Rear Right Seat Heater | OFF | |
| Rear Left Seat Heater | OFF | |
| Rear Defogger Relay | OFF | rear window defroster |
| Front Deicer Relay | OFF | front windshield deicer (separate from defroster) |

### Sensor presence (resistance reads)

| Parameter | Sample | Notes |
|---|---|---|
| Front Left Seat Heat Sensor | 9.98 kΩ | seat heater NTC sensor |
| Front Right Seat Heat Sensor | 9.81 kΩ | |
| Rear Left Seat Heat Sensor | 11.02 kΩ | |
| Rear Right Seat Heat Sensor | 11.28 kΩ | |

Resistance values around 10 kΩ at room temp are normal for NTC thermistors. Useful as "is the seat heater wired up" sanity checks.

## Implications for cabin preconditioning

A client can implement **cabin climate preconditioning** alongside battery preconditioning by:

1. Reading current cabin temperature (`0x1001`) and setpoint (`0x1036`)
2. Determining heating vs cooling need
3. Engaging the HV Electric Heater (for cabin warming) or compressor (for cabin cooling) — **but the activation mechanism isn't yet known**. Toyota likely has a "Remote Climate" routine or DID; the EV ECU's "Remote Air Control System" parameter (currently "Unable") is suggestive — triggering that would let the HVAC autonomously run to setpoint without driving each individual valve and pump.
4. Pairing with battery preconditioning so both are ready when the vehicle reaches a charger or departure time

## Cross-ECU coordination map

```
EV Battery ECU (0x747) ──── battery coolant pump ──→ battery coolant loop
                                                       ↓
                                                   chiller heat exchanger
                                                       ↑
HVAC ECU (0x7C4) ──── compressor + Battery EEV ──→ refrigerant loop ──→ chiller heat exchanger
                                                       ↑
                                                   evaporator (cabin AC)
                                                       ↑
                                                   cabin air
```

When EV Battery's Cooling Test runs, BMS signals HVAC ECU → HVAC opens Battery EEV (currently 0%) → compressor ramps → chiller path engaged → battery coolant cooled → BMS sees cooler return temp.

For preconditioning **cooling**, the `0x31 01 11 24` routine on EV Battery is sufficient (BMS coordinates with HVAC autonomously).

For preconditioning **heating**, the `0x2F 28 06 03 00 01 00 01` IO control on EV Battery engages the **battery coolant heater only** — it does NOT involve the HVAC ECU's HV Electric Heater (cabin PTC). Warming the cabin too requires a separate HVAC-side command.

## Open questions

- **DIDs for blower / dampers / heaters** — single-parameter isolation runs in Techstream Data List would pin them quickly. Each one unlocks another decoded signal.
- **Compressor speed control DID** — for instantaneous AC kW estimation alongside the EV ECU's "A/C Consumption Power" cross-broadcast.
- **"Remote Air Control System" mechanism** — the EV ECU exposes this parameter (currently "Unable" because no Bluetooth/cellular remote command is active). Find the routine ID or DID that triggers it for externally-driven cabin preconditioning. Likely a `0x31 01` routine on this ECU.
- **Heat-pump mode flag** — the system supports heat-pump heating (extracts heat from refrigerant loop instead of using PTC). When the heat pump is in heating mode, the refrigerant flow direction reverses. Worth identifying the mode flag for downstream display.
- **Why air outlet (78.6 °F) > cabin (72.0 °F) when AC is running** — likely the HV Electric Heater coolant loop is at 82-84 °F warming the cabin heater core even though the heater itself is OFF. Need to characterize this further to understand cabin thermal lag.
