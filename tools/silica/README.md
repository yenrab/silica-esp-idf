# tools/silica

Home of the ESP-IDF side of Silica language support.

* `bin/silica-compiler` is a **placeholder**, not the Silica compiler. It mimics
  the real compiler's file-level contract (reads `silica.config` from the
  working directory, writes one `.sams` and one `.iface` per unit, honours the
  exit-75 resume protocol) so the build-system plumbing can be developed and
  tested now. It generates no code.
* The real compiler is developed in a separate repository and is deliberately
  **not** linked or vendored here. When it is ready it replaces the placeholder
  through the normal `tools/tools.json` tool-download mechanism.

See `SILICA_INTEGRATION_PLAN.md` at the repository root for the conversion
plan, the phase this directory is in, and the interface contract the real
compiler has to meet before it can be dropped in.
