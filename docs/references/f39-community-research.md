# F39 Accessory Connector — Community Research

> Researched May 2026. Goal: find existing documentation on the Toyota bZ4X / Subaru Solterra **F39 Accessory Connector** (the 2-pin A-CAN tap at gateway pins CA8H/CA8L) — physical location, sleep/wake behavior, mating connector parts, existing aftermarket installs.

## Bottom line

**Nobody has publicly documented F39 on the bZ4X / Solterra.** Searches across bZForums, Solterra Forum, Reddit, optskug, openinverter, comma.ai, opendbc, and YouTube returned zero direct hits. Anyone publishing concrete F39 capture data would be the first.

But — the **same connector body and architectural pattern exists on Toyota GR Corolla and GR Yaris** (where it's labeled **H62**, same Toyota housing P/N **90980-12936**, same "Bus Buffer ECU" role), and the GR community is actively using it as a CAN logger tap. That gives strong extrapolation ground for parts and behavior.

## What's confirmed by community work on adjacent platforms

### Mating connector parts (TRANSFERABLE from GR Corolla)

| Part | Description | Source |
|---|---|---|
| **`90980-12936`** | Housing (body of F39 — the female side mounted in the car) | confirmed in Solterra EM; same on GR Corolla H62 |
| **`90980-12937`** | Pre-crimped wire variant (the male plug + wires you'd use to mate to F39) | GR Corolla CAN-bus thread |
| **Ballenger Motorsports CONN-100954** | Stocking source for the kit | GR Corolla forum |
| Toyota dealer | Generic "Toyota Housing, Connector M" | toyotapartsdeal.com |

**Wire colors on the GR Corolla H62 equivalent**: Brown = CAN-H, White = CAN-L. Verify with a multimeter on the Solterra before trusting.

Sources:
- https://www.grcorollaforum.com/threads/can-bus-reverse-engineering.7486/
- https://www.gr-zoo.com/threads/can-bus-reverse-engineering.7834/
- https://www.toyotapartsdeal.com/oem/toyota~housing~connector~m~90980-12936.html

### Physical location pattern (TRANSFERABLE from Toyota platform-wide)

GR Corolla owners describe H62 as *"an empty plug Toyota generously left us by the RH footwell"*. That matches:

- The bZ4X EM placing the Network Gateway ECU in the **ECU Integration Box RH** (passenger-side under-dash)
- The bZ4X Repair Manual's gateway-removal procedure calling for removal of the right kick panel + lower instrument panel to access the gateway

**Practical: F39 should be a small 2-pin pigtail taped to the harness in the passenger-side footwell area, near the ECU Integration Box.** Consistent with what we'd already inferred from the EM, now confirmed by the analog on adjacent platforms.

### A-CAN sleep behavior (PARTIAL transfer)

Toyota service docs on adjacent platforms (Yaris XP210, Sienna, etc.) document a formal **"Communication Stop Mode"** for the Bus Buffer ECU side of the gateway:
- Triggered when ignition is off + doors quiet + no switch activity for **~1 minute**
- A-CAN goes silent in this state

So **A-CAN almost certainly sleeps with similar latency to OBD-II** — the choice between F39 and DLC may not change sleep behavior much. The exact behavior on bZ4X / Solterra is **unverified empirically**.

Source: https://www.toyaris4.com/toyota_yaris_bus_buffer_ecu_communication_stop_mode-1458.html

### A-CAN data content (architectural extrapolation only)

Toyota's design intent for A-CAN across all platforms:
- Curated subset of vehicle data for accessory ECUs (RES gateway, alarms, telematics)
- Typical signals: vehicle speed, gear, door states, ignition/READY state, lock state, charging connector state
- **NOT** raw battery pack data (that's on V-CAN behind the gateway and only proxied to OBD-II diagnostically)

So expect telematics-grade signals, not full BMS visibility. **No public capture of bZ4X A-CAN traffic exists** to confirm exactly what's there.

### Existing bZ4X / Solterra aftermarket installs (none use F39)

Every documented aftermarket telemetry install on bZ4X / Solterra uses **OBD-II**:
- ABRP via OBDLink CX (OBD-II)
- Car Scanner / Carista (OBD-II)
- OEM dashcam PT949-08230 (fuse-tap power, no CAN)
- FITCAMX, Safe Drive Solutions dashcams (fuse-tap)
- OWLCAM (OBD-II)
- ScanGauge2 (OBD-II)

**Nobody has documented an F39 / A-CAN install** on this platform. The Solterra dashcam thread cites a **different connector** (the 5-pin **N20 "Option Connector"** in the overhead console near the rear-view mirror) — that's the headliner accessory tap, not the dash F39 CAN tap.

Source: https://www.solterraforum.com/threads/dashcam-powered-mirror-adapter-outback-temporary-example.538/

### Why this gap exists (the SecOC / openpilot factor)

- **bZ4X/Solterra is "Not compatible" with opendbc / openpilot** because of CAN-FD + SecOC. SecOC keys are locked in the EPS HSM and can't be extracted on this generation (per [optskug/docs](https://github.com/optskug/docs)).
- That's slowed the typical comma.ai-style reverse-engineering pipeline.
- A-CAN is almost certainly **classical CAN, not CAN-FD** (since it's the accessory bus historically), so should be more accessible than the SecOC-protected V-CAN — but nobody has done the work yet.

## What we now know vs. what's still open

| Item | Source | Status |
|---|---|---|
| F39 = 90980-12936 connector body | Solterra EM | ✅ confirmed |
| F39 wired to gateway pins CA8H/CA8L | Solterra EM | ✅ confirmed |
| F39 located behind passenger-side kick panel | EM gateway-removal proc + GR Corolla H62 analog | ✅ very likely |
| Mating connector source: 90980-12937 + Ballenger CONN-100954 | GR Corolla community | ✅ confirmed (transferable) |
| Wire color: Brown = CAN-H, White = CAN-L | GR Corolla H62 | ⚠ likely transferable; verify |
| F39 sleep behavior matches OBD-II (~1 min after idle) | Toyota platform pattern (Yaris, Sienna) | ⚠ likely; unconfirmed on Solterra |
| A-CAN data content (what frames are broadcast) | none | ❓ completely unknown for Solterra |
| Whether F39 is populated/capped from factory | none | ❓ unknown |
| 2WD vs AWD vs region differences in F39 wiring | none | ❓ unknown |

## Practical implication

Both directions of this finding are valuable:

**The negative (no community work)** means tapping F39 and capturing its traffic would be **genuinely novel reverse-engineering**. Both bZForums and Solterra Forum have active interest in CAN tap topics; a documented F39 install with traffic captures would be a notable contribution.

**The positive (parts well-known on GR Corolla)** means **the mating connector and pigtail are sourceable today** without waiting — the Ballenger / Toyota-dealer parts work because the Toyota housing P/N is identical platform-to-platform.

## Recommendation

A **single combined experiment** that resolves all the open questions in the table above:

1. Pop the passenger kick panel
2. Locate F39 (small 2-pin connector with cap, near the ECU Integration Box)
3. Photograph the area before disturbing anything
4. Verify Brown/White color convention with multimeter (continuity to gateway pins CA8H/CA8L)
5. Connect a CAN logger to F39 (via 90980-12937 mating pigtail)
6. Capture for a full sleep/wake cycle: drive, park, ignition off, lock, walk away 10 min, return, unlock, drive again
7. Simultaneously capture from OBD-II for comparison
8. Compare frame inventories and silence intervals
9. Publish on bZForums / Solterra Forum / GitHub — first public documentation of this connector on the platform

That single session resolves all the open questions in the table above.

## Sources

- GR Corolla CAN reverse-engineering: https://www.grcorollaforum.com/threads/can-bus-reverse-engineering.7486/
- GR Yaris CAN reverse-engineering: https://www.gr-zoo.com/threads/can-bus-reverse-engineering.7834/
- Toyota Yaris Bus Buffer ECU stop mode: https://www.toyaris4.com/toyota_yaris_bus_buffer_ecu_communication_stop_mode-1458.html
- Toyota Sienna gateway ECU CAN docs: https://www.tsienna.net/network_gateway_ecu-2857.html
- Toyota housing 90980-12936: https://www.toyotapartsdeal.com/oem/toyota~housing~connector~m~90980-12936.html
- bZ4X dashcam install (kick panel routing): https://toyotaparts.sparkstoyota.com/install/PT949-08230inst-bz4x.pdf
- bZ4X 12V drain (OBD dongle related): https://www.bzforums.com/threads/dead-12v-battery.717/
- Solterra 12V drain: https://www.solterraforum.com/threads/solterra-connect-and-12-volt-battery-drain.2955/
- Solterra dashcam mirror adapter (N20 Option Connector, NOT F39): https://www.solterraforum.com/threads/dashcam-powered-mirror-adapter-outback-temporary-example.538/
- Solterra PIDs / OBD: https://www.solterraforum.com/threads/pids-obd-commands.1172/
- Solterra OBD-2 dongle thread: https://www.solterraforum.com/threads/obd-2-dongle.2712/
- bZ4X ABRP via OBD: https://www.bzforums.com/threads/abrp-added-obd-live-data-for-bz4x-today.476/
- bZ4X OBDLink CX AMA: https://www.bzforums.com/threads/i-bought-an-obdlink-cx-for-my-awd-bz4x-ama.1296/
- opendbc CARS.md (bZ4X "Not compatible"): https://github.com/commaai/opendbc/blob/master/docs/CARS.md
- optskug SecOC tracker: https://github.com/optskug/docs
- SecOC RAV4 Prime extraction (background): https://icanhack.nl/blog/secoc-key-extraction/
- toyota-can-bus-multitool: https://github.com/cydia2020/toyota-can-bus-multitool
