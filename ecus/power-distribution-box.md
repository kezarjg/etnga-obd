---
name: Power Distribution Box
diagnostic_request_id: "0x750"
diagnostic_response_id: "0x758"
gateway_sub_target: "0x4D"
em_reference: F143/F144 (per EM, in passenger kick panel area)
isotp: mixed-addressing
sessions_observed:
  - default                # 0x01
confidence: high
---

# Power Distribution Box

Behind the gateway at `0x750` sub-target `0x4D`. Per Toyota EM topology, this is **F143/F144 in the passenger-side kick panel area** (same area F39 sits in).

Small Data List (~22 parameters) — just a handful of relay/fuse-level circuits and a power-bus status panel. **But it owns the 3 rear seatbelt buckle switches**, which closes the seatbelt picture for the rear seats.

## Diagnostics

| Aspect | Value |
|---|---|
| Request ID | `0x750` with sub-target byte `0x4D` |
| Response ID | `0x758` with sub-target byte `0x4D` echoed |
| Transport | ISO-TP mixed-addressing |
| Confirmed services | `0x10`, `0x19`, `0x22`, `0x3E` |

## Functional content (per Data List, 2026-05-08)

### Power bus state (4 buses monitored)

| Parameter | Sample | Notes |
|---|---|---|
| Power Supply Voltage | 13.4 V | this ECU's 12V supply |
| +BA Status Signal | ON | always-on battery feed |
| ACC Status Signal | ON | accessory bus |
| IGR Status Signal | ON | ignition (run) bus |
| IGP Status Signal | ON | ignition (positive) bus |
| +BA Output | ON | output to downstream |

### Rear defogger circuit (with current monitoring + fuse health)

| Parameter | Sample | Notes |
|---|---|---|
| Rear Defogger Input Signal | OFF | command from cluster/HVAC |
| Rear Defogger Output Signal | OFF | actual relay output |
| Rear Defogger Output Current | 0.0 A | live current draw — useful for confirming heater grid is working |
| Rear Defogger Fuse Shut Off Status | OFF | fuse open flag |
| Rear Defogger Fuse Shut Off Count | 0 | lifetime overcurrent events |

### Back-up light circuit (with current monitoring + fuse health)

| Parameter | Sample | Notes |
|---|---|---|
| Back-up Light Input Signal | OFF | command (gear in R) |
| Back-up Light Output Signal | OFF | actual relay output |
| Back-up Light Output Current | 0.0 A | live current draw |
| Back-up Light Fuse Shut Off Status | OFF | fuse open flag |
| Back-up Light Fuse Shut Off Count | 0 | lifetime overcurrent events |

### Tail light relay state

| Parameter | Sample | Notes |
|---|---|---|
| Tail Light Internal Relay Input Signal | ON | command from Headlight Control |
| Tail Light Internal Relay Output Signal | ON | actual output (matches Headlight Control's Clearance Light = ON) |

### **Rear seatbelt buckle switches** (the headline content of this ECU)

| Parameter | Sample | Notes |
|---|---|---|
| Rear Seat RH Buckle Switch Status | Unset | rear-right belt |
| Rear Seat Center Buckle Switch Status | Unset | rear-center belt |
| Rear Seat LH Buckle Switch Status | Unset | rear-left belt |

**Combined with the other ECUs**, the complete 5-seat seatbelt picture is available across the diagnostic surface:

| Seat | Source ECU | Parameter |
|---|---|---|
| Driver | Cluster `0x7C0` | "Driver Buckle Switch" (observed value: "Not Fastened") |
| Front Passenger | SRS Airbag `0x780` | "Passenger Seat Buckle Switch Status" |
| Rear Left | **Power Distribution Box `0x750/0x4D`** | "Rear Seat LH Buckle Switch Status" |
| Rear Center | **Power Distribution Box `0x750/0x4D`** | "Rear Seat Center Buckle Switch Status" |
| Rear Right | **Power Distribution Box `0x750/0x4D`** | "Rear Seat RH Buckle Switch Status" |

## Open questions

- **Why are these specific circuits (rear defogger, back-up light, tail light) here vs. elsewhere?** Likely because the Power Distribution Box has the high-current relays and fuses for these high-load circuits — separating them physically from the lower-current loads on Main Body / Headlight Control. Toyota architectural choice.
- **No live current monitoring on tail light?** Rear defogger and back-up light expose Output Current (in Amps), but Tail Light only exposes Input/Output Signal (boolean). Likely the tail light is an LED chain with low current and isn't worth instrumenting.
- **F143 vs F144 distinction** — the EM lists both. May be left/right or main/sub PDBs. Worth checking if there's a sibling sub-target for the other one.
