---
name: Combination Meter (Instrument Cluster)
diagnostic_request_id: 0x7C0
diagnostic_response_id: 0x7C8
isotp: standard
sessions_observed:
  - default
sources:
  - 2026-05-08_1955_ecu-mapping-marathon
  - 2026-05-08_2049_combo-meter-data-list
confidence: high
---

# Combination Meter (Cluster)

Toyota's "Combination Meter" — the instrument cluster ECU. Direct OBD-II addressing (no gateway), standard ISO-TP, single-DID polling (no dynamic DIDs).

> **History**: until 2026-05-08, this file documented `0x7E2/0x7EA` as the cluster, based on (a) `0x7E2` owning OBD-II Mode 01 PID `0xA6` (odometer), and (b) `0x7E2` being a typical cluster ID on legacy Toyota platforms. The mapping marathon disproved this — Techstream's "Combination Meter" entry connects to `0x7C0`. The odometer at OBD-II Mode 01 PID `0xA6` is still served from `0x7E2`, but `0x7E2` is some other ECU (likely a virtual gateway exposing the standardized OBD-II Mode 01 surface). The cluster proper is here at `0x7C0`.

## Diagnostics

| Aspect | Value |
|---|---|
| Request ID | `0x7C0` (standard 11-bit) |
| Response ID | `0x7C8` (request + 8) |
| Transport | Standard ISO-TP (no address-extension byte) |
| Confirmed services | UDS `0x10`, `0x19`, `0x22`, `0x3E` |

## Data List parameter inventory (Techstream, 40+ parameters)

```
Total Distance Traveled                    (value)
Total Distance Traveled - Unit             (mile/km)
ODO/TRIP Change Switch                     (ON/OFF)
Light Control Switch (UP)                  (ON/OFF)
Light Control Switch (DOWN)                (ON/OFF)
Multi Switch (Up/Down/Left/Right/Enter/Back) (ON/OFF) — 6 booleans
+B Voltage                                 (volts)        ← decoded
Vehicle Speed Meter                        (MPH or KPH)   ← partial
Driver Buckle Switch                       (Fastened/Not Fastened)
Hazard Flasher Switch                      (ON/OFF)
Ambient Temperature (Celsius)              (°C)           ← decoded
Ambient Temperature (Fahrenheit)           (°F)           ← decoded
Lighting System                            (With/Without)
Back Door System                           (With/Without)
Integrated value for Maintenance           (mile/km)      ← decoded
TPMS / VSC / LDA / PCS / ICS / Radar Cruise / Road Sign Assist System (With/Without — static config)
HV/EV System Indicator                     (%)            ← partial
ODO Display Time After IG OFF Adjust       (seconds)
Tail Remind Buzzer Function                (ON/OFF)
Driver/Passenger/Rear-{Right,Center,Left} Seatbelt Warning Buzzer Function (ON/OFF — 5 booleans)
Lane Change Flashing Times Adjust          (count)
Flasher Sound Volume Adjust                (Low/Medium/High)
Vehicle Speed Meter (duplicate)            ← shows up twice in the Data List
Nonvolatile Memory Status                  (Normal/...)
Reverse Buzzer Setting                     (Continual/...)
```

**Notable absence:** there is **no "Cruising Distance" / "Distance to Empty" / "Range Remaining"** parameter in this Data List. Confirms the EM datalist analysis (`docs/references/em-datalist-findings.md`) — range-to-empty is not exposed as a DID by design. It's either computed in the cluster from internal state (no diagnostic surface) or comes off a CAN broadcast (passive listening required to find).

## DIDs decoded (2026-05-08_2049_combo-meter-data-list)

Mapped via single-parameter isolation in Techstream Data List.

| DID | Length | Parameter | Encoding | Confirmed value |
|---|---|---|---|---|
| `0x1021` | 1 byte | **+B Voltage** | uint8 × 0.1 V (range 0–25.5 V) | `0x84` = 13.2 V; `0x85` = 13.3 V (toggling at idle, matches display) |
| `0x1041` | 1 byte | **Vehicle Speed Meter** | TBD — at 0 we can confirm the zero point only. Likely uint8 with implicit unit (raw MPH or km/h) or × scale factor. | `0x00` = 0 MPH/KPH |
| `0x1141` | 2 bytes | **Ambient Temperature** | byte 0 = (T_C × 2) + 80 → `T_C = (b - 80) / 2`, 0.5 °C resolution; byte 1 = T_F + 40 → `T_F = b - 40`, 1 °F resolution | `0x6E 63` = 15.0 °C / 59 °F (cross-check: 15 °C → 59 °F ✓) |
| `0x12A1` | 1 byte | **Integrated value for Maintenance** | uint8 × 100 (miles, possibly km — depends on locale) | `0x1E` = 30 → 3000 mile (matches display). Direction TBD: counts up (miles since service) or down (miles to next). |
| `0x1641` | 2 bytes | **HV/EV System Indicator** | TBD — at 0 % can't pin scale. Likely the dashboard power-flow gauge (regen-left / power-out-right). | `0x00 00` = 0 % at idle (car not in Ready mode). Needs drive-cycle capture to decode. |

## OVMS mappings

| OVMS metric | DID | Encoding | Status |
|---|---|---|---|
| `v.b.12v.voltage` | `0x7C0` `0x1021` | uint8 × 0.1 V | ✅ Decoded |
| `v.e.temp` | `0x7C0` `0x1141` byte 0 | (b - 80) / 2 = °C | ✅ Decoded (backup to HVAC `0x7C4` `0x1002`) |
| `v.p.speed` | `0x7C0` `0x1041` | TBD | ⚠ Partial — needs drive cycle |
| `v.e.serv.range` | `0x7C0` `0x12A1` | uint8 × 100 (mi) | ⚠ Direction TBD |
| `v.b.power` (or similar) | `0x7C0` `0x1641` | TBD | ⚠ Needs Ready mode |
| `v.b.range.est` (USER-STATED GAP) | **not exposed as a DID on this ECU** | — | ❌ Confirmed absent — must be derived in module or sniffed from broadcast |

## Open questions

- **Vehicle Speed encoding** — mph vs km/h vs scaled? Capture during a drive cycle and read the same DID at known speeds (5/10/15 mph) to nail the multiplier.
- **HV/EV System Indicator encoding and meaning** — dashboard power gauge at 0 % when parked. Needs Ready mode + light accelerator/brake to see the dynamic range. If int16 with negative for regen, it may map to `v.b.power` directly.
- **Integrated for Maintenance direction** — counts up or down? Compare value before and after a known drive (or just compare across two sessions a week apart with miles in between).
- **Range estimate** — confirmed not in this Data List. Next step: passively monitor the bus during a drive cycle and look for broadcast frames whose value tracks the displayed range estimate.
- **Why does `0x7E2` own OBD-II Mode 01 PID `0xA6`?** That's the odometer; the cluster owns it via Toyota DID `0x0103` over at PSC (`0x750/0xE9`). On `0x7E2` it shows up as the standardized OBD-II Mode 01 PID. `0x7E2` is probably a virtual ECU that aggregates standardized OBD-II surface signals — worth a dedicated session.
