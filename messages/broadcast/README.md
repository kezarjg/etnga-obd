# messages/broadcast/

Per-CAN-ID files for periodic broadcast frames. These are the messages that will eventually populate a DBC file.

Bit numbering follows DBC Intel/little-endian convention — see `../../docs/conventions.md`. Pick once, never deviate.

Every signal carries a `confidence` rating. Unknown bytes get explicit `Unknown_bN` signal entries — documenting what's been looked at but not yet decoded is as valuable as documenting what has been.

**First targets to investigate** (observed live on the OBD-II bus during initial bring-up, 2026-05-03):
- `0x45A` — repeating payload `5A 00 A6 02 0F 0F 0F 0F`
- `0x4E0` — repeating payload `24 00 A6 02 00 00 00 00`
