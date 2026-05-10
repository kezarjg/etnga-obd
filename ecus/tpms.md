---
name: Tire Pressure Monitor ECU (TPMS)
diagnostic_gateway_request_id: 0x750
diagnostic_gateway_response_id: 0x758
diagnostic_subtarget: 0x2A         # mixed-addressing prefix byte for this ECU
isotp: mixed-addressing            # ISO 15765-2 mixed addressing (1 address-extension byte + standard ISO-TP)
sessions_observed:
  - default                        # 0x01
  - extended                       # 0x03
confidence: high
verified_live_read: true            # confirmed by reading these DIDs directly from the CAN bus, without Techstream
---

# Tire Pressure Monitor ECU (TPMS)

The TPMS ECU is **not directly addressable** as a top-level OBD-II `0x7XX` CAN ID. It sits **behind the `0x750` gateway** and is reached via ISO 15765-2 **mixed addressing** — the gateway selects which downstream ECU to forward to based on the first byte of every CAN frame. For TPMS, that sub-target byte is **`0x2A`**.

See `messages/0x750.md` for the full gateway / mixed-addressing convention.

## Diagnostics

| Aspect | Value |
|---|---|
| Gateway request CAN ID | `0x750` |
| Gateway response CAN ID | `0x758` |
| Sub-target (this ECU) | `0x2A` |
| Transport | ISO 15765-2 mixed addressing — 1 address-extension byte (always `0x2A` for TPMS), then standard ISO-TP (PCI byte + payload) |
| Confirmed services | `0x10` SessionControl, `0x19` ReadDTCInformation, `0x22` ReadDataByIdentifier, `0x3E` TesterPresent |
| Sessions observed | default (`0x01`), extended (`0x03`) |
| Session timing returned | P2 = 50 ms, P2* = 500 ms |
| Stored DTCs | none (Health Check & open-ECU scans returned `5902 B9` — empty status mask) |
| **Does NOT use dynamic DIDs** | Unlike the EV ECU / EV Battery ECU, TPMS polls real DIDs directly. No `0x2C` service in use. |

## Frame format on the wire (mixed-addressing example)

Single-frame request (`0x22 0x10 0x05` = ReadDataByIdentifier 0x1005):
```
750#2A 03 22 10 05 00 00 00
       │  └─┬─────┘
       │    └── ISO-TP payload (3 bytes: 22 10 05)
       └────── sub-target = TPMS
```

Multi-frame response (13-byte payload for pressure, 8-byte for temp/position):
```
758#2A 10 LL 62 …               ← FF: PCI 10, total length LL, then payload starting with response service + DID
750#2A 30 00 01                  ← FC: continue, BS=0, STmin=1ms
758#2A 2N <up to 6 bytes>        ← CFs: PCI 2N (sequence), then payload bytes (final CF padded)
```

When parsing, **always strip the first byte (sub-target) before applying standard ISO-TP rules**.

## Slot-vs-corner architecture (CRITICAL)

The TPMS ECU exposes 5 "slots" (0–4 in the response payload, displayed by Techstream as "ID 1" through "ID 5"). **Slots are NOT corners.** The receiver assigns each learned sensor to a slot in some internal order; the *corner* a slot belongs to is reported separately via DID `0x2021`.

To know "what's the FL tire's pressure?", you must:
1. Read `0x2021` → find which slot has corner enum `0x01` (FL).
2. Read `0x1005` → take the pressure value at that same slot's position.

Observed slot-to-corner mapping on a 2024 Solterra:

| Slot | Corner (`0x2021`) | Pressure (`0x1005`) | Temperature (`0x1004`) |
|---:|---|--:|--:|
| 1 | `0x03` = RL | 41.15 PSI | 61 °F |
| 2 | `0x04` = RR | 40.65 PSI | 61 °F |
| 3 | `0x01` = FL | 41.65 PSI | 70 °F |
| 4 | `0x02` = FR | 41.15 PSI | 68 °F |
| 5 | `0x00` = none | – | – |

This slot-to-corner mapping is car-specific (assigned at sensor-learning time) and could change after a tire rotation or sensor relearn.

## DIDs identified

### `0x1005` — All Tire Pressures (HIGH CONFIDENCE)

13-byte response: `62 10 05` + 10 data bytes (5 × uint16 BE).

Each uint16 = `[status_byte][raw_pressure_byte]`. High byte = status (always `0x00` = OK so far). Low byte = raw pressure.

**Pressure formula** (PSI gauge):

> ```
> pressure_psi_gauge = (raw_low_byte × 0.25) − 7.35
> ```
>
> equivalently: `raw_low_byte = 4 × (pressure_psi_gauge + 7.35)`

- **1 LSB = 0.25 PSI** (PSI-native, NOT kPa)
- **Offset: −7.35 PSI** (= ½ atmosphere; raw `0` ↔ "Initial Value" in Techstream)

Verified to **0 PSI rounding error** across all 5 displayed values.

### `0x1004` — All Tire Temperatures (HIGH CONFIDENCE)

8-byte response: `62 10 04` + 5 data bytes (one per slot, 1 byte each).

**Temperature formula** (°C):

> ```
> temperature_C = raw_byte − 40
> ```
>
> equivalently: `raw_byte = temperature_C + 40`

