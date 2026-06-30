---
title: UI/UX Design Decisions and User Interface Behavior
status: draft
---

# UI/UX Design Decisions and User Interface Behavior

This document describes the user interface and interaction design of the Markdown Magpie web console (`apps/web`). It covers layout, navigation, keyboard shortcuts, and the rationale behind key design choices.

## Overview

The web console is a Next.js application (port 3000) that provides a review and administration interface for managing knowledge bases. It is primarily designed for maintainers to:

- Ask questions against the indexed knowledge base.
- Browse indexed documents and sections.
- Review and manage gap clusters and proposals.
- Configure and trigger scheduled maintenance (Crunch).

The UI is deliberately sparse and utilitarian, focusing on clarity and fast task completion over rich visual presentation.

## Navigation and Layout

- **Top-level pages** are accessed via a sidebar or header: **Ask**, **Knowledge**, **Proposals**, **Crunch**.
- Each page corresponds to a key workflow:
  - **Ask**: Submit a question and view the answer after the async watcher processes it.
  - **Knowledge**: List and search indexed documents and sections.
  - **Proposals**: Review draft Markdown proposals, change their status, and publish them.
  - **Crunch**: View scheduled tidy runs, trigger new runs, and publish crunch plans.
- The layout uses responsive CSS; the sidebar collapses on narrow screens.

## Keyboard Shortcuts

The web console intentionally exposes **no custom keyboard shortcuts** beyond native browser defaults. This is a deliberate design choice to:

- Avoid conflicts with screen reader navigation and browser extensions.
- Keep the learning curve minimal for occasional users.
- Reduce accidental submissions during long-running async operations.

Notable absence: the Ask page does not bind the Enter key to submit a question. Users must click the **Ask** button. This prevents duplicate job submissions when the Enter key is pressed while the system is still processing a previous request.

> **Note:** The Markdown Magpie pitch deck (`presentation/`) implements its own keyboard navigation (arrow keys, Home, End, O, F) for slide advancement. Those shortcuts do **not** apply to the web console.

## Ask Page Interaction Flow

1. The user types a question into a `<textarea>` or `<input>` element.
2. Clicking **Ask** sends a `POST /api/ask` request.
3. The API returns HTTP 202 with a job ID.
4. The UI shows a loading indicator and polls the job status or displays a link to wait.
5. When the watcher completes the job, the answer (with citations) appears.

This async flow is central to the product: all AI work runs in a background watcher, so the UI never blocks. The lack of immediate response on Enter is a tradeoff for reliability and queue management.

## Design Rationale

- **Simplicity over complexity**: The console is a tool, not a marketing site. Excessive styling or animations would distract from the review tasks.
- **Async-first**: Since `POST /api/ask` is enqueue-only, the UI must accommodate a wait. The current design avoids hiding this — it shows the job state rather than simulating instant answers.
- **Radix UI primitives**: The project uses Radix UI for accessible components (tooltips, etc.) and React Flow for visualising knowledge flows (Crunch and Proposal pipelines). These libraries provide a solid baseline for accessibility and interaction.
- **No inline editing**: To keep the UI predictable, all editing of knowledge content is performed via proposals and published pull requests, not directly in the web console.

## Future Considerations

- A keyboard shortcut (e.g., Ctrl+Enter) to submit a question from the Ask page could be added in a follow-up, provided the UI can debounce or warn about duplicate submissions.
- The lack of documentation for UI/UX decisions is itself a gap — this document begins to fill it. Additional sections on error states, empty states, and mobile behaviour may be added as the product evolves.
