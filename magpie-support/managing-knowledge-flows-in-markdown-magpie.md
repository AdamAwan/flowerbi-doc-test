---
title: Managing Knowledge Flows in Markdown Magpie
status: draft
---

# Managing Knowledge Flows in Markdown Magpie

Knowledge flows are the core pipeline that connects your raw documentation sources to a curated knowledge base. They define **what** content is read, **how** it is processed, and **where** the resulting knowledge is stored and served. This guide explains how to set up, configure, and manage knowledge flows as a customer of Markdown Magpie.

## Overview

A knowledge flow is a named pipeline that links one or more **sources** (where your raw Markdown lives) to a **destination** (the curated repository that Magpie searches and maintains). The flow is used by the system to:

- Clone or sync source repositories.
- Index the destination repository for answering questions.
- Propose and publish updates to the destination when gaps are detected.

You manage flows through environment variables and the HTTP API / web console.

## Configuring a Knowledge Flow

Flows are defined via the environment variable `KNOWLEDGE_FLOWS`. Each flow has an ID, a display name, a list of source IDs, and a destination ID. Sources and destinations are defined separately:

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts

KNOWLEDGE_SOURCES=[
  {"id":"flowerbi-code","name":"FlowerBI Source Code","kind":"git","url":"https://github.com/example/flowerbi.git","subpath":"src"},
  {"id":"external-guide","name":"External Guide","kind":"internet","url":"https://example.com/guide.md"}
]

KNOWLEDGE_DESTINATIONS=[
  {"id":"flowerbi-docs","name":"FlowerBI Docs","kind":"git","url":"https://github.com/example/flowerbi-docs.git","subpath":"docs"}
]

KNOWLEDGE_FLOWS=[
  {"id":"flowerbi","name":"FlowerBI Knowledge Base","sourceIds":["flowerbi-code","external-guide"],"destinationId":"flowerbi-docs"}
]
```

### Supported Source Kinds

- `local` – a folder on the server (e.g., `"path":"knowledge-bases/cats"`)
- `git` – a remote Git repository, optionally with a `subpath` to a subfolder
- `internet` – a plain HTTP/HTTPS URL that returns Markdown
- `agent` – the agent’s own knowledge (no URL needed)

### Supported Destination Kinds

Destinations are always writable Git repositories that Magpie will publish proposals and edits to. They must be Git repositories configured with push access.

### Legacy Configuration

If `KNOWLEDGE_SOURCES` and `KNOWLEDGE_DESTINATIONS` are not set, the system falls back to the older `KNOWLEDGE_REPOSITORIES` and `KNOWLEDGE_REPO_PATH` variables. New deployments should use the flow-based configuration.

## Indexing a Flow

Once a flow is configured, you must index its destination so Magpie can answer questions from it. Indexing parses the Markdown, splits it into sections, and stores them for retrieval.

### Using the API

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"flowerbi"}'
```

This indexes the destination repository of the flow with ID `flowerbi`. The API returns a summary including document and section counts.

### Using the Web Console

Navigate to **Knowledge > Repositories** in the web console. Click the **Index** button next to your flow to trigger indexing.

## Viewing Flow Status

### API Endpoints

- `GET /api/knowledge/repositories` – list all indexed repositories (each corresponds to a flow’s destination).
- `GET /api/knowledge/stats` – get total document and section counts.
- `GET /api/knowledge/documents` – list all indexed documents across flows.

### Web Console

The **Knowledge** page shows each flow’s index status, including last indexed commit and counts.

## Managing Sources and Destinations Over Time

To add, remove, or modify flows, update the environment variables and restart the API. The system will re-sync Git repositories on every startup. For runtime changes without restart, you can use `/api/config` to reload configuration, but this is primarily for AI provider switching. Changes to flow definitions currently require a restart.

## Source Change Synchronisation

Magpie includes a background scheduler task called `source-change-sync` that watches each flow’s Git sources and automatically rewrites knowledge-base documents when a source change has gone out of date. This runs every 10 minutes by default. You can adjust the interval via the Crunch settings page.

## Next Steps

- **Ask questions** against your indexed flow using `POST /api/ask` or the `kb.ask` MCP tool.
- **View gaps** in your knowledge base with `GET /api/gaps/candidates` or the Gaps page in the web console.
- **Trigger Crunch** to consolidate or split documents by using `POST /api/crunch/run`.
- **Manage proposals** – once gaps are identified, you can generate proposals and publish them as pull requests to your destination repository.

## Troubleshooting

- **Flow not appearing in stats** – Make sure the flow ID is defined in `KNOWLEDGE_FLOWS` and that you have indexed it.
- **Index fails** – Check that the destination repository URL is accessible and that `MAGPIE_CHECKOUT_ROOT` is writable.
- **No answers returned** – Confirm that indexing completed successfully and that the retrieval mode is active (check `GET /api/config` for `retrieval.mode`).
- **PRs not being created** – You may need to set `GITHUB_TOKEN` or equivalent Git host token for automatic publishing.

---

*For detailed API reference, see the [HTTP API documentation](docs/api.md). For ingestion specifics, see [Markdown Ingestion](docs/ingestion.md).*