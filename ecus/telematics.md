---
name: Telematics (Data Communication Module / DCM)
diagnostic_request_id: 0x750
diagnostic_response_id: 0x758
gateway_sub_target: 0xC7
isotp: mixed-addressing
sessions_observed:
  - default                # 0x01
sources:
  - 2026-05-08_1955_ecu-mapping-marathon
  - 2026-05-08 Telematics Data List snapshot (Techstream, Ready mode)
confidence: high
---

# Telematics (DCM)

Behind the gateway at `0x750` sub-target `0xC7`. Toyota's Data Communication Module — the cellular modem that connects the car to Toyota's cloud services (Toyota Connected, Stolen Vehicle Tracking, Automatic Collision Notification, Remote A/C, etc.).

> **Lower direct telemetry value than expected.** The Data List is mostly **Toyota Connected service-enablement flags** (which features are Enable / Disable / Registered) — configuration state, not real-time vehicle metrics. The cellular modem identifiers (IMEI / MSISDN / ICCID) are exposed here. Notably **absent**: cell signal strength, modem state, GPS position.

## Diagnostics

| Aspect | Value |
|---|---|
| Request ID | `0x750` with sub-target byte `0xC7` |
| Response ID | `0x758` with sub-target byte `0xC7` echoed |
| Transport | ISO-TP mixed-addressing |
| Confirmed services | `0x10`, `0x19`, `0x22`, `0x3E` |

## Functional content (per Data List, 2026-05-08)

### Cellular modem identifiers — **PII / car-identifying — keep out of public docs**

The actual values for this car are stored in workspace-only context (treat like VIN in `CAR.md`):

| Parameter | What it is | Format / value (redacted in public docs) |
|---|---|---|
| IMEI | Modem hardware identifier | 15-digit decimal, ITU-T E.212 — readable here as a single field |
| MSISDN | The DCM's embedded SIM phone number | 10-digit US format |
| ICCID | SIM card serial number | 20-digit, split as ICCID(High) and ICCID(Low) — concatenate for full ICCID |
| Software Version High | Modem firmware version | "172.0.6a00" on this car (May 2026) |
| Software Version Low | (paired with High) | empty on this car |
| PLMN(MCC) | Mobile country code | 310 (USA) |
| PLMN(MNC) | Mobile network code | 410 (AT&T) — confirms US T-Mobile/AT&T cellular for this car |

### Toyota Connected service activation

| Parameter | Sample | Notes |
|---|---|---|
| Telematics Activation Status | Active | Toyota Connected services are active on this car |
| Communication Status with TSC | Complete | Toyota Service Center / Smart Center connection healthy |
| Communication Pausing State | Pausing | DCM currently in low-power pause state (probably normal at idle) |
| Brand TOYOTA | No | Subaru-rebranding — Solterra reports as not-Toyota even though it uses Toyota's DCM |

### Per-feature enable/disable flags (~35 features)

These map to user-toggle settings in Toyota Connected. Currently enabled on this car:

- OTA Reprogramming
- Remote Service Function, Vehicle State Notification, Remote Warning
- Remote Detected DTC, Remote FFD, Remote Detect DTC, Remote Confirmation, Remote Precaution, Remote Monitoring, Remote Diagnostics Recorder, Remote Operation
- Remote Vehicle Location, Maintenance Message, Remote A/C by Smart Key
- eConnect: State Notification, Event Notification, State Change Notification, Remote Operation, Remote Vehicle Location
- Driving Trip Data Notification
- **Automatic Collision Notification Function** — post-crash auto-alert is enabled
- **Manual Emergency Call Function** — SOS button
- Wireless Health Check Function
- Stolen Vehicle Tracking Function

Currently **disabled**:
- HELPNET Function (Japan-market emergency-call service — properly disabled in US)
- Remote Vehicle Tracking
- Information Collection via Ethernet
- Information Collection via CAN
- Stolen Vehicle Tracking *(the TSC-side service — note the Function flag above is enabled but the SVT-policy flag here is Disable)*
- IP Remote Detected DTC, IP Remote FFD, IP Remote Monitoring
- Alarm Notification, Remote Immobilizer
- Remote Navigation Display, Remote Vehicle Control History
- Collision Management Function

### Registration state of paired remote-control features

| Parameter | Sample | Notes |
|---|---|---|
| Remote Engine Starter Registration Status | Registered | Smart-Key-triggered Remote A/C is set up |
| Remote Door Lock Function | Registered | Toyota App can lock/unlock |
| Communication Remote Engine Starter | Not Available | (different from above — refers to remote-start-via-cellular, not via key fob) |

### Indicator + audio (the dashboard "telematics" lights)

| Parameter | Sample | Notes |
|---|---|---|
| Green Indicator | Nighttime Brightness-Middle | the green DCM status LED |
| Red Indicator | Lighting OFF | the red DCM error LED |
| Beep | OFF | audible alert |
| Telematics Microphone Volume Adjust | 5 | for voice connections |
| Telematics Speaker Volume Adjust | 9 |  |

### State flags

| Parameter | Sample | Notes |
|---|---|---|
| Dormant Flag | OFF | DCM not in long-term dormant state |
| New Energy Vehicle | OFF | unexpected "OFF" — this IS a BEV. Possibly market-specific flag. |

## Useful telemetry from this ECU

Almost nothing — this ECU exposes service configuration, not real-time vehicle state. The only potentially useful items:

| Signal | Source | Use |
|---|---|---|
| Telematics activation status | Telematics Activation Status | one-time read for "is Toyota Connected active" |
| Cellular network identifiers | IMEI / MSISDN / ICCID | one-time read for device provenance — informational only for any external integration with its own connectivity |

## Notable absences (things I expected but didn't find)

- **Cell signal strength (RSSI / RSRP)** — the modem clearly knows this; it's just not exposed via this Data List
- **Cell network state** (registered / connected / transmitting / idle) — not visible
- **Last successful TSC connection time** — would be useful for diagnosing "why didn't my remote command go through"
- **GPS position** — confirmed not on diagnostic surface (cross-checked with Navigation ECU's empty Data List too)
- **Currently active digital key sessions** — Main Body has the *paired* digital keys; this would be the *currently authenticated* sessions

## Open questions

- **`Brand TOYOTA: No`** — explicit "not Toyota" flag despite this being Toyota's DCM. Probably the Subaru rebranding signaling, but worth confirming.
- **`New Energy Vehicle: OFF`** on a BEV — what is this flag actually tracking? Possibly an Asia-market regulatory flag (NEV is a China/regulatory term) that's irrelevant in the US market.
- **`Stolen Vehicle Tracking Function: Enable` vs `Stolen Vehicle Tracking: Disable`** — the *capability* is enabled but the *active subscription/policy* appears disabled. Toyota's stolen vehicle tracking is typically a paid-subscription add-on; this likely reflects "feature available" vs "subscription paid".

## Privacy note

The IMEI / MSISDN / ICCID values **identify this specific car**. Same standard as the VIN in `CAR.md` — workspace-internal, not in any committed public artifact.
