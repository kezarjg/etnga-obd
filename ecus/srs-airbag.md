---
name: SRS Airbag ECU (Supplemental Restraint System)
diagnostic_request_id: 0x780
diagnostic_response_id: 0x788
isotp: standard
sessions_observed:
  - default                # 0x01
confidence: high
---

# SRS Airbag ECU

The Supplemental Restraint System controller. Owns airbag deployment logic, seatbelt pretensioner control, and **passenger seat occupancy detection** (the bZ4X has weight sensors on the passenger seat only — used to gate airbag deployment force, not driver-side).

> **Lower vehicle-state yield than expected.** The Data List is heavily weighted toward diagnostic data (squib resistance values for every airbag igniter, load-sensor calibration history, sensor serial numbers). Only 3 parameters surface as standard vehicle-state signals. Useful for completeness but not a major coverage win.

## Diagnostics

| Aspect | Value |
|---|---|
| Request ID | `0x780` (standard 11-bit) |
| Response ID | `0x788` (request + 8) |
| Transport | Standard ISO-TP |
| Confirmed services | `0x10`, `0x19`, `0x22`, `0x3E` |

## Vehicle-state parameters (3)

| Parameter | Sample | Notes |
|---|---|---|
| **Driver Seat Position Status** | "Vehicle Back Side" | Track position enum |
| **Passenger Seat Buckle Switch Status** | "Unbuckle" | passenger seatbelt buckled/unbuckled |
| **Occupant Detection Status** | "Child" | Passenger seat classification: Empty / Child / Adult — gates airbag force |

The SRS Airbag ECU **does not expose**:
- Airbag deployment status (probably a DTC, not a Data List parameter)
- Driver-side occupancy (no driver-side weight sensor on this platform)
- Driver seatbelt buckle (that's on the Cluster `0x7C0`)
- Rear seat occupancy (those are on Main Body `0x750/0x40`)

So for "is anyone in the car" awareness, a client would aggregate:
- Driver seat: occupancy assumed if Cluster's "Driver Buckle Switch" or shift-out-of-P
- Passenger seat: this ECU's "Occupant Detection Status"
- Rear seats × 3: Main Body's RC/RL/RR-Seat Occupant Sensor Switch

## Diagnostic-only content

### Squib resistance values (15 igniters, all measured ~2.2-2.97 Ω, all "Normal")

The SRS ECU continuously measures the resistance of every airbag igniter circuit to confirm wiring integrity. If a connection is broken, deployment would fail.

- Driver Seat Airbag (primary + 2nd stage)
- Passenger Seat Airbag (primary + 2nd stage + variable vent hole)
- Driver / Passenger Knee Airbags
- Right / Left Side 1st Seat Side-airbags
- Driver / Passenger Pretensioners (front seatbelt tensioners)
- Right / Left Curtain Shield Airbags (rooftop curtain bags)
- Right / Left 2nd Seat Pretensioners (rear seatbelt tensioners)

All resistance values are typical for pyrotechnic squibs (~2 Ω). Each has a paired "Diagnosis Result" parameter, all "Normal".

### Load sensor calibration history (passenger seat weight sensors)

Three calibration epochs are stored per sensor (Front Inner / Rear Inner / Front Outer / Rear Outer), each with both "Executed/Unexecuted" flag and "Learning Value":

- **Seat Factory Zero Point** — calibrated at the seat assembly plant
- **Vehicle Plant Zero Point** — calibrated at the car assembly plant
- **Dealer Zero Point** — calibrated post-delivery (e.g. after seat removal/replacement)

Both Outer sensors show "Not Learn Recorded" with sentinel values (-167.25 lbs) — likely **not equipped on the test vehicle** (the Solterra may use a 2-sensor instead of 4-sensor passenger seat).

### Live load sensor values

| Parameter | Sample | Notes |
|---|---|---|
| Load Sensor Total Load Value Information | 3.402 lbs | empty passenger seat — should be ~0; small reading is normal sensor noise |
| Front Inner Load Sensor | -4.569 lbs | individual sensor reading |
| Rear Inner Load Sensor | 7.876 lbs | |
| Front Outer Load Sensor | 0.000 lbs | sentinel — not equipped |
| Rear Outer Load Sensor | 0.000 lbs | sentinel — not equipped |

**The "Occupant Detection Status: Child" classification** with an empty seat is a known quirk: with low total load (~3.4 lbs, essentially noise), the classifier defaults to "Child" rather than "Empty" as a safety bias. From a downstream-consumer perspective, this means **"Empty" might never be reported** even when the seat is genuinely unoccupied — the classifier may always say at least "Child". Worth confirming by sitting in the seat with empty hands vs. with weight to see what threshold "Adult" requires. Treat "Child" as ambiguous between "small occupant" and "empty".

### Sensor serial numbers (manufacturing provenance, low-value telemetry)

Each load sensor exposes 6 fields: production year, month, date, sensor number, line number, serial number. Front Inner sensor was made 2024-04-23, Rear Inner sensor 2024-04-14. Outer sensors all zero (not equipped, confirms hypothesis).

## Open questions

- **Airbag deployment state** — not a Data List parameter, but should be readable as a DTC after deployment. For "has the car been in a crash" detection, a client would need to read DTCs from this ECU periodically, OR find a CAN broadcast that signals deployment in real time. Worth a separate investigation.
- **Whether "Empty" is ever reported** — the empty-seat-classified-as-Child behavior may persist unless the seat sensor is properly calibrated, or it may be design (always classify as at least Child to ensure airbag doesn't deploy at full force on a small occupant).
- **Driver seat track position enum values** — "Vehicle Back Side" suggests the rear-most slider position. There's likely a "Vehicle Front Side" / "Middle Position" / etc. Worth observing during driver-seat memory recall to enumerate.
