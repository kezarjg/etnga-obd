---
id: 0x7DA
id_type: standard
kind: diagnostic-response
counterpart: 0x7D2
target_ecu: ev
isotp: true
sources:
  - 2026-05-03_2357_techstream-connect
services_observed:
  - sid: 0x50
    name: DiagnosticSessionControl (positive response)
    confidence: high
    notes: "Returned as '06 50 SS T1 T1 T2 T2' where SS = session sub-function (0x01 default, 0x03 extended), T1/T2 = P2/P2* timing in ms. Observed with sub 0x01 (defaultSession) and 0x03 (extendedDiagnosticSession). Timing values returned: P2 = 0x0032 = 50 ms, P2* = 0x01F4 = 500 ms."
  - sid: 0x71
    name: RoutineControl (positive response)
    confidence: medium
    notes: "Observed only as '05 71 03 10 02' = positive response to 'requestRoutineResults' for routine 0x1002. The 0x1002 routine appears to be a Techstream-specific 'prepare for dynamic DID definition' step run before each 0x2C exchange. Meaning of routine 0x1002 itself unknown; called every time the Data List selection changes."
  - sid: 0x6C
    name: DynamicallyDefineDataIdentifier (positive response)
    confidence: high
    notes: "Returned as '04 6C 01 F3 0X' for define-by-identifier (sub 0x01) and '04 6C 03 F3 0X' for clear (sub 0x03). The dynamic DID slots Techstream uses on this ECU are 0xF301 and 0xF302."
  - sid: 0x7E
    name: TesterPresent (positive response)
    confidence: high
    notes: "Returned as '02 7E 00' in response to '02 3E 00'."
  - sid: 0x62
    name: ReadDataByIdentifier (positive response)
    confidence: high
    dids_observed:
      - did: 0xF190
        name: VIN
        encoding: ascii
        length: 17
        confidence: high
        kind: standard
        notes: "Multi-frame ISO-TP: First Frame `10 14 62 F1 90 XX XX XX` (total 0x14 = 20 bytes), CF#1 `21 XX XX XX XX XX XX XX` (7 bytes of VIN ASCII), CF#2 `22 XX XX XX XX XX XX XX` (last 7 bytes). Reassembled = 17-character ASCII VIN matching the vehicle VIN sticker."
        sources: [2026-05-03_2357_techstream-connect]
      - did: 0xF186
        name: ActiveDiagnosticSessionDataIdentifier
        encoding: uint8
        length: 1
        observed_value: 0x01
        confidence: high
        kind: standard
        notes: "Single Frame '04 62 F1 86 01'. 0x01 = defaultSession per ISO 14229-1 Annex C. Polled by Techstream as keep-alive while sitting on a vehicle-info screen."
        sources: [2026-05-03_2357_techstream-connect]
      - did: 0xF301
        name: DynamicSlot1
        kind: dynamic-did-slot       # NOT a fixed-meaning DID — Techstream redefines via 0x2C 01
        confidence: high
        notes: "Dynamic DID slot, NOT a static signal. Techstream creates short-lived compositions in this slot via service 0x2C 01 (defineByIdentifier), polls the slot via 0x22, then clears via 0x2C 03 when the user changes the Data List selection. Current contents change with every Data List edit; do not record observed values here."
        sources: [2026-05-03_2357_techstream-connect]
      - did: 0xF302
        name: DynamicSlot2
        kind: dynamic-did-slot
        confidence: high
        notes: "Second dynamic DID slot, same mechanism as 0xF301. Used when the Data List has too many items for a single F301 composition (or always paired with F301 — pattern not yet fully understood)."
        sources: [2026-05-03_2357_techstream-connect]

      # --- True source DIDs identified via 0x2C 01 define payloads ---
      # These are the underlying telemetry DIDs that actually carry stable signals.
      # Names where known come from Techstream's Data List label when only that item was enabled.

      - did: 0x1F0D
        name: VehicleSpeed
        encoding: uint8
        length: 1
        observed_value: 0x00
        unit: "km/h or mph (TBD — bus value 0 with car parked is consistent with both)"
        confidence: high
        kind: source
        notes: "Identified via Data-List-isolation: when only 'Vehicle speed' was enabled in Techstream, the dynamic-DID define payload was '2C 01 F3 01 1F 0D 01 01' = source DID 0x1F0D, position 1, length 1. Subsequent polls of F301 returned '04 62 F3 01 00' = 1 byte value 0. Scale and units TBD until non-zero observed."
        sources: [2026-05-03_2357_techstream-connect]

      - did: 0x10E4
        name: VehicleSpeedAtDcQuickChargingConnect
        encoding: uint16-be (likely)
        length: 2
        observed_value: 0x8000
        unit: "TBD; 0x8000 likely sentinel meaning 'no recorded value'"
        confidence: high
        kind: source
        notes: "Identified via Data-List-isolation. Define payload: '2C 01 F3 01 10 E4 01 02' = source DID 0x10E4, position 1, length 2. F301 poll returned '05 62 F3 01 80 00' (2-byte value 0x8000). Techstream displays 0.00 MPH — 0x8000 is a common sentinel for 'never recorded' in signed-16 telemetry. The value is presumably set when a DC fast charger is plugged in while the car is moving (a safety-relevant snapshot). On this car it has never happened, hence sentinel."
        sources: [2026-05-03_2357_techstream-connect]

      # --- Other source DIDs observed in 'all items' big-definition (no labels yet) ---
      # See ecus/ev.md "Source DID inventory from 'all items' Data List" for the full
      # list of 40 source DIDs and their lengths. Identifying which Techstream item
      # each maps to requires per-item Data-List-isolation captures.
---

## Notes

The diagnostic-response side from the EV ECU. Pairs with `0x7D2`.

**Dynamic DID mechanism (critical):** the EV ECU implements ISO 14229-1 service `0x2C` (DynamicallyDefineDataIdentifier). Techstream uses this to create custom compositions in slots `0xF301` and `0xF302`, then polls those slots via `0x22`. **Treat F301/F302 values as opaque without simultaneously knowing the most recent `2C 01 F30X ...` define payload.** The actual telemetry signals live in *source DIDs* like `0x1F0D`, `0x10E4`, `0x15EA`, etc.

Decoding workflow when staring at a capture:
1. Find the most recent `2C 01 F3 0X` request before the F30X poll of interest.
2. Reconstruct the multi-frame payload to recover the source-spec list (sequences of `SS SS PP LL`).
3. Map response bytes to source DIDs using the position+length spec.

**Negative responses (`0x7F`)** were not observed in this session. We may see them when probing extended-session-only DIDs from the default session, or when probing source DIDs that don't exist.

**ISO-TP:** standard framing, no addressing prefix. Tester flow-control parameters (`30 00 01`) — block size 0 (send all), STmin 1 ms.