- **1 LSB = 1 °C**
- **Offset: −40 °C** (the universal SAE TPMS sentinel — raw `0` represents both "−40 °C" and "−40 °F" since the two scales coincide at −40)
- raw `0` = "Initial Value" / no sensor

Verified to ±0.5 °F (limited by Techstream's integer-PSI display rounding) across all 5 displayed values.

### `0x2021` — Tire Corner Mapping (HIGH CONFIDENCE)

8-byte response: `62 20 21` + 5 data bytes (one per slot, 1 byte each).

**Corner enum:**

| Value | Corner |
|--:|---|
| `0x00` | No information / unpopulated / spare |
| `0x01` | **FL** (Front Left) |
| `0x02` | **FR** (Front Right) |
| `0x03` | **RL** (Rear Left) |
| `0x04` | **RR** (Rear Right) |

This is the standard SAE convention (FL=1, FR=2, RL=3, RR=4) and is likely consistent across Toyota TPMS implementations.

### Other supported DIDs (not yet decoded)

The capture caught Techstream briefly scanning many DIDs at low rates (~11 polls each) while paging through Data List sections. All received responses, indicating they're supported. Categories observed:

- **`0x10XX` range** (~30 distinct): probably the "tire data" category. Adjacent to confirmed `0x1004`/`0x1005`. Likely contains sensor IDs, batteries, signal strengths.
  - Specific blocks: `0x1002, 0x1003, 0x100E, 0x100F` (small group near pressure/temp); `0x1022–0x1027`, `0x1030–0x103F`, `0x1040–0x1044` (possible per-sensor detail blocks); `0x1050–0x105B` (12 DIDs — could be 3 per sensor × 4 sensors).
- **`0x11XX` range** (~10 distinct): mirrors the EV-ECU's `0x11XX`. Might be shared "vehicle state" data.
- **`0x20XX` range** (~15 distinct): "configuration / mapping" category. Confirmed `0x2021` here. Others may be learn-mode flags, sensor pairing config, threshold values.
- **`0x0103, 0x2628, 0x2363, 0x3034, 0x4090, 0x4868`**: outliers; identification, version, or ECU-config.

To extract the full list from the capture file:
```bash
grep -E ' can0 750#2A03' "$capture_log" \
  | grep -oE '22[0-9A-F]{4}' | sort -u
```

## Live read direct from CAN bus (verified 2026-05-04)

After Techstream was disconnected, all three TPMS DIDs (`0x1005`, `0x1004`, `0x2021`) were successfully read **directly from the CAN bus** using raw `cansend` + `candump` to drive the ISO-TP handshake manually. Decoded values matched Techstream's last-known display **exactly**.

Procedure (one-shot for a single DID via SSH):

```bash
ssh "$CAN_HOST" bash -s "10" "05" <<'REMOTE'   # DID hi/lo
DID_HI="$1"; DID_LO="$2"
candump -tA can0 > /tmp/cap.log 2>&1 &
CDPID=$!
sleep 0.08
# Single Frame request: ext=2A, PCI=03 (SF len 3), payload = 22 (RDBI) + DID_HI + DID_LO
cansend can0 "750#2A0322${DID_HI}${DID_LO}000000"
sleep 0.04
# Flow Control: ext=2A, PCI=30 (FC), BS=00 (send all), STmin=01 (1ms)
cansend can0 750#2A30000100000000
sleep 0.3
kill $CDPID 2>/dev/null || true
wait $CDPID 2>/dev/null || true
awk '/can0[[:space:]]+758[[:space:]]/' /tmp/cap.log
rm -f /tmp/cap.log
REMOTE
```

Reassemble (Python):

```python
# Each frame is 8 bytes: [ext_byte=0x2A, pci, ...payload]
# FF: pci high nibble 1, total length = (pci & 0x0F)<<8 | next byte; payload starts at byte 3
# CF: pci high nibble 2; payload starts at byte 2
out = bytearray()
total = None
for f in frames:
    pci = f[1]
    if (pci & 0xF0) == 0x10:
        total = ((pci & 0x0F) << 8) | f[2]
        out.extend(f[3:8])
    elif (pci & 0xF0) == 0x20:
        out.extend(f[2:8])
out = out[:total]
```

The reassembled payload starts with `62 ${DID_HI} ${DID_LO}` (positive ReadDataByIdentifier response + DID echo); the rest is the DID's data, decoded per the formulas above.

**Note:** the snippet above is a prototype, not packaged into `bin/`. If TPMS reads become a regular workflow, lift it into a helper script (and consider similar helpers for other decoded ECUs/DIDs).

## Open questions

- **Sensor IDs** (32-bit per sensor) — likely in `0x1022–0x1025` or similar 4-DID block.
- **Sensor battery / signal strength** — probably 1 byte per sensor in `0x10XX`.
- **Status nibble alternative values** — high byte of pressure uint16 has only been `0x00` (OK). Faulted sensor needed to expose other states (likely `0xFF` for missing or specific bit flags).
- **Other gateway sub-targets behind `0x750`** — `0x5F` and `0xE9` are still unidentified.
- **TPMS routines** — Toyota TPMS often exposes `0x31` routines for sensor learn / unlearn / calibration. Check the Utility menu in Techstream.
