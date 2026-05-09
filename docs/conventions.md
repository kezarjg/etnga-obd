# Conventions

Single source of truth for naming, formatting, and schema decisions in this workspace. Read once, follow always.

## Session stems

Format: `YYYY-MM-DD_HHMM_<slug>`

- Date and time use the local timezone of the machine that started the capture.
- Slug is lowercase, alphanumeric, hyphens only (regex: `^[a-z0-9][a-z0-9-]*$`). No spaces, no underscores in the slug part.
- Slug describes what was *done* in the experiment, not what was found. `read-vin`, `actuator-rear-wiper`, `change-headlight-delay`. Findings get backported into `messages/` — they don't show up in the slug.

The same stem is used for the capture file (`captures/<stem>.log`) and the notes file (`sessions/<stem>.md`).

## Bus bitrate

Default and assumed: **500 kbps** (standard OBD-II HS-CAN per ISO 15765-4). All scripts default to this.

If you ever need a different rate (e.g., probing a non-OBD bus that happens to be physically wired in), pass it explicitly to `bin/bus-up <bitrate>` and note the rate in the session frontmatter as a non-default — most analysis tools assume 500 kbps.

## CAN ID hex casing

- Always uppercase hex with `0x` prefix: `0x3B7`, `0x7E0`, `0x7E8`.
- Never lowercase: not `0x3b7`. Avoids accidental duplicate files.

## Bit numbering convention

DBC-compatible **Intel / little-endian** convention. `start_bit` = bit 0 of the signal, where bits are numbered from the LSB of byte 0 across to the MSB of byte 7 (so byte 1 contains bits 8–15, etc.).

Pick this once, document it here, never deviate. DBC bit numbering is famously confusing — inconsistency would force a rewrite at DBC-export time.

If a Toyota / Subaru source ever documents a signal in Motorola big-endian terms, **convert it to Intel here** before recording. Note the conversion in the signal's `notes` field.

## Confidence levels

Used in session frontmatter, message frontmatter, and individual signal definitions.

| Level | Meaning |
|---|---|
| `low` | Single observation, plausible but unverified. Often a guess based on one capture. |
| `medium` | Repeated observation or strong correlation, but not independently confirmed. |
| `high` | Confirmed by independent stimulus (signal value tracks dashboard reading across many captures, DID is named in a public Toyota or OBD-II reference, etc.). |

Be honest. Most things start `low` and get upgraded.

## Capture retention

`bin/capture-pull` rsyncs the log to devbox, writes a `.sha256` sidecar, re-hashes the remote copy, and **deletes from the capture host only after the hashes match**. Devbox is the sole long-term copy.

Don't keep your own redundant copies on the capture host — disk is often small there. If you need a second copy for safety, take it on devbox.

## Session notes frontmatter

```yaml
---
date: 2026-05-03T14:30-07:00
slug: read-vin
capture: ../captures/2026-05-03_1430_read-vin.log
capture_sha256: <filled in by capture-pull>
techstream_actions:
  - "Connected to Engine ECU"
  - "Read DID F190"
related_messages: [0x7E0, 0x7E8]
related_ecus: [engine]
confidence: medium
---
```

Body sections (in order): `What I did`, `What I saw on the bus`, `Hypotheses`, `Knowledge backported`, `Open questions`.

## Broadcast message frontmatter (`messages/broadcast/0xNNN.md`)

```yaml
---
id: 0x3B7
id_type: standard            # standard (11-bit) | extended (29-bit)
kind: broadcast
sender: unknown              # best-guess ECU; "unknown" is honest
dlc: 8
cycle_ms: 100                # observed period; null if aperiodic
confidence: medium
sources:
  - 2026-05-03_1430_read-vin
signals:
  - name: VehicleSpeed
    start_bit: 16
    length: 16
    byte_order: little
    signed: false
    factor: 0.01
    offset: 0
    min: 0
    max: 250
    unit: km/h
    receivers: [unknown]
    confidence: high
    notes: "Matched dashboard speed across 5 captures."
  - name: Unknown_b4
    start_bit: 32
    length: 8
    confidence: low
    notes: "Increments by 1 every 100ms, wraps 0..255 — likely a rolling counter."
---
```

## Diagnostic message frontmatter (`messages/0xNNN.md`)

```yaml
---
id: 0x7E0
id_type: standard
kind: diagnostic-request     # or diagnostic-response
counterpart: 0x7E8           # paired request/response ID
target_ecu: engine
isotp: true
sources:
  - 2026-05-03_1430_read-vin
services_observed:
  - sid: 0x22
    name: ReadDataByIdentifier
    dids_observed:
      - did: 0xF190
        name: VIN
        encoding: ascii
        length: 17
        confidence: high
        sources: [2026-05-03_1430_read-vin]
---
```

## Three rules

1. **Every signal has `confidence` and at least one `sources` entry.** No orphan claims.
2. **Unknown bytes get explicit `Unknown_bN` signal entries.** Documenting what we don't know yet is as valuable as what we do.
3. **Bit numbering = Intel/little-endian.** No exceptions.
