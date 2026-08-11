# oan-internal-doc

Internal architecture documentation for the OpenAgriNet platform.

## Contents

- **[OVERVIEW.md](OVERVIEW.md)** — **start here.** The whole system in ten minutes: what it is,
  how a question gets answered, what the tools do, how it differs from real Beckn, and the three
  problems worth fixing first.

- **[ARCHITECTURE.md](ARCHITECTURE.md)** — the full trace. How the system works today across the three deployments
  (BharatVistaar, MahaVistaar, Amul): module map, request flow, catalog publish/discover,
  telemetry, auth, and the gap list for re-architecting onto the current Beckn protocol.

## Viewing the diagrams

Both docs use [Mermaid](https://mermaid.js.org/) diagrams in ` ```mermaid ` fences.
GitHub renders these natively — just open the file here in the browser.

If you are reading it locally and see raw code instead of diagrams, your viewer lacks Mermaid support:

| Viewer | How to get rendering |
|---|---|
| VS Code | install the **Markdown Preview Mermaid Support** extension, then `Cmd+Shift+V` |
| Obsidian / Typora / Joplin | works out of the box |
| Any browser | paste a single diagram into [mermaid.live](https://mermaid.live) |
| Export to PDF/PNG | `npx @mermaid-js/mermaid-cli -i ARCHITECTURE.md -o ARCHITECTURE-rendered.md` |

> Note: the macOS Finder Quick Look preview and most plain Markdown viewers do **not** render Mermaid.

## Status

Traced 2026-08-11. Claims are marked ✅ verified (read in code, with `file:line`),
⚠️ inferred, or ❌ not traced — see §10 for known blind spots.
