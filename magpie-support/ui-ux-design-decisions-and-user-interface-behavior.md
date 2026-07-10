---
title: UI/UX Design Decisions and User Interface Behavior
status: draft
---

# UI/UX Design Decisions and User Interface Behavior

This document describes the user interface and interaction design of the Markdown Magpie web console (`apps/web`). It covers layout, navigation, keyboard shortcuts (referenced externally), and the rationale behind key design choices.

## Overview

The web console is a Next.js application (port 3000) that provides a review and administration interface for managing knowledge bases. It is primarily designed for maintainers to:

- Ask questions against the indexed knowledge base.
- Browse indexed documents and sections.
- Review and manage gap clusters and proposals.
- Monitor background job status and history.
- Configure and manage scheduled background tasks (e.g., patrol cadences, source-change sync).

The UI is deliberately sparse and utilitarian, focusing on clarity and fast task completion over rich visual presentation.

## Navigation and Layout

- **Top-level pages** are accessed via a sidebar or header: **Ask**, **Knowledge**, **Gaps**, **Seed**, **Proposals**, **Source Map**, **Jobs**, **Activity**, **Insights**, **Schedules**, **Dataflow**.
- Each page corresponds to a key workflow:
  - **Ask**: Submit a question and view the answer after the async watcher processes it.
  - **Knowledge**: List and search indexed documents and sections.
  - **Gaps**: View and manage gap clusters detected from low-confidence answers.
  - **Seed**: Propose a seed plan for a flow by exploring its sources, then review and approve it.
  - **Proposals**: Review draft Markdown proposals, change their status, and publish them.
  - **Source Map**: Display navigation hints contributed by agents as they explore source repositories for source-grounded drafting, verification, and improvement jobs.
  - **Jobs**: Monitor background job status and history.
  - **Activity**: View audit log of system events (maintenance runs, proposal actions, etc.).
  - **Insights**: Visualise pipeline performance trends (backlog, throughput, latency, verification success, freshness, patrol impact) with interactive charts.
  - **Schedules**: Configure and manage background job schedules (e.g., patrol cadences, source-change sync).
  - **Dataflow** (`/dataflow`): Visualise the job type fan-out with an interactive diagram.
- The layout uses responsive CSS; the sidebar collapses on narrow screens.

## Keyboard Shortcuts

The web console supports a limited set of custom keyboard shortcuts to streamline common tasks. For a complete and up-to-date listing, see [Keyboard Shortcuts in Markdown Magpie](keyboard-shortcuts-in-markdown-magpie.md).

Note: The Markdown Magpie pitch deck (`presentation/`) implements its own keyboard navigation (arrow keys, Home, End, O, F) for slide advancement. Those shortcuts do **not** apply to the web console.

## Ask Page Interaction Flow

1. The user types a question into a `<textarea>` element.
2. Optionally, the user selects a **knowledge flow** from a dropdown. The dropdown shows all configured flows, with an **"Auto (let Magpie decide)"** option as the default. When `auto` is selected, the question is routed normally. If a specific flow is chosen, the question is pinned to that flow.
3. Clicking **Ask** sends a `POST /api/ask` request. The request body includes the `question` and, if a flow other than `auto` was selected, the `flow` parameter.
4. The API returns HTTP 202 with a job ID and question ID.
5. The UI shows a loading indicator and polls the job status or displays a link to wait.
6. When the watcher completes the job, the answer (with citations) appears as formatted Markdown, rendered using `react-markdown`.
7. If the answer has a `flowSelectionRequired` field (meaning the router could not decide which flow to use), the UI displays an inline picker listing the available flows. The user can click a flow to re‑ask the same question pinned to that flow. This creates a fresh question/job – the original unanswered question remains as-is.

This async flow is central to the product: all AI work runs in a background watcher, so the UI never blocks. The lack of immediate response on Enter is a tradeoff for reliability and queue management. The flow dropdown lets callers control which knowledge area the question is answered from, and the re‑ask mechanism handles the case where the router is uncertain.

## Design Rationale

- **Simplicity over complexity**: The console is a tool, not a marketing site. Excessive styling or animations would distract from the review tasks.
- **Async-first**: Since `POST /api/ask` is enqueue-only, the UI must accommodate a wait. The current design avoids hiding this — it shows the job state rather than simulating instant answers.
- **Radix UI primitives**: The project uses Radix UI for accessible components (tooltips, etc.) and React Flow for visualising knowledge flows (Crunch (scheduled maintenance) and Proposal pipelines, as well as the Dataflow diagram). These libraries provide a solid baseline for accessibility and interaction.
- **Styling architecture**: The UI is styled entirely with Emotion CSS-in-JS using a typed design-token theme (`src/theme/`) and a library of primitives (`src/components/ui/` — Button, Badge, Chip, Surface, Field, Tabs, Stack/Row). There are no `.css` files: every component reads from the theme via `p => p.theme.*`. This prevents style drift and makes theme changes predictable.
- **No inline editing**: To keep the UI predictable, all editing of knowledge content is performed via proposals and published pull requests, not directly in the web console.

## Future Considerations

- The lack of documentation for UI/UX decisions is itself a gap — this document begins to fill it. Additional sections on error states, empty states, and mobile behaviour may be added as the product evolves.
