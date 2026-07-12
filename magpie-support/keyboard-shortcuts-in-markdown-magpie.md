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

- **Escape** – Close any open modal or drawer.

> **Note:** The web console deliberately keeps custom keyboard shortcuts to a minimum to avoid conflicts with screen reader navigation and browser extensions, and to reduce the learning curve for occasional users.

## MCP Server

The MCP server does not expose keyboard shortcuts; all interaction is programmatic via tools.

## Contributing

Pitch deck shortcuts are defined inline in the generated HTML's JavaScript; Escape-to-close is implemented via keydown event handlers in the modal components themselves. Contributions are welcome via pull requests.
