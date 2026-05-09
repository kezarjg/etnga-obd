---
name: Main Body ECU (Toyota EM "F8 Main Body ECU")
diagnostic_request_id: 0x750
diagnostic_response_id: 0x758
gateway_sub_target: 0x40
em_reference: F8 Main Body ECU
physical_bus: B-CAN
isotp: mixed-addressing
sessions_observed:
  - default                # 0x01
sources:
  - 2026-05-08_1955_ecu-mapping-marathon
  - 2026-05-08 Main Body Data List snapshot (Techstream, Ready mode)
confidence: high
---

# Main Body ECU

Behind the gateway at `0x750` sub-target `0x40`. Frames address as `750#40 …` / `758#40 …`. Toyota EM "F8 Main Body ECU" on the B-CAN bus.

This is the **body-state mother lode** — owns door states, lock states, hood/trunk, window positions, seat memory + driver recognition, mirror controls, wiper system, rain sensor (if equipped), light sensor / solar / illuminance, AHS/LDM coordination, and **humidity + glass temperature** (used by auto-defrost and useful for preconditioning logic).

## Diagnostics

| Aspect | Value |
|---|---|
| Request ID | `0x750` with sub-target byte `0x40` |
| Response ID | `0x758` with sub-target byte `0x40` echoed |
| Transport | ISO-TP mixed-addressing (strip first byte before applying ISO-TP rules) |
| Confirmed services | `0x10`, `0x19`, `0x22`, `0x3E` |

## Functional roles (per Data List inventory, 2026-05-08)

### Door states (5 doors)

| Parameter | Sample | Notes |
|---|---|---|
| FR/FL/RR/RL Door Courtesy Switch Status | Close each | per-door open/closed |
| Back Door Courtesy Switch Status | Close | trunk/liftgate |
| Hood Courtesy Switch Status | Close | hood |

### Door lock states

| Parameter | Sample | Notes |
|---|---|---|
| FR/FL/RR/RL Door Lock Position Switch Status | Unlock each | per-door lock |
| Back Door Lock Position Status | Unlock | back-door lock |
| **All-locked** (derived) | not directly exposed | All 5 in "Lock" state ⇒ vehicle fully locked |

### Lock/unlock command switches (input source detection)

| Parameter | Sample | Notes |
|---|---|---|
| Manual Door Lock Switch (D/P Door) | OFF | physical lock button being pressed |
| Manual Door Unlock Switch (D/P Door) | OFF |  |
| Door Lock Switch Status by a Mechanical Key (D/P Door) | OFF | metal-key turn detection |
| Door Unlock Switch Status by a Mechanical Key (D Door) | OFF |  |

### Back door / liftgate

| Parameter | Sample | Notes |
|---|---|---|
| Back Door Opener Switch (Outside) | OFF | exterior tailgate button |
| Back Door Operation Switch (Instrument) | OFF | dash button |

### Outer mirrors

| Parameter | Sample | Notes |
|---|---|---|
| Outer Mirror Fold Switch | OFF | manual fold button |
| Outer Mirror Auto Switch | ON | auto-fold-when-locked enabled |
| Outer Mirror Control Switch (RH/LH Select) | OFF | mirror selector |
| Outer Mirror Control Switch (Surface Adjust Right/Left/Up/Down) | OFF each | adjust direction |

### Power windows (×4 doors with quadrant-resolution position monitoring)

For each of D / P / RR / RL doors, the ECU exposes:

| Parameter | Sample | Notes |
|---|---|---|
| P/W Jam Protection Glass Position (Close-1/4) | OK | window in fully-closed quartile |
| P/W Jam Protection Glass Position (1/4-2/4) | OK | window in 25-50% open quartile |
| P/W Jam Protection Glass Position (2/4-3/4) | OK | 50-75% |
| P/W Jam Protection Glass Position (3/4-Open) | OK | 75-100% / fully open |
| Power Window AUTO Switch | OFF | auto-up/down button |
| Power Window UP / DOWN Switch | OFF / OFF | manual buttons |
| Power Window Initialize Status | Initialized | calibration learned |

The 4 quadrant flags per window give **discrete window position** — closer to closed/open vs. nominal "are windows open" without full % resolution.

### Wipers + rain sensor

| Parameter | Sample | Notes |
|---|---|---|
| Wiper Motor Cam Switch 1 | OFF | mechanical position cam |
| Wiper Control Switch HI | OFF | high-speed switch |
| Wiper Switch Auto Signal | ON | auto-mode enabled |
| Wiper Operation Request (from other system) | OFF | request from another ECU (e.g. Headlight Control for cleaning) |
| Wiper Motor Control Status | Normal | health |
| Wiper Intermittent Time Volume | Longest | intermittent setting |
| Automatic Wiper Wiping Mode (Sensor to Wiper) | Stop | currently no wiping |
| Rain Sensor | Without | possibly not equipped on this car, or "no rain currently" — ambiguous |
| Rain Sensor Status | Normal | sensor health |
| Rain Sensor Level Status | Normal |  |
| Rain Sensor High/Low Temperature Status | Normal |  |
| Wiper LO Single Operation Signal | With | capability flag |

