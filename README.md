# etnga-obd

OBD-II reverse-engineering knowledge for **Toyota eTNGA-platform BEVs** —
**Toyota bZ4X**, **Subaru Solterra**, and **Lexus RZ**. These three vehicles
share the same EV-dedicated platform and largely the same diagnostic surface,
so findings here generally apply to all three. Variant-specific differences
are called out where they're known.

This is a reference for anyone building a vehicle integration (ABRP,
EVNotify, custom dashboards) or doing their own RE. It documents the OBD-II
diagnostic surface (UDS over ISO-TP) plus the small handful of broadcast CAN
frames visible on the OBD-II bus.

## Status

Active reverse-engineering project. Coverage as of 2026-05-10:

- **~36 of 39 ECUs mapped** — 14 directly addressable on OBD-II plus 22 sub-targets behind the gateway at `0x750`. Three top-level OBD-II IDs in the standardized range remain unidentified.
- **~12 high-priority telemetry signals decoded with verification** — pack SOC/voltage/current, per-cell voltages (×96), per-sensor cell temperatures (×24), 12V auxiliary battery, vehicle speed, gear, odometer, TPMS pressures/temps/corner mapping, and others.
- Charging-state machines, dynamic-DID composition (`0x2C 01`), and gateway mixed-addressing all decoded.
- SOH / pack capacity / range-to-empty are confirmed *not* exposed as DIDs anywhere on the diagnostic surface — they must be derived externally or read from broadcast frames during driving.

## Where to start

- **OBD/UDS basics on this platform**: `docs/obd-and-uds-primer.md`
- **ECU inventory** (top-level IDs and gateway sub-targets): `ecus/README.md`
- **Gateway addressing scheme** (mixed-addressing format, sub-target list): `messages/0x750.md`
- **Decoded battery signals** (SOC, voltages, temps): `ecus/ev-battery.md`
- **Decoded vehicle-supervisor signals** (speed, gear, 12V health, motors): `ecus/ev.md`
- **TPMS** (pressures, temps, slot-vs-corner): `ecus/tpms.md`

## What's here

- **`ecus/`** — per-ECU surface maps. For each addressable ECU on the OBD-II
  bus: diagnostic IDs, Toyota's name, observed DIDs, decoded formulas with
  confidence ratings, and Techstream cross-references where applicable.
- **`messages/broadcast/`** — broadcast CAN frame meanings (e.g., `0x45A`
  Central Gateway heartbeat, `0x4E0`).
- **`messages/`** — diagnostic-ID surface docs (request/response IDs used by
  UDS over ISO-TP).
- **`docs/obd-and-uds-primer.md`** — quick primer on UDS-over-OBD-II as it
  applies to this platform. Useful if you're new to ISO-TP / DID-style reads.
- **`docs/techstream-research.md`** — Toyota Techstream notes relevant to
  Solterra/bZ4X RE.
- **`docs/conventions.md`** — how the per-ECU files are structured (front-matter,
  confidence levels, evidence requirements).
- **`docs/references/`** — community hardware research (F39 accessory connector,
  N20 roof power tap).
- **`bin/`** — small bash helpers for capturing CAN traffic via a remote
  capture host, parameterized via the `$CAN_HOST` env var.

## Scope

**In scope.** Anything readable from the OBD-II port without bus injection:
diagnostic reads (UDS DIDs), broadcast frames passively observed, Techstream
parameter cross-references, ISO-TP transport details, gateway sub-target
relays.

**Out of scope.**
- **SecOC / TSK key extraction** — see [`optskug/docs`](https://github.com/optskug/docs)
  for the active project on the ADAS-bus side. eTNGA SecOC is uncracked as
  of writing; everything in this repo is read-only diagnostic data that
  doesn't require SecOC.
- **Internal vehicle CAN buses** beyond what's bridged to OBD-II by the
  Central Gateway. The OBD bus only sees a subset.
- **Active probing protocols** that could affect vehicle state. Some `0x31`
  Routine Control IDs are documented for completeness, but invocation is
  the reader's responsibility.

## Related public work

- **[OBDb](https://github.com/OBDb)** maintains structured JSON signal-set
  catalogs across hundreds of vehicles, including
  [`Toyota-bZ4X`](https://github.com/OBDb/Toyota-bZ4X) and
  [`Subaru-Solterra`](https://github.com/OBDb/Subaru-Solterra). This repo
  goes broader (broadcast frames, gateway relays, derivation methodology)
  than OBDb's schema can model; the subset that fits will be upstreamed.
- **[`optskug/docs`](https://github.com/optskug/docs)** — SecOC/TSK key
  extraction for eTNGA ADAS. Complement to this repo's diagnostic-bus
  scope.
- **[Solterra Forum PIDs/OBD thread](https://www.solterraforum.com/threads/pids-obd-commands.1172/)**
  is the de-facto community knowledge base in forum form. Cross-reference
  when in doubt.

## Confidence model

Each decoded signal carries a confidence rating in its `ecus/<ecu>.md`
front-matter:

- **`high`** — formula confirmed by both a live bus exchange and an
  independent reference (Techstream parameter, OBDb signal, public ABRP
  bZ4X/Solterra OBD config, community publication).
- **`medium`** — formula confirmed by live bus exchange OR a strong
  reference, but not both.
- **`low`** — single source, plausible but not cross-verified.

See `docs/conventions.md` for the full per-ECU file structure.

## License

Dual-licensed — see `LICENSE`.

- **Code** under `bin/` — MIT.
- **Documentation and decoded knowledge** elsewhere — CC-BY-4.0.

Decoded PIDs, formulas, and ECU maps may be reused freely with attribution.

## Contributing

PRs welcome. See `CONTRIBUTING.md`.

---

Last updated: 2026-05-10.
