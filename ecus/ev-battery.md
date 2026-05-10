---
name: EV Battery ECU (Toyota official: Battery ECU, k3)
diagnostic_request_id: 0x747
diagnostic_response_id: 0x74F
toyota_name: Battery ECU
em_reference: k3
physical_bus: P-CAN-FD          # per EM39J0U/system/MPX_P_*.pdf
isotp: standard
sessions_observed:
  - default                # 0x01
  - extended               # 0x03 — required for 0x2C
techstream_parameter_count: ~250
confidence: high
---

# EV Battery ECU (Toyota: Battery ECU)

Per the Toyota Electrical Manual (EM39J0U), this is officially **k3 — Battery ECU**, sitting on the **P-CAN-FD bus**. The high-voltage battery management ECU on the Solterra. Direct OBD-II addressing — no gateway extension byte.

> **Note**: Earlier versions of this file claimed `0x745` was an alias diagnostic identity for this same ECU ("high-level customer view"). The 2026-05-08 mapping marathon disproved that — `0x745` is a **separate ECU** (Plug-in Charge Control), not an alias. Both ECUs expose SOC and charging state but with different views/scaling. See `plug-in-charge-control.md`.

Distinct from the **EV ECU** (`0x7D2/0x7DA`, propulsion/inverter/motor control). On the Solterra these are two separate diagnostic targets.

## Diagnostics

| Aspect | Value |
|---|---|
| Request ID | `0x747` (standard 11-bit) |
| Response ID | `0x74F` (request + 8) |
| Transport | Standard ISO-TP — no extended-addressing prefix |
| Confirmed services | `0x10` SessionControl, `0x19` ReadDTCInformation, `0x22` ReadDataByIdentifier, `0x2C` DynamicallyDefineDataIdentifier, `0x31` RoutineControl, `0x3E` TesterPresent, `0xAB` (Toyota proprietary) |
| Sessions observed | default (`0x01`), extended (`0x03`) |
| Session timing returned | P2 = 50 ms, P2* = 500 ms (`06 50 SS 00 32 01 F4`) |

## Data List uses dynamic DIDs

Techstream uses UDS service `0x2C 01` (DynamicallyDefineDataIdentifier By Identifier) to compose source DIDs into dynamic-DID slots `0xF301` and `0xF302`, then polls them with `0x22 F3 01` / `0x22 F3 02`.

**Format of the `0x2C 01` definition** (per ISO 14229):

```
2C 01 <DDDID hi> <DDDID lo> [<SDID hi> <SDID lo> <position> <size>]+
```

- `position` is **1-indexed** byte offset within the source DID's data
- `size` is bytes to copy
- Multiple source DIDs concatenate into the dynamic DID's response

**Always-bundled fault sidecar:** every dynamic DID definition observed in 2026-05-08 included `0x1F01` byte 1 (1 byte) at the end. Likely a fault/health aggregate that Techstream watches alongside whatever specific signal is being read. Worth a direct `0x22 1F 01` read to dump the full surface.

