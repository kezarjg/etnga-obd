# ecus/

One Markdown file per ECU as we identify them. Maps ECU name → response IDs, supported services, observed DIDs, and any noteworthy quirks.

Filenames are lowercase functional names: `ev.md`, `engine.md`, `body.md`, `airbag.md`, etc. Use whatever the user (or Techstream) calls them as the seed.

Each file should at minimum list:
- The diagnostic request/response ID pair (e.g., `0x7E0` ↔ `0x7E8`).
- Diagnostic session types this ECU accepts.
- Sessions where this ECU was probed.

## ECU ID inventory observed on OBD-II

The list below is everything we've seen as a request/response pair on the OBD-II bus. **Techstream reports the Solterra has 39 ECUs total**. As of `2026-05-08_1955_ecu-mapping-marathon` we have **identities for 35**: 14 top-level OBD-II IDs (1 of which is a multi-target gateway) plus 22 sub-targets behind the gateway. Three top-level OBD IDs (`0x727`, `0x7E2`, `0x7E3`, `0x7E6`) are still unidentified, and likely correspond to virtual ECUs in the OBD-II standardized address range.

Pattern: response ID = request ID + 8 in every observed case.

### Top-level OBD-II IDs

| Request | Response | Techstream name | Source session | Notes |
|---|---|---|---|---|
| `0x705` | `0x70D` | **Rear Motor Generator = L4 Rear Transaxle with Motor & Inverter** | ecu-mapping-marathon, 2026-05-08 Data List review | AWD-only ECU. Same parameter shape as Front Motor Generator. Adds: rear stator temp, rear transaxle oil temp, rear DC bus voltage (VLR), 3-phase currents, rear inverter shutdown status (currently ON when parked — Toyota saves energy by parking the rear inverter at idle). See `motor.md`. |
| `0x724` | `0x72C` | **Motor Generator (front) = D9 Front EV Motor Control Inverter** | ecu-mapping-marathon, 2026-05-08 Data List review | Owns front motor + inverter telemetry: motor RPM, torque, 3-phase currents (lower + higher resolution), stator temp (66°F at idle), inverter temp, inverter coolant + water pump (shared with rear), motor/generator cooling oil pump, transaxle oil temperature, resolver state. Also serves the OBD-II Mode 01/06/09 emission compliance shell with all sentinels (engine speed = 0, etc.). See `motor.md`. |
| `0x727` | `0x72F` | unknown | health-check | Tiny ECU — Health Check only queried `F181` (version). Not present in Techstream's connect list as tested. |
| `0x745` | `0x74D` | **Plug-in Charge Control = OBC + DC-DC Converter (Toyota EM "A33")** | health-check, ABRP config, ecu-mapping-marathon | The full On-Board Charger ECU — owns PFC, DC/DC converter, charge inlet temps, AC/DC relays, charging session state, charge history. Owns SOC (DID `0x1739`), charge-state enum (`0x10D1`), DCFC-state enum (`0x1668`), charge-amount-upper-limit setting, "Battery Temperature when Charging Start" log. **Previously mislabeled "BMS Monitor" then "session controller only"** — re-corrected 2026-05-08 when the full Data List inventory revealed PFC / DC-DC / charger-I/O parameters. See `plug-in-charge-control.md`. |
| `0x747` | `0x74F` | **EV Battery** | health-check, ev-battery-connect, ecu-mapping-marathon | The actual battery ECU (Toyota EM "k3"). High-feature, uses dynamic DIDs. Owns the 192-byte `0x182E` cell-voltage block. See `ev-battery.md`. |
| `0x750` | `0x758` | **Gateway** (multi-target, mixed addressing) | smoke-test, health-check, tpms-connect, ecu-mapping-marathon | NOT a single ECU. 22 sub-targets identified — see "Gateway sub-targets" below and `messages/0x750.md`. |
| `0x780` | `0x788` | **SRS Airbag** | health-check, ecu-mapping-marathon, 2026-05-08 Data List review | Restraint system. Mostly diagnostic (15 squib resistance values, load sensor calibration history). 3 OVMS-relevant params: passenger occupancy (Empty/Child/Adult), passenger seatbelt buckle, driver seat track position. **No deployment-state Data List parameter** — that's only DTCs after the fact. See `srs-airbag.md`. |
| `0x792` | `0x79A` | **Front Recognition Camera** | health-check, ecu-mapping-marathon | Forward-facing ADAS camera (lane keep, AEB, traffic sign recognition). |
| `0x7A1` | `0x7A9` | **EMPS / Steering Control Actuator** | health-check, ecu-mapping-marathon | Electric motor power steering rack ECU. |
| `0x7B0` | `0x7B8` | **Brake/EPB** | health-check, ecu-mapping-marathon, 2026-05-08 Data List review | The main ABS/VSC/TRC + Electric Parking Brake ECU. Owns wheel speeds (×4), per-wheel accelerations, brake pedal stroke, master cylinder pressure, yaw rate (with redundant high-resolution sensors), G sensors, RH+LH EPB actuators with full motor diagnostics, ABS solenoids (×8), TSS/VMC integration, **regenerative cooperation** (regen-blending coordinator), brake fade flag. See `brake.md`. |
| `0x7B3` | `0x7BB` | **Steering Angle Sensor** | health-check, ecu-mapping-marathon | Standalone steering wheel angle sensor (Toyota module). |
| `0x7C0` | `0x7C8` | **Combination Meter** | ecu-mapping-marathon | The actual instrument cluster. **Previously thought to be `0x7E2`** — corrected 2026-05-08. |
| `0x7C4` | `0x7CC` | **HVAC / Air Conditioner = F28 Air Conditioning Amplifier Assembly** | health-check, ABRP config, ecu-mapping-marathon, 2026-05-08 Data List review | Coordinates the entire thermal system: cabin climate (temp/setpoint/blower/dampers), refrigerant cycle (compressor + 3 EEVs + 3-way flow valve, HFO-1234yf), cabin PTC heater (HV Electric Heater), seat/steering heaters, defogger/deicer relays, **and the battery chiller path** (the chiller lives on this ECU, not the EV Battery ECU). See `hvac.md`. |
| `0x7D0` | `0x7D8` | **Navigation System** | ecu-mapping-marathon, 2026-05-08 Data List review | Multimedia/infotainment head unit. **Diagnostic surface is minimal** — only ~10 parameters, all cross-broadcast vehicle state (odometer, gear/reverse, parking brake, voltage, speed) plus a few head-unit state flags. **GPS / map / route / destination data are NOT exposed via UDS** — Toyota keeps that internal to the unit. OVMS needs its own GPS module. EM PDF guess of "Front EV Motor Inverter" was wrong. |
| `0x7D2` | `0x7DA` | **EV ECU = F45 Hybrid Vehicle Control ECU** | techstream-connect, health-check, 2026-05-08 Data List review | The vehicle supervisor. Owns front + rear motor control (AWD coordination), vehicle dynamics (wheel speeds, steering, G/yaw), the **main HV→12V DC/DC converter**, inverter coolant loop + Coolant Distribution Valve, comprehensive **12V battery health telemetry** (incl. Status of Full Charge in Ah, lifetime Ah counters, last-5-trips history), Gear Shift Control Module integration, and 30+ lifetime trigger counters. High-feature, uses dynamic DIDs (`0x2C 01` setup → `0xF301`/`0xF302` polls). See `ev.md`. |
| `0x7E2` | `0x7EA` | unknown | health-check | Owns OBD-II Mode 01 PID `0xA6` (odometer, 0.1 km units). **Previously mislabeled "Combination Meter"** — corrected 2026-05-08. Likely a virtual ECU exposing the standardized OBD-II Mode 01 surface. |
| `0x7E3` | `0x7EB` | unknown | health-check | Standard OBD-II range. |
| `0x7E6` | `0x7EE` | unknown | health-check | Standard OBD-II range. |

