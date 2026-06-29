---
title: Quick Start — Legacy Reference (Consolidated)
owner: magpie-ops
status: archived
tags: [getting-started, quickstart, deprecated, old, superseded]
review_cycle_days: 90
---

> **Note:** This guide has been consolidated into the [Getting Started: Onboarding and Indexing Content into Markdown Magpie](getting-started-onboarding-and-indexing-content-into-markdow.md) document. Please refer to the Getting Started guide for the most current and comprehensive instructions. The content below is retained only for reference and links.

For quick reference, the main steps are:
1. Clone and install
2. Configure environment (set `AI_EXECUTION_MODE=queue`, `AUTH_REQUIRED=false`)
3. Start Postgres via Docker
4. Run migrations
5. Configure knowledge flows
6. Start the API
7. (Queue mode) Start the watcher
8. Index content
9. Ask a question
10. (Optional) Start web console

See the full [Getting Started](getting-started-onboarding-and-indexing-content-into-markdow.md) guide for detailed instructions, including embedding configuration and troubleshooting.

## Queue Mode Specifics

For queue mode, Markdown Magpie enqueues AI jobs that a separate **watcher** process claims and completes. This section covers setup steps specific to queue mode.

### Start the Watcher (Required for Queue Mode)

Start the watcher in a separate terminal so it can claim and complete AI jobs:

```bash
AUTH_REQUIRED=false WATCHER_API_CLIENT_ID= WATCHER_API_CLIENT_SECRET= \
  MAGPIE_CHECKOUT_ROOT="$PWD/.magpie/checkouts" npm run dev:watcher
```

> The watcher is **required** for all generative work: answering questions, drafting proposals, publishing, and maintenance jobs. Without it, `POST /api/ask` will return `202` and the question will never be answered. The mock provider works out of the box — no additional credentials needed.

### Troubleshooting (Queue Mode)

| Problem | Solution |
|---|---|
| `/ask` returns `202` but never completes | The watcher is not running. Start it (see above) and retry the question. |
| Watcher logs `Capability … not ready` | Check that the environment variables for the chosen provider are set correctly. For mock, no extra variables are needed. |
| `401` on API calls | Locally, set `AUTH_REQUIRED=false` in `.env` or as an environment variable. |

For all other setup steps (clone, install, configure environment, start dependencies, run migrations, configure flows, index, and the web console), please follow the [Getting Started guide](getting-started-onboarding-and-indexing-content-into-markdow.md).