**Direct DID polling also works** — confirmed by the public ABRP bZ4X/Solterra OBD config (https://abetterrouteplanner.com), which reads source DIDs (e.g., `0x1F9A`, `0x1814`, `0x1D3E`) directly via `0x22` without setting up a dynamic DID. The `0x2C` mechanism is a Techstream optimization (one big poll vs. many small polls); any client can ignore it and read source DIDs directly.

## DIDs decoded (2026-05-08, EV Battery Data List snapshot)

Mapped via single-parameter Data List isolation. Each row gives the source DID location (revealed by Techstream's `0x2C 01` setup) and the encoding (deduced from the response data + displayed value).

| Signal | Source DID | Position (1-indexed) | Size | Encoding | Notes / verified value |
|---|---|---|---|---|---|
| **Pack SOC** | `0x1F5B` | 1 | 1 byte | `byte × 100 / 255` → % | `0xE6` = 230 → 90.196% (display 90.19%) ✓ |
| **Pack voltage** | `0x1F9A` | 3 | 2 bytes | uint16 BE × 1/64 → V (LSB ≈ 15.625 mV) | `0x6200` = 25088 → 392.00 V ✓ |
| **Pack current** | `0x1F9A` | 5 | 2 bytes | int16 BE × 0.1 → A | `0x0018` = 24 → 2.4 A ✓ **Sign confirmed 2026-05-09 via drive cycle: `+` = discharge** (peak +312 A under acceleration, −62 A during regen) |
| **Ready ON** | `0x1076` | 2 | 1 byte | bool (1 = Ready ON, 0 = OFF) | `0x01` = ON ✓ |
| **Per-cell voltage [1..96]** | `0x182E` | 1 (then 3, 5, …) | 2 bytes per cell | uint16 BE × 5/65535 → V (LSB ≈ 76.3 µV) | `0xD19D` = 53661 → 4.0946 V (display 4.09 V) ✓ — full 192-byte block returns all 96 cells in cell order |
| **Per-sensor cell temperature [1..24]** | `0x1814` | 1 (then 3, 5, …) | 2 bytes per sensor | byte 0 of slot − 50 → °C (1 °C resolution) | `0x42` = 66 → 16 °C (display 60.8 F = 16.0 °C) ✓ — full 48-byte block. byte 1 of each slot purpose TBD. |
| **Battery coolant temperature** | `0x1848` | 1 | 2 bytes | byte 1 − 50 → °C; byte 0 = sensor voltage × 1/80 V | `0x87 0x44` → 0.5×voltage 1.69 V + 18 °C (display 64.94 F = 18.30 °C, 1.69 V sensor) ✓ |
| Odometer | `0x0103` | 1 | 4 bytes | byte 0 = unit (0x02 = mile), bytes 1-3 = uint24 BE odometer | `02 00 61 A8` = 25000 mile ✓ (synthetic example) — same DID as Power Source Control |

## Pack-level structure of `0x1F9A`

Implied from the voltage and current decodes: `0x1F9A` is a "pack essentials" struct with at least the following layout:

| Bytes (1-indexed) | Field | Encoding |
|---|---|---|
| 1-2 | unknown | — |
| 3-4 | Pack voltage | uint16 BE × 1/64 V |
| 5-6 | Pack current | int16 BE × 0.1 A |

Worth a direct `0x22 1F 9A` read (no dynamic DID setup needed) to see the complete struct. Likely also has additional bytes for pack power, sub-current sensors, etc. — the snapshot showed multiple "Battery Current" parameters (main, Sub, IBL, Driving Control) that may all live here at different positions.

## Cell voltage block `0x182E` — confirmed structure

192 bytes = 96 × 2-byte uint16 BE values, in cell-numbered order (Cell N at byte offset 2N-2 zero-indexed, or position 2N-1 1-indexed). Encoding: voltage = `raw × 5 / 65535` V (5 V full-scale ADC mapping, LSB ≈ 76.3 µV).

To poll all 96 cells: single `0x22 18 2E` request, parse 96 × 2 bytes from the multi-frame response.

## Cell temperature block `0x1814` — confirmed structure

48 bytes = 24 × 2-byte slots. byte 0 of each slot encodes T_C with offset 50 (`T_C = byte − 50`, range -50 to +205 °C, 1°C resolution). byte 1 of each slot purpose unknown — possibly Q8.8 fractional, possibly fault/balancing flag.

To poll all 24 sensors: single `0x22 18 14` request.

## Snapshot-time ECU state (2026-05-08, car in Ready mode)

Captured for context — confirms which parameters are realistic to expect / decode:

- Ready Signal: ON
- Pack: 392 V, 2.4 A, 90.19% SOC
- HV bus: VL 391 V / VH 388 V
- Power limits: WIN -24.22 kW (regen), WOUT 180 kW (drive)
- Contactors: SMRG ON, SMRB ON, SMRP OFF (pre-charge done)
- All 96 cells at 4.09 V (well-balanced)
- All 24 cell temps in range 60-62 °F
- Coolant: 65 F, heater off, fan off
- AC charging relays: all OFF (not charging)

## Other parameters in the Data List (~250 total) — not yet isolated

Among the unmapped, several are high-priority for general telemetry or for SoH derivation:

- **Cell Maximum/Minimum Voltage Up to 1 trip before** — direct SoH proxy (cell spread)
- **Cell Internal Resistance × 96** — direct SoH input when read under load
- **Stack 1-4 / Block 1-4 Cell Average Voltage** — pack-balance views
- **WIN / WOUT Control Limit Power** — load-derived SoH proxies
- **SMRG / SMRB / SMRP Control Status** — contactor states
- **Smoothed Value of BATT Voltage** — filtered 12V voltage (different from raw `0x1021` from cluster)
- **Distance from DTC Cleared** — useful trip metric
- **Hybrid/EV Battery SOC just after IG-ON / Maximum / Minimum** — SOC drift indicators

## Active Tests / Routines (2026-05-08, Battery active-cooling test)

Active Tests on this ECU use **two distinct UDS mechanisms** depending on the test type:

### `0x31` RoutineControl — for complex test sequences

Standard ISO 14229 service. Pattern:

| Sub-function | Request | Response |
|---|---|---|
| `0x01` startRoutine | `31 01 RR RR` | `71 01 RR RR` (no payload) |
| `0x02` stopRoutine | `31 02 RR RR` | `71 02 RR RR` (no payload) |
| `0x03` requestRoutineResults | `31 03 RR RR` | `71 03 RR RR <status>` (1 byte: `0x00` inactive, `0x01` active) |

Techstream **polls `0x31 03`** continuously while the routine is active to monitor status (and likely to refresh keep-alive).

### `0x2F` InputOutputControlByIdentifier — for direct relay/output overrides

Standard ISO 14229 service. Pattern:

| Action | Payload (8 bytes) | Note |
|---|---|---|
| Apply control | `2F <DID> 03 <state> <mask>` | `0x03` = shortTermAdjustment; state and mask are 2 bytes each |
| Release control | `2F <DID> 00` (predicted) | `0x00` = returnControlToECU |

Techstream **repeats the command every ~4 sec** to refresh the override (shortTermAdjustment has a server timeout).

### Confirmed routines and control DIDs

| Test | Mechanism | RID / Control DID | Notes |
|---|---|---|---|
| **Hybrid/EV Battery Water Cooling System** | `0x31` RoutineControl | RID `0x1124` | Per RM: "activate the battery coolant water pump assembly continuously". Engages the **coolant pump only** — the AC chiller engagement seen during testing (+1.0 kW A/C consumption) was the BMS *autonomously* responding to detected coolant flow. Cooled coolant from 64.94 → 53.42 F (−11.5 F) in ~2 min. Runs until explicit Stop. |
| **Hybrid/EV Battery Heater Relay** | `0x2F` IOControl | DID `0x2806` | Per RM: "activate the EV battery heater continuously" — drives the BATT HTR NO. 1 relay which powers the **Battery Coolant Heater Assembly (A36)**, not a PTC strapped to the cells. ON: `2F 28 06 03 00 01 00 01`. OFF: `2F 28 06 03 00 00 00 01`. Heater 1 sensor (inside the heater body) hit 153 F local-spot temp in ~2 min ON; thermal lag after OFF. |

### Component architecture (from RM DTC P091E72/P091E73 inspection steps)

Heater path:
```
Battery ECU (k3) ── pin BHRB (k3-11) ──┐
                  └── pin BHRG (k3-17) ─┴── BATT HTR NO. 1 Relay (motor compartment relay block)
                                            └── Battery Coolant Heater Assembly (A36, two internal elements: BATTERY HEATER 0 and BATTERY HEATER 1)
                                                └── coolant loop
```

The heater assembly has **two internal heating elements**, which is why the Data List has "Heater 1 Temperature" specifically — there's almost certainly a "Heater 0 Temperature" reachable via direct DID polling that wasn't exposed during the Data List sweep.

Cooling pump path: separately driven by the Battery ECU; physical pump is the "battery coolant water pump assembly" (separate component code, not pinned). When the pump runs, the BMS evaluates pack temperature vs. setpoints and may autonomously engage the AC chiller path on its own initiative.

### Restrict conditions (per RM Active Test table)

The Battery ECU enforces preconditions before accepting these tests. If conditions aren't met, expect a UDS negative response (NRC `0x22` conditionsNotCorrect or similar) instead of a positive Start ACK.

**Heater Relay** (`0x2F 28 06`):
- Ignition switch ON (Ready mode is sufficient)
- EV battery system normal (no active fault)
- Auxiliary 12V battery ≥ 9.5 V
- EV battery temperature in normal range (won't run if cells already hot)

**Water Cooling System** (`0x31 01 11 24`):
- Ignition switch ON
- EV system normal
- Not in maintenance mode
- Other Active Tests not being performed
- Auxiliary 12V battery ≥ 9.5 V

Note the "other Active Tests not being performed" guard for Water Cooling. Toyota's documented expectation is one test at a time; in practice the BMS auto-runs the chiller alongside the pump (observed empirically), so the constraint may be enforced only between user-invoked tests, not between user-invoked and BMS-internal.

### Safety NOTICE from RM (applies to any active-control consumer)

> "It is necessary to use caution, because if the tester DLC connector becomes disconnected or if a communication error occurs during an Active Test, the vehicle could become inoperative (the READY light may go off)."

Implication for any client running these tests autonomously (e.g. for preconditioning): the client **must** maintain a reliable command path and **must** issue the corresponding Stop (`0x31 02 11 24`) or returnControlToECU (`0x2F 28 06 00`) before any condition that could interrupt the connection. A keep-alive watchdog with bounded test duration is essential.

### Heuristic — which mechanism for which test

- Tests that have a Start/Stop button in Techstream → likely `0x31` RoutineControl
- Tests that have an On/Off toggle in Techstream → likely `0x2F` IOControl with shortTermAdjustment
- The DID for `0x2F` tests is often the same as the corresponding Data List read DID (e.g., `0x2806` Heater Relay status is also where you write to override it)

### Notes on active control

To actively control the battery (e.g., trigger a balance or capacity routine), a client can issue `0x31 01 RR RR` directly — no `0x2C` setup or session escalation needed beyond default session, based on what Techstream did. For monitoring whether a routine *would* succeed, `0x22 28 06` reads the current relay state without invoking control.

## Open questions

- **Sign convention for pack current**: positive observed at idle in Ready mode; need to charge the car and confirm negative when charging.
- **Vehicle Speed encoding** (`0x1F0D` per F302 inventory) — same DID as on EV ECU.
- **Direction of "Integrated value for Maintenance"** (counts up since service or down to next).
- **Cell temperature byte 1** — what does the second byte of each `0x1814` slot encode?
- **`0x1F01` fault sidecar** — what does the full DID contain? Direct read would clarify.
- **Capacity / SoH via UTILITY routine** — Toyota's Battery Diagnosis routine (`0x1125`?) reportedly returns binary Normal/Replace, not %. Re-run after a full drive cycle to see if it gives a useful judgment.
- **`0x1D3E` battery capacity DID** (from the public ABRP bZ4X/Solterra OBD config) — Techstream Data List doesn't expose a "Capacity" parameter; was `0x1D3E` a dead-end or is it a routine output? Worth re-investigating.
- ~~**Where is the actual OBC (A33)?**~~ — **Resolved 2026-05-08**: A33 is `0x745` (the same ECU as Plug-in Charge Control). See `plug-in-charge-control.md`. Toyota integrates session control + OBC hardware + DC-DC into one ECU; the Techstream "Plug-in Control" name disguised the full role.
