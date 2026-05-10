# Toyota Techstream — Research Notes for Solterra/bZ4X Reverse Engineering

> Research conducted May 2026. This document focuses on the diagnostic tool itself and what's been done by the broader community.

## 1. Current Techstream Version

**Latest publicly known stable release: Techstream v18.00.008**, dated March 15, 2023 in [Toyota TIS release notes](https://techinfo.toyota.com/techInfoPortal/staticcontent/en/techinfo/html/prelogin/tsrss/ts_new_features.html).

No 18.10 / 18.20 / 19.x has been released to the public Techstream channel since. Bootleg redistributors (UpdateStar, Autosofts) re-bundle the same MSI under "2025/2026" labels but the binary hasn't changed.

**v18.00.008 is the first Techstream that explicitly lists Toyota bZ4X 2024 / Subaru Solterra in its model coverage.** Any earlier 18.x point release will have incomplete coverage and should be upgraded before further sessions.

## 2. Successor Product Status

**Techstream is functionally end-of-life.** Toyota has migrated new-vehicle support to **GTS+ (Global Techstream Plus)**, a cloud-based dealer-only platform. Toyota TSBs since 2020 (e.g. T-SB-0107-20 Rev2, [NHTSA mirror](https://static.nhtsa.gov/odi/tsbs/2023/MC-10241404-9999.pdf)) reference **"GTS+ ECU Flash Reprogramming With Security"** for SecOC-protected operations rather than Techstream.

In **North America**, GTS+ is dealer-facing for new vehicles requiring secure programming (anything with TSK/SecOC). Techstream remains supported for legacy diagnostics and is still accepted via TIS subscription. There is **no public-tier GTS+** — it's gated behind dealer business registration. For independent users, **Techstream 18.00.008 is the practical ceiling**; SecOC-gated tasks on bZ4X fundamentally need GTS+ or a dealer.

## 3. bZ4X / Solterra Support in Techstream

- **Minimum version: v18.00.008** for 2024 model year. Earlier 18.x lacks the model-info package.
- A first-connection registration step ("model info download") is required once per Techstream install.
- **Solterra-specific quirk:** community reports on [Diagnostic Network](https://diag.net/msg/m13u4wh5arsuajyxbckdrnxd0h) state that the Toyota build of Techstream **cannot fully program a Subaru Solterra**. The VIN/model lookup converts but programming sessions fail because Subaru-specific module IDs and calibration files are absent from Toyota TIS. Read-only DTC + live-data over Toyota Techstream generally works because the underlying ECUs are Toyota; programming, key registration, and some bidirectional functions don't.
- **Subaru SSM4** is needed for Subaru-badged modules on the Solterra (X-Mode-related calibration, body-control variants).

### Features known to be broken / restricted on bZ4X/Solterra via Techstream

- **EMPS (electric power steering)** comm is broken up to v18.0 per [Techstream Known Bugs page](https://techinfo.toyota.com/techInfoPortal/staticcontent/en/techinfo/html/prelogin/tsrss/ts_known_bugs.html).
- **Reprogramming and ECU security-key writing** require a Security Signature handshake (SecOC/TSK). TSBs [T-SB-0111-20](http://media.fixed-ops.com/Toy_ServiceBulletins/sb0111t20.pdf) and the bZ4X-specific [MC-10228596](https://static.nhtsa.gov/odi/tsbs/2022/MC-10228596-9999.pdf) describe the flow. Toyota stated the bZ4X EPS stores the key in an HSM; existing TSK extraction exploits **do not work** on this generation.
- **Subaru-only modules** (e.g. body-control variants on Solterra) need SSM4.

## 4. Variants and Alternatives

- **Techstream LITE vs full Techstream** ([Toyota TS-Lite FAQ PDF](https://techinfo.toyota.com/techInfoPortal/staticcontent/en/techinfo/html/prelogin/docs/tslfaqtinfo.pdf)): same software binary; LITE uses a J2534-compatible third-party VIM (Mongoose, Bosch CCI, etc.) and a cheaper aftermarket subscription (~$300–500/yr) without the business-registration requirement of full TIS (~$1,200/yr). Both can do ~95%+ of TIS functions on DLC3/J1962 vehicles.
- **Subaru SSM4** ([PDI Tech Subaru aftermarket](https://security.pditechnologies.com/subaru-tech/)): dealer-level Subaru tool, MY2004–2024 incl. Solterra; the right tool for Subaru-side modules.
- **Open-source alternatives:** [SavvyCAN](https://www.savvycan.com/) is the standard reverse-engineering UI. CAN-FD support in common open-source tooling is partial; a comma panda firmware tweak forces the bZ4X back to classical CAN for diagnostic comms.

## 5. Community Reverse-Engineering Status (THIS IS THE GOOD PART)

**bZ4X / Solterra are still uncracked at the SecOC layer**, but a substantial OBD-II PID community is active.

### Trackers / negative-status sources

- **[optskug/docs](https://github.com/optskug/docs)** — primary Toyota/Subaru SecOC tracker. Lists 2023+ bZ4X and Solterra as "🔴 Not hacked" with a suspected updated TSK/SecOC chip. Key extraction has only succeeded on 2020–2021 TSK vehicles. Active discussion in comma Discord `#toyota-security`.
- **[openpilot supported cars](https://github.com/commaai/openpilot/blob/master/docs/CARS.md)** — bZ4X 2023+ and Solterra 2023+ are explicitly **unsupported** because of SecOC-signed control messages. See also [bZForums FSD thread](https://www.bzforums.com/threads/full-self-driving-for-bz4x.1431/).

### Active community PID work (HIGH RELEVANCE TO OUR WORK)

- **[Subaru Solterra Forum — PIDs/OBD commands](https://www.solterraforum.com/threads/pids-obd-commands.1172/)** — substantial active thread documenting working PIDs for HV battery SOC, pack voltage/current, individual cell temps, motor torque, coolant temps. **The 2021 RAV4 Prime XGauge config was found to mostly work on Solterra.** This is the single highest-leverage starting point for new discovery work.
- **[Subaru Solterra Forum — SoC for the traction battery](https://www.solterraforum.com/threads/soc-state-of-charge-for-the-traction-battery.925/)** — community-decoded SOC PID(s) and scaling.
- **[Subaru Solterra Forum — ScanGauge2 SoC display](https://www.solterraforum.com/threads/displaying-soc-hv-batt-voltage-hv-batt-current-with-scangauge2.979/)** — ScanGauge2-format strings for SoC, pack voltage, pack current. Each ScanGauge2 string is essentially a hex-encoded UDS read with a scaling formula → directly translates to a poll-list entry in any UDS-capable client.
- **[ABRP added native bZ4X/Solterra OBD live-data telemetry](https://www.bzforums.com/threads/abrp-added-obd-live-data-for-bz4x-today.476/)** — A Better Route Planner now supports it, which means *someone* did the PID work and shipped it. Their app is closed-source but their PID list could potentially be extracted by sniffing what the ABRP OBD adapter polls.

### DBC file status

- **No consolidated public bZ4X/Solterra DBC** exists in [commaai/opendbc](https://github.com/commaai/opendbc) or anywhere else publicly.
- PIDs are circulating in forum posts, **not DBC form**.
- [Project Gus's Kona writeup](https://www.projectgus.com/2023/10/kona-can-decoding/) is referenced as a methodology template by Solterra hackers but is for a different platform.

## 6. Implications for reverse-engineering work

1. **There is genuinely novel work to do.** No public DBC, no consolidated decoded protocol document. Per-message YAML in `messages/`, per-ECU profiles in `ecus/`, and encoded scaling formulas could be the first public structured documentation of the Solterra protocol — **publishable upstream value** at opendbc, OBDb, or as a standalone repo.

2. **Cross-reference the Solterra Forum PID threads early.** Before doing blind discovery on basic signals like SoC, pack voltage, motor RPM, etc., fetch the community-known PIDs and validate them on the test vehicle. Confirmed-working PIDs save many Techstream sessions; divergences flag things worth investigating.

3. **Use the RAV4 Prime XGauge config as a cross-check.** The Solterra forum thread suggests the 2021 RAV4 Prime XGauge mostly works on Solterra — confirms the inheritance from RAV4 Prime's Toyota Hybrid System architecture and provides a known-good signal map to compare reverse-engineered findings against.

4. **Don't bother with SecOC for now.** Active control commands (lock/unlock, climate control) are SecOC-gated and aren't crackable on this generation. Stick to read-only diagnostic reads via UDS over OBD-II — that's where all the value is and it's not security-restricted.

5. **Verify Techstream version is 18.00.008.** Earlier 18.x has incomplete bZ4X coverage. Quick check in Techstream → Help → About.

## 7. Summary

- Stay on **Techstream 18.00.008**; verify the install. No newer "official" version exists.
- **bZ4X/Solterra reprogramming, key writing, and SecOC-gated bidirectional features are off-limits** without GTS+ or a dealer.
- For Subaru-specific modules add **SSM4**.
- For deeper CAN-bus reverse engineering, the live action is on the **[Solterra Forum PID threads](https://www.solterraforum.com/threads/pids-obd-commands.1172/)** and the comma Discord, not in any Techstream-derived export.
- **Highest-leverage next step:** pull the published RAV4 Prime XGauge / Solterra-Forum PID list and validate each PID against the test vehicle (passively first, since a Techstream session may still be active) — much faster than blind Data-List-isolation for already-known signals.

## Sources

- [Toyota TIS Techstream Release Notes](https://techinfo.toyota.com/techInfoPortal/staticcontent/en/techinfo/html/prelogin/tsrss/ts_new_features.html)
- [Toyota TIS Techstream Known Bugs](https://techinfo.toyota.com/techInfoPortal/staticcontent/en/techinfo/html/prelogin/tsrss/ts_known_bugs.html)
- [GTS+ v2023.04.003.02 Known Bugs (Toyota TIS PDF)](https://techinfo.toyota.com/techInfoPortal/staticcontent/en/techinfo/html/prelogin/docs/GTS+_Known_Bugs_V2023.04.003.02_Techinfo.pdf)
- [Toyota TS-Lite Aftermarket FAQ (PDF)](https://techinfo.toyota.com/techInfoPortal/staticcontent/en/techinfo/html/prelogin/docs/tslfaqtinfo.pdf)
- [T-SB-0107-20 Rev2 GTS+ ECU Flash Reprogramming With Security](https://static.nhtsa.gov/odi/tsbs/2023/MC-10241404-9999.pdf)
- [T-SB-0111-20 Rev2 ECU Security Key Writing](http://media.fixed-ops.com/Toy_ServiceBulletins/sb0111t20.pdf)
- [Toyota bZ4X TSB MC-10228596](https://static.nhtsa.gov/odi/tsbs/2022/MC-10228596-9999.pdf)
- [Diagnostic Network — Subaru Solterra Key Programming](https://diag.net/msg/m13u4wh5arsuajyxbckdrnxd0h)
- [optskug/docs (Toyota/Subaru TSK/SecOC tracker)](https://github.com/optskug/docs)
- [openpilot supported cars](https://github.com/commaai/openpilot/blob/master/docs/CARS.md)
- [bZForums — Techstream options](https://www.bzforums.com/threads/techstream-options.974/)
- [bZForums — ABRP OBD live data added for bZ4X](https://www.bzforums.com/threads/abrp-added-obd-live-data-for-bz4x-today.476/)
- [Subaru Solterra Forum — PIDs/OBD commands](https://www.solterraforum.com/threads/pids-obd-commands.1172/)
- [Subaru Solterra Forum — SOC for traction battery](https://www.solterraforum.com/threads/soc-state-of-charge-for-the-traction-battery.925/)
- [Subaru Solterra Forum — ScanGauge2 SoC display](https://www.solterraforum.com/threads/displaying-soc-hv-batt-voltage-hv-batt-current-with-scangauge2.979/)
- [SavvyCAN](https://www.savvycan.com/)
- [commaai/opendbc](https://github.com/commaai/opendbc)
- [PDI Subaru SSM aftermarket](https://security.pditechnologies.com/subaru-tech/)
- [OBDII365 — TIS Techstream vs Techstream Lite](http://blog.obdii365.com/2019/05/27/toyota-tis-techstream-vs-techstream-lite/)
- [Project Gus — Kona CAN decoding](https://www.projectgus.com/2023/10/kona-can-decoding/)
