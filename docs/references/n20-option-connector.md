# N20 "Option Connector" (5-pin Roof Power Tap) — Reference

> Companion to `f39-community-research.md`. N20 is the **5-pin power/ignition/ground** option connector in the **headliner / roof harness near the rear-view mirror**. It's the "dashcam mounting point" connector — distinct from F39 which is the 2-pin CAN tap behind the passenger kick panel.

## Connector identification

| Field | Value |
|---|---|
| Toyota name | N20 "Option Connector" (also "Roof Option Connector", "Dash Cam Connector") |
| Type code | 5-Pin Type A (community naming) |
| **Housing in vehicle** | Toyota **`90980-12365`** ("Housing, Connector M" — male side, in the car) |
| **Mating housing for installer** | Toyota **`90980-12366`** ("Housing, Connector F" — female side, the plug you'd buy) |
| Color | White |
| Location on Solterra/bZ4X | Headliner / roof harness near the rear-view mirror |
| Cross-platform | Same connector on Tundra (2022+), Sequoia (2023+), Tacoma 4G (2024+), 4Runner 6G (2025+), Land Cruiser 250, Sienna, plus Lexus equivalents |

## Pinout (community-confirmed, matches the Solterra EM)

Of the 5 pin cavities, **only 3 are populated** from the factory:

| Pin | Function | Wire color | Source |
|---:|---|---|---|
| **1** | **B+** (always-on +12V battery) | Pink | 10A `ECU-B No. 2` fuse |
| **2** | **IG1** (switched +12V, on with ignition) | Black | 10A `ECU-IGR No. 2` fuse |
| 3 | unpopulated cavity | — | — |
| **4** | **GND** | White w/ black stripe | chassis ground |
| 5 | unpopulated cavity | — | — |

## Where to buy

### Just the bare housing (Toyota OEM)
- Toyota dealer P/N **`90980-12366`** — bare 5-pin female housing, no terminals included
  - https://www.toyotapartsdeal.com/oem/toyota~housing~connector~f~90980-12366.html
  - https://autoparts.toyota.com/products/product/housing-connector-female-9098012366
- Amazon listing for the M/F housing pair (no terminals): https://www.amazon.com/Automotive-Replacement-Connector-90980-12365-90980-12366/dp/B0DN5YZRS6

### Pre-wired plug-and-play pigtails (recommended for installs)

The community has converged on these because they avoid crimping:

| Product | Source | Notes |
|---|---|---|
| **Dongar 5-Pin Type A Dash Cam Power Adapter** | https://dongar.tech/products/toyota5pin <br> https://www.amazon.com/Dongar-Adapter-Select-Connector-Compatible/dp/B0CWB3MGYW | Community standard, ~$40, plug-and-play |
| **Hifihia 5-Pin Type A** | https://www.amazon.com/Hifihia-Connector-Compatible-2024-2026-2022-2026/dp/B0GM545WZ2 | Cheaper alternative |
| Toyota OEM dashcam kit | PT949-08230 (https://toyotaparts.sparkstoyota.com/install/PT949-08230inst-bz4x.pdf) | The factory bZ4X dashcam install kit; uses N20 |

⚠️ **Don't buy the "10-Pin Type B"** variant (Mangoaltech / Dongar 10-Pin) — that's a different Toyota connector and won't fit N20.

## Important: N20 is NOT co-located with F39

This matters for an OVMS install:

| Connector | Location | Purpose |
|---|---|---|
| **F39** | Passenger-side kick panel area, near gateway / ECU Integration Box | CAN data tap (2-pin, A-CAN bus) |
| **N20** | Headliner near rear-view mirror | Power tap (5-pin, B+/IG1/GND) |

These are **physically far apart**. For an OVMS install you have three options:

1. **OVMS module mounted at the headliner**: power from N20 (Dongar pigtail), run a cable down the A-pillar to F39 for CAN. Cable is just 2 wires (CAN-H/CAN-L) — easy enough to route along the existing A-pillar harness.
2. **OVMS module mounted behind passenger kick panel**: power from a separate source near the kick panel (e.g., fuse-tap into Power Distribution Box F143/F144, or a different option connector if one exists in that area), CAN direct from F39. Cleanest mechanically but requires a separate power solution.
3. **OVMS module mounted in the headliner area, no F39 use**: power from N20, CAN from somewhere else (e.g., a tap on the headliner harness itself). Probably not viable since the headliner harness doesn't carry the diagnostic / vehicle-state CAN buses.

## Recommendation for OVMS install on Solterra

Most likely **Option 1**: OVMS at the headliner near N20 (out of sight, easy power), with a 2-wire CAN extension running down the A-pillar to F39. Mounting up high also helps the OVMS module's GPS reception (better antenna view).

Alternative **Option 2** is mechanically cleaner but needs a power source near the kick panel — typically a fuse-tap into the Power Distribution Box (F143/F144) using an add-a-fuse adapter on a circuit you don't mind sharing.

## Cross-references

- Solterra Forum dashcam thread: https://www.solterraforum.com/threads/dashcam-powered-mirror-adapter-outback-temporary-example.538/
- Solterra Forum OEM dashcam thread: https://www.solterraforum.com/threads/oem-dashcam.1407/
- Tundra dashcam connector reference: https://www.tundras.com/threads/usb-dashcam-connector-5-pin-harness-in-dome-light-roof-cavity.130672/
- Tacoma 4G dashcam install: https://www.tacoma4g.com/forum/threads/how-to-access-the-5-pin-oem-dash-cam-connector-in-all-2024-tacoma-4th-gen-for-dashcam-install.4546/
- 4Runner 6G dashcam install: https://www.4runner6g.com/forum/threads/how-to-access-the-oem-dash-cam-connector-for-a-dashcam-install.1422/
- Pinout cross-reference (Tacoma World): https://www.tacomaworld.com/threads/pin-out-wiring-diagram-oem-dashcam.821892/
- Toyota OEM bZ4X dashcam install PDF: https://toyotaparts.sparkstoyota.com/install/PT949-08230inst-bz4x.pdf
