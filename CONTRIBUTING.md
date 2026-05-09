# Contributing

PRs welcome. Three things to know before submitting:

1. **Redact your VIN.** Captures, log excerpts, and screenshots in PRs must not
   contain a 17-character VIN. We run a pre-commit hook against the maintainer's
   common patterns; please do the same on your own clones.

2. **One claim, one citation.** When adding a decoded signal, include either:
   the raw bus exchange (5–10 lines, redacted) that backs the formula, OR a
   reference to a specific Techstream parameter / community thread / DBC file
   that confirms it. Confidence levels (`high` / `medium` / `low`) are encoded
   in the per-ECU markdown front-matter — see `docs/conventions.md`.

3. **Platform-shared findings deserve a callout.** This repo covers Toyota bZ4X,
   Subaru Solterra, and Lexus RZ — all on Toyota's eTNGA platform. If a finding
   is verified on only one variant, say so; if it's likely shared, say that too.

For broader RE work that doesn't fit our scope (in-vehicle protocols beyond OBD-II,
SecOC/TSK key extraction, ADAS bus), see `optskug/docs` on GitHub.