### Light sensor / solar / illuminance

| Parameter | Sample | Notes |
|---|---|---|
| Light Sensor Illuminance | 0 | ambient light (parked indoors / evening) |
| Insolation Amount of Solar Sensor RH | 0 | sun load right side |
| Insolation Amount of Solar Sensor LH | 0 | sun load left side |
| Light Sensor Connection History | With | sensor-presence flag |
| Light Control Switch (HEAD) | OFF | headlight switch position |
| Auto High Beam Main Switch | OFF | auto high-beam button |

The solar sensors are **duplicated from HVAC** (`0x7C4` also has Front Left/Right Solar Sensor). Likely cross-broadcast from one source.

### Adaptive High-beam System (AHS) / Lane Departure Mitigation (LDM)

| Parameter | Sample | Notes |
|---|---|---|
| AHS Function | Not Available | this car doesn't have AHS (Adaptive High-beam System) |
| High Beam Headlights (By AHS ECU) | Light OFF | sentinel since AHS not available |
| Headlight ECU Function | Available | basic headlight control is present |
| LDM ECU Function | Not Available | no Lane Departure Mitigation as separate ECU |
| Automatic High Beam Headlights Sensor Detection Status | Speed | ? |

### Glass breakage / intrusion

| Parameter | Sample | Notes |
|---|---|---|
| Glass Breakage Sensor | Without | not equipped |
| Intrusion Sensor Cancel Switch | OFF | physical cancel button (sensor itself isn't installed) |

### Power state (12V supply tracking)

| Parameter | Sample | Notes |
|---|---|---|
| Sub +B Power / Second +B Power | 0.6 V | secondary supply rail (low — backup battery off?) |
| +B Power | ON | main supply |
| IGR Power | ON | ignition-relay power |

### Rear seat occupancy (3 rear seats)

| Parameter | Sample | Notes |
|---|---|---|
| RC-Seat Occupant Sensor Switch | OFF | rear center |
| RL-Seat Occupant Sensor Switch | OFF | rear left |
| RR-Seat Occupant Sensor Switch | OFF | rear right |

Front seat occupancy is on the SRS Airbag ECU (`0x780`), not here.

### **Humidity and glass temperatures** (for auto-defrost + preconditioning use)

| Parameter | Sample | Notes |
|---|---|---|
| **Humidity** | 25.4 % | in-cabin relative humidity sensor |
| **Glass Temperature** | 75.2 °F | windshield surface temp |
| **Glass Surroundings Temperature** | 83.3 °F | air temp near the glass |

**This is high-value novel content.** Cabin preconditioning logic can use humidity + glass temp delta to anticipate defrost needs and decide whether to engage the rear defogger / front deicer along with cabin warming.

### Driver Recognition + Smart Key linking

| Parameter | Sample | Notes |
|---|---|---|
| Driver Recognition Status | Driver1 | currently active driver profile |
| My Settings Display Device | Multi Display | which display shows the my-settings menu |
| Driver1/2/3 and key ID1-7 Link Status (21 cells) | "With" only at Driver1↔key ID1; rest "Without" | which keys are paired to which driver profiles |
| Driver1/2/3 and Digital key ID1-7 Link Status (21 cells) | all "Without" | digital-key (phone-as-key) bindings — none paired |

So the user has 1 physical key paired to Driver1, no digital keys paired.

### Memory seats

| Parameter | Sample | Notes |
|---|---|---|
| D Seat Information | "With Return" | driver seat is motorized with memory + auto-return on key change |
| Driver Seat MEM_1/2/3 Memory | Without each | no memory positions saved |
| MEM Switch No. with Key ID 1-7 | NONE each | no key-to-memory-position bindings |

## Notable findings about THIS car

- **AHS Not Available** — no Adaptive High-beam System.
- **LDM ECU Not Available** — no Lane Departure Mitigation as a separate module.
- **Glass Breakage Sensor: Without** — not equipped.
- **Rain Sensor parameter ambiguous** — sensor itself appears to exist (Status/Level/Temperature all "Normal") but the top-level "Rain Sensor: Without" might mean "no rain currently" rather than "not equipped". Worth disambiguating.
- **Driver Recognition active**: Driver profile 1 is the active driver, with one paired physical key. No digital keys paired. Memory positions not saved.
- **Outer Mirror Auto = ON** — mirrors auto-fold when locked.
- **Wiper Auto = ON** — auto-wiping mode enabled.

## Open questions

- **DID isolation for the 6 door/trunk/hood states** — single biggest yield (knocks out all per-door state signals at once).
- **Lock-state aggregation** — is there a single "all locked" DID, or do we read 5 individual flags and AND them?
- **Window positions** — the 4-quadrant resolution per door is unusual. Is there a finer-grained DID with actual % position, or is the quadrant the best Toyota exposes?
- **Headlight on/off state** — not in this Data List. Probably on Headlight Control (`0x750/0x70`) or scaffolded via Wiper / Light switch DIDs.
- **Turn signal / hazard state** — not visible here. May be on a different ECU (Combination Switch?).
- **Disambiguate "Rain Sensor: Without"** — equipped-or-not vs. currently-not-detecting.