Functional broadcast: `0x7DF` (tester → all ECUs).

### Gateway sub-targets (`0x750/0x758`)

22 sub-targets identified during `2026-05-08_1955_ecu-mapping-marathon`. Sorted by sub-target byte. Frame format: `750#<sub-target> <ISO-TP PCI> <payload>` — strip the first byte before applying standard ISO-TP rules. See `messages/0x750.md` for the full addressing model.

| Sub-target | Downstream ECU |
|---|---|
| `0x0F` | Front Radar Sensor |
| `0x1E` | Headlight Control (Sub) |
| `0x29` | Brake Booster |
| `0x2A` | Tire Pressure Monitor (TPMS) — see `tpms.md` |
| `0x40` | Main Body |
| `0x41` | Blind Spot Monitor B (rear corner radar) |
| `0x42` | Blind Spot Monitor A (rear corner radar) |
| `0x4A` | Power Integration No 1 |
| `0x4D` | Power Distribution Box |
| `0x5F` | **Central Gateway** (Toyota EM "F38") |
| `0x67` | Clearance Warning (parking sonar) |
| `0x70` | Headlight Control |
| `0x7B` | Circumference Monitoring Camera (Surround View) |
| `0x81` | Front Side Radar A (front corner radar) |
| `0x82` | Front Side Radar B (front corner radar) |
| `0x96` | Driver Monitor Camera |
| `0xB5` | Smart Key |
| `0xB8` | Back Door (power liftgate) |
| `0xC7` | Telematics (DCM) |
| `0xD3` | Acoustic Vehicle Alerting System (AVAS) |
| `0xE9` | **Power Source Control** — owns HV state machine; pinged by Techstream before opening any other gateway-routed ECU |
| `0xEC` | Master Switch |

### Notes on the mapping process

- To identify an ECU's function: connect to it in Techstream and watch which `0x7XY` traffic flows. The ID becomes named the moment we see a Techstream "EV ECU" / "ABS" / "Engine" / etc. screen open and that ID's traffic spike.
- **The Toyota EM39J0U PDF is unreliable for OBD-II diagnostic IDs**: every OBD-bus ID guessed from EM bus topology turned out to be wrong (front motor, rear motor, navigation). The EM is still authoritative for which ECUs *exist* and which internal CAN bus each ECU sits on, but the OBD-II diagnostic addresses must be confirmed empirically with Techstream.
- Whenever Techstream opens a gateway-routed ECU, it first pings sub-target `0xE9` (Power Source Control) — likely a check that the HV system is in the right state to accept diagnostic requests. The probe is cached for some interval; rapid sequential clicks skip the re-probe.
