---
name: Power Source Control
diagnostic_request_id: 0x750
diagnostic_response_id: 0x758
gateway_sub_target: 0xE9
isotp: mixed-addressing
sessions_observed:
  - default
  - extended (0x10 03)
confidence: high
---

# Power Source Control

Behind the gateway at `0x750` sub-target `0xE9`. Frames address as `750#E9 …` / `758#E9 …`.

**Despite the name, this is the 12V ignition / power-supply state machine, NOT the HV state machine.** Toyota's "Power Source" terminology refers to 12V power distribution (ACC / IGP / IGR buses), not the HV system. PSC handles the push-start logic, decides which 12V buses are powered based on switch state, and reports the relay-monitor feedback. It does **not** own contactor state, HV bus voltage, or ready-to-drive — those live on EV ECU (`0x7D2`) and EV Battery (`0x747`).

> **Why investigated**: PSC is the always-pinged sub-target during gateway-routed ECU connects (every Techstream click probes `0xE9` first). The initial hypothesis was that it owned the HV state machine — which turned out to be wrong. It's a precondition check that the 12V power supply is in a sane state before any other ECU is poked.

PSC is also the source for the **Total Distance Traveled** (odometer) — distinct from the OBD-II Mode 01 PID `0xA6` odometer served by `0x7E2`.

## Diagnostic surface

Confirmed services: `0x10`, `0x19`, `0x22`, `0x3E`. Standard probe DID `0x2001` returns `0x70 70` (the value Techstream uses to verify PSC is alive).

### Data List parameter inventory (from Techstream, 25 parameters)

```
Total Distance Traveled            (value)
Total Distance Traveled - Unit     (mile / km)
Push Start Switch 1                (ON/OFF)
Push Start Switch 2                (ON/OFF)
Push Start Switch 3                (ON/OFF)
Steering Unlock Switch             (ON/OFF)
Stop Light Switch                  (ON/OFF)
IGP Relay Circuit (Outside) Monitor (ON/OFF)
IGP Relay Circuit (Inside) Monitor  (ON/OFF)
IGR Relay Circuit (Outside) Monitor (ON/OFF)
IGR Relay Circuit (Inside) Monitor  (ON/OFF)
IGP Hold Circuit Monitor           (ON/OFF)
ACC Relay Monitor                  (ON/OFF)
Vehicle Running Condition (Line)   (Stop/...)
IGB Output Condition               (ON/OFF)
Power Supply Condition             (IGP ON / ACC / OFF / ...)
Shift P Signal Condition (Line)    (Shift P / ...)
Powertrain Type                    (PHV/EV-AT)
Shift P Signal Mismatch            (Detected/Not Detected)
Steering Lock - Unlock Time Out    (Detected/Not Detected)
Key Certification Time Out         (Detected/Not Detected)
IGR Relay Circuit (Outside) Malfunction  (Detected/Not Detected)
IGP Relay Circuit (Outside) Malfunction  (Detected/Not Detected)
Auto Power OFF Cancel Mode         (ON/OFF)
Accessory Mode (ACC) Transition    (Valid/Invalid)
```

### DIDs polled (only 8 — most parameters are bit-packed)

The Data List uses simple Read-DID polling (no dynamic DIDs). When all 25 parameters are enabled, Techstream cycles through these 8 DIDs:

| DID | Length | Sample value (idle, P, switches off, all relays on) | Notes |
|---|---|---|---|
| `0x0103` | 4 bytes | `02 00 61 A8` | **Confirmed: Total Distance Traveled + Unit.** byte 0 = unit (0x02 = mile, expect 0x01 = km). bytes 1-3 = uint24 BE odometer = `0x0061A8` = 25000 miles ✓ (synthetic — real reading redacted). |
| `0x1001` | 4 bytes | `EA 08 00 00` | **Switches block** (5 booleans pinned via isolation): PSS1, PSS2, PSS3, Steering Unlock Switch, Stop Light Switch. Bit positions not yet pinned — needs physical-input toggle (press brake → SLS flips). |
| `0x1003` | 4 bytes | `F8 F8 80 80` | **Relay-monitor block** (≥1 confirmed): IGP Relay Circuit (Outside) Monitor. Probably also IGP Inside, IGR Outside, IGR Inside, IGP Hold, ACC Relay. Bit positions not pinned. |
| `0x1004` | 4 bytes | `40 00 80 80` | Not yet isolated. Likely the "condition / mode" block: Vehicle Running Condition, Power Supply Condition, Shift P Signal Condition, Powertrain Type. |
| `0x1005` | 1 byte | `07` | Not yet isolated. |
| `0x1006` | 1 byte | `01` | Not yet isolated. |
| `0x1007` | 1 byte | `45` | Not yet isolated. |
| `0x1008` | 2 bytes | `F8 00` | Not yet isolated. Likely fault-flag aggregate (the 5 "Not Detected" parameters). |

### What's confirmed

| Parameter | DID | Decode |
|---|---|---|
| Total Distance Traveled | `0x0103` | uint24 BE in bytes 1-3 (odometer in mile or km per byte 0) |
| Total Distance Traveled - Unit | `0x0103` | byte 0 enum (0x02 = mile, 0x01 ≈ km hypothesized) |
| Push Start Switch 1 | `0x1001` | one bit in byte block (position TBD) |
| Push Start Switch 2 | `0x1001` | one bit (position TBD) |
| Push Start Switch 3 | `0x1001` | one bit (position TBD) |
| Steering Unlock Switch | `0x1001` | one bit (position TBD) |
| Stop Light Switch | `0x1001` | one bit (position TBD) |
| IGP Relay Circuit (Outside) Monitor | `0x1003` | one bit (position TBD) |

## Decoded signal summary

| Signal | DID | Status |
|---|---|---|
| Odometer | `0x0103` (this ECU) | Decoded — bytes 1-3 (uint24 BE), unit-aware via byte 0. Note: also available via OBD-II Mode 01 PID `0xA6` from `0x7E2` (no transcoding needed if we use the standardized path). |

**No HV-side metrics** — this ECU does not own ready state, contactor state, or HV bus voltage despite the misleading name. Look for those on EV ECU (`0x7D2`) and EV Battery (`0x747`).

## Open questions

- Bit positions for the booleans in `0x1001` and `0x1003` — needs a session toggling physical inputs (brake, push start, gear) one at a time with capture running.
- DID-to-parameter mapping for `0x1004`, `0x1005`, `0x1006`, `0x1007`, `0x1008` — only mapped 8 of 25 parameters before pivoting. Worth completing if a future session needs body/ignition state — but per the analysis above, the higher-value telemetry is on other ECUs.
- Confirm `0x02 = mile` / `0x01 = km` for `0x0103` byte 0 by changing the unit in the cluster settings (or, more practically, just trust the inference).
