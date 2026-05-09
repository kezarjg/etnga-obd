# OBD-II / UDS / ISO-TP primer

A short refresher for context when staring at captures. Not a tutorial — see references at the bottom for that.

## CAN frame basics

Each frame on the bus has a CAN ID (11-bit "standard" or 29-bit "extended") and 0–8 data bytes. `candump -tA -L can0` prints them as:

```
(1746301823.412) can0 7E0#0211900000000000
```

`(timestamp) iface ID#hexpayload`. This is the canonical can-utils log format and parses cleanly with python-can / cantools.

## OBD-II IDs you'll see constantly

| ID | Direction | Meaning |
|---|---|---|
| `0x7DF` | tester → all ECUs | Functional broadcast diagnostic request |
| `0x7E0` | tester → ECU 1 | Physical request to ECU 1 (commonly Engine) |
| `0x7E8` | ECU 1 → tester | Physical response from ECU 1 |
| `0x7E1` | tester → ECU 2 | Physical request to ECU 2 (commonly Transmission/HV) |
| `0x7E9` | ECU 2 → tester | Physical response from ECU 2 |
| `0x7E2`–`0x7E7` | tester → ECU 3..8 | Further physical requests |
| `0x7EA`–`0x7EF` | ECU 3..8 → tester | Further physical responses |

(Pattern: response ID = request ID + 8.) On a Toyota, additional diagnostic ID pairs exist for less common ECUs — discover and document in `ecus/` as we encounter them.

## ISO-TP framing (transport over CAN)

A single CAN frame holds 8 bytes; UDS messages are often longer. ISO-TP (ISO 15765-2) chunks them:

| First nibble | Frame type | Layout |
|---|---|---|
| `0` | Single Frame | `0L XX XX XX XX XX XX XX` — `L` = payload length (1–7), then payload |
| `1` | First Frame | `1L LL XX XX XX XX XX XX` — `LLL` = 12-bit total length, then first 6 bytes of payload |
| `2` | Consecutive Frame | `2N XX XX XX XX XX XX XX` — `N` = sequence counter (1, 2, 3, ..., wraps to 0 after F) |
| `3` | Flow Control | `3F BS ST 00 00 00 00 00` — `F` = flow status (0=continue, 1=wait, 2=overflow); `BS` = block size; `ST` = separation time (ms or 100µs) |

When you see `1014621904E1234567` in a response, that's a First Frame: total length `0x014` = 20 bytes, then payload starts `62 19 04 E1 23 45 67...`. Continued by Consecutive Frames `21 ...`, `22 ...`, `23 ...`.

## UDS service IDs Techstream uses heavily

| SID | Name | Notes |
|---|---|---|
| `0x10` | DiagnosticSessionControl | Sub-functions: `01`=default, `02`=programming, `03`=extended, `40`=safety system. Toyota also uses proprietary sub-functions in the `0x80–0xFF` range. |
| `0x11` | ECUReset | |
| `0x14` | ClearDiagnosticInformation | Clear DTCs |
| `0x19` | ReadDTCInformation | |
| `0x22` | ReadDataByIdentifier | The bread and butter — read a DID (16-bit identifier). Response SID = `0x62`. |
| `0x27` | SecurityAccess | Seed/key challenge before privileged ops |
| `0x28` | CommunicationControl | |
| `0x2E` | WriteDataByIdentifier | |
| `0x2F` | InputOutputControlByIdentifier | Actuator tests |
| `0x31` | RoutineControl | Trigger built-in routines |
| `0x3E` | TesterPresent | Keep-alive while a non-default session is active |
| `0x85` | ControlDTCSetting | |

Positive response SID = request SID + `0x40`. Negative response = `0x7F <requested_sid> <NRC>`. Common NRCs: `0x10` general reject, `0x11` service not supported, `0x12` sub-function not supported, `0x22` conditions not correct, `0x33` security access denied, `0x78` request correctly received, response pending.

## Toyota proprietary services

Toyota uses additional SIDs outside the standard UDS range. These will get filled in as we observe them:

- `0xA*` and `0xB*` ranges are commonly Toyota-specific.
- Some services overlap with old KWP2000 conventions.

When we see an unknown SID, log it in this file with the request/response patterns we observed.

## OBD-II Mode 01 vs UDS

The classic OBD-II "modes" (Mode 01 = current data, Mode 03 = stored DTCs, etc.) are functionally a parallel protocol with its own short request format:

```
7DF#02 01 0C 00 00 00 00 00     <- request engine RPM (mode 01, PID 0C)
7E8#04 41 0C 1A F8 00 00 00     <- response: 0x41 = 0x01+0x40, PID 0C, value 0x1AF8
```

Mode 01 PIDs are publicly documented (Wikipedia has a complete list). UDS DIDs are mostly manufacturer-specific.

## References

- ISO 15765-2 (ISO-TP)
- ISO 14229-1 (UDS)
- can-utils: https://github.com/linux-can/can-utils
- python-can: https://python-can.readthedocs.io
- cantools: https://cantools.readthedocs.io
