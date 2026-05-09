# messages/

Per-CAN-ID knowledge files — the *model* layer. One file per CAN ID, named with uppercase hex and `0x` prefix (e.g., `0x7E0.md`).

Two flavours, both with YAML frontmatter:

- **Diagnostic IDs** (top level): request/response IDs used by UDS over ISO-TP. Examples: `0x7DF`, `0x7E0`, `0x7E8`, `0x7E1`, `0x7E9`, etc.
- **Broadcast IDs** (`broadcast/`): periodic single-frame messages with decoded signals. The schema here is what eventually becomes a DBC file.

Frontmatter schemas: see `../docs/conventions.md`.

**Rule:** every claim in these files must cite at least one session stem in its `sources` field. No orphan claims.
