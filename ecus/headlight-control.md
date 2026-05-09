---
name: Headlight Control
diagnostic_request_id: 0x750
diagnostic_response_id: 0x758
gateway_sub_target: 0x70
isotp: mixed-addressing
sessions_observed:
  - default                # 0x01
sources:
  - 2026-05-08_1955_ecu-mapping-marathon
  - 2026-05-08 Headlight Control Data List snapshot (Techstream, Ready mode, dusk)
confidence: high
---

# Headlight Control

Behind the gateway at `0x750` sub-target `0x70`. The primary headlight controller (the AFS / auto-leveling ECU). Has a sub-controller at `0x750/0x1E` ("Headlight Control (Sub)") presumably for the other side or for backup.

Smaller Data List than expected (~25 parameters). Closes the basic "are the headlights on" question for OVMS but **does not expose** turn signals, hazard flash state, high beam, or fog lights — those live elsewhere (TBD).

## Diagnostics

| Aspect | Value |
|---|---|
| Request ID | `0x750` with sub-target byte `0x70` |
| Response ID | `0x758` with sub-target byte `0x70` echoed |
| Transport | ISO-TP mixed-addressing |
| Confirmed services | `0x10`, `0x19`, `0x22`, `0x3E` |

## Functional content (per Data List, 2026-05-08)

### Light states (the OVMS-relevant content)

| Parameter | Sample | OVMS use |
|---|---|---|
| **Low Beam** | ON | `v.e.headlights` primary indicator |
| **Daytime Running Light** | OFF | DRL state (custom) |
| **Clearance Light (+ Front Side Marker Light)** | ON | parking/marker lights (custom) |
| **Cornering Light / Front Side Illuminate Light** | OFF | cornering light state |
| **Cornering Light / Front Side Illuminate Light (Dim)** | OFF | dimmed cornering state |

Capture context: low beams + clearance lights ON, DRLs OFF — the user is driving at dusk/night with proper lighting active.

### Auto-leveling system (vehicle height tracking)

The Solterra has front and rear vehicle height sensors used for headlight auto-leveling — keeps the beam on the road regardless of vehicle pitch (cargo loading, braking, etc.).

| Parameter | Sample | Notes |
|---|---|---|
| Front Height Sensor Power Supply Voltage | 4.95 V | sensor health |
| Rear Height Sensor Power Supply Voltage | 3.61 V | sensor health (lower than front — different sensor scaling, or indicates calibration) |
| Rear Vehicle Height Calibration Value | 759 | learned baseline |
| Rear Vehicle Height Initialization Determined Result | Valid | calibration good |
| Initialization Count | 1 | how many times re-initialized |
| Headlight Leveling Indicator Display | OFF | dash warning indicator |
| Headlight Leveling Initialize Incomplete Display | OFF | dash warning |
| Headlight System Malfunction Display | OFF | dash warning |

### Cross-broadcast vehicle state

Used by the AFS for steering-tracking and motion-aware lighting:

| Parameter | Sample | Notes |
|---|---|---|
| FR Wheel Speed / FL Wheel Speed | 0.00 MPH each | for AFS speed gating |
| Vehicle Speed | 0.00 MPH | cross-broadcast |
| Vehicle Acceleration | 0.00 m/s² |  |
| Steering Angle Value After Calibration | 21.0 deg | for AFS swivel tracking |
| Steering Angle Zero Point Calibration Value | 3.0 deg | learned offset |

### Power and bin

| Parameter | Sample | Notes |
|---|---|---|
| ECU Power Source Voltage | 13.38 V | this ECU's 12V supply |
| IG Voltage | 13.26 V | ignition power |
| Powertrain Status | ON | cross-broadcast |
| BIN1 Positive Terminal Rank | Rank D | possibly market/region bin coding ("Rank D" purpose TBD) |

## OVMS mappings

| OVMS metric | Source parameter | Status |
|---|---|---|
| **`v.e.headlights`** | Low Beam (ON/OFF) | ⬜ DID isolation pending |
| Custom: DRL | Daytime Running Light | ⬜ |
| Custom: parking/marker lights | Clearance Light + Front Side Marker | ⬜ |
| Custom: cornering light | Cornering Light / Front Side Illuminate Light | ⬜ |

## What's NOT here (OVMS gaps still open)

The Data List doesn't expose:
- **Turn signal state** (left / right) — `v.e.headlights.turn` not findable here
- **Hazard light flashing state** — Cluster's "Hazard Flasher Switch" tracks the *button*, not the actual flash output
- **High beam state** — owned by the AHS (Adaptive High-beam System) ECU which on this car reports "Not Available". May be readable via a different parameter or Combination Switch ECU.
- **Fog light state** — not exposed here

These remaining lighting metrics likely live on the **Combination Switch / Light Control ECU** (often integrated into Main Body or steering column). Worth a future investigation.

## Open questions

- **`BIN1 Positive Terminal Rank: Rank D`** — what does this enum encode? Possibly market region code or headlight assembly bin sorting (different headlight optical bins for different specs).
- **Front vs rear height sensor voltage discrepancy** (4.95 V vs 3.61 V) — different sensor designs, or asymmetric calibration?
- **High beam / fog light / turn signal state** — find which ECU exposes these.

## Notes

- The "Headlight Control (Sub)" entry at gateway sub-target `0x1E` is presumably a redundant/secondary controller. Its Data List response was too short during the marathon to characterize. Worth a brief look in a future session — likely passes through some of the same parameters as a backup path.
- The Solterra's headlights are LED with auto-leveling but **no AHS** (Adaptive High-beam System) on this car — confirmed by the AHS Function = "Not Available" parameter on the Main Body ECU.
