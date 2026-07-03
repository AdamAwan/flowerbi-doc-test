---
title: Keyboard Shortcuts in Markdown Magpie
status: draft
---

# Keyboard Shortcuts in Markdown Magpie

Markdown Magpie provides keyboard shortcuts for navigating the pitch deck (`presentation/index.html`) and the web console. This page documents the currently known shortcuts.

## Pitch Deck (`presentation/index.html`)

The self-contained HTML pitch deck supports the following keyboard shortcuts for navigation:

| Key | Action |
| --- | --- |
| `→` / `Space` / `PageDown` | Next slide |
| `←` / `PageUp` | Previous slide |
| `Home` / `End` | First / last slide |
| `O` | Overview grid (click a slide to jump) |
| `F` | Fullscreen |
| Click right / left third | Next / previous |

The URL hash tracks the current slide (e.g. `index.html#8`) for deep linking.

## Web Console

The web console (`@magpie/web`) is built with Next.js and React. The following shortcuts are available:

- **Ctrl+Enter** (or **Cmd+Enter** on macOS) in the Ask page textarea – Submit the question.
- **Escape** – Close any open modal or drawer.
- **?** – Open the keyboard shortcuts help dialog (if implemented).

> **Note:** The web console deliberately keeps custom keyboard shortcuts to a minimum to avoid conflicts with screen reader navigation and browser extensions, and to reduce the learning curve for occasional users.

### Rationale for Ctrl+Enter over Enter

The Ask page uses a `<textarea>` rather than a single-line `<input>`. Pressing Enter inserts a newline, preventing accidental submission during long-running async operations. Users must use Ctrl+Enter or click the **Ask** button to submit. This design avoids duplicate job submissions when the Enter key is pressed while the system is still processing a previous request.

## MCP Server

The MCP server does not expose keyboard shortcuts; all interaction is programmatic via tools.

## Future Shortcuts

Improvements planned include:

- **Submit on Enter** (single-line mode) – Currently the Ask textarea requires Ctrl+Enter to submit. A configuration option to enable Enter-as-submit in single-line mode is under consideration.
- **Global search** (`Ctrl+K`) – For quick access to knowledge base search.

## Contributing

Shortcuts are implemented in the web app components. To add or modify shortcuts, see the web app source at `apps/web/src/`. Contributions are welcome via pull requests.
