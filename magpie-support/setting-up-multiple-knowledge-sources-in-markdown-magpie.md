---
title: Setting Up Multiple Knowledge Sources
status: draft
---

# Setting Up Multiple Knowledge Sources

Markdown Magpie can ingest content from multiple sources and route them to one or more curated knowledge bases (destinations). This guide walks you through the complete configuration, from environment variables to indexing, with real examples.

## Overview

Each knowledge source is a location where raw Markdown lives. Each destination is a curated knowledge base that the system indexes for answering questions. **Flows** link sources to destinations. You configure all of this in your `.env` file.

> **Note:** The destination must already exist as a local folder or git checkout. For git destinations, use the same shape as the git source but with a `url`.

## How Sources Work

Sources are defined in the environment variable `KNOWLEDGE_SOURCES` as a JSON array of source objects. Each source has:

- `id` – unique identifier within your configuration
- `name` – human‑readable label
- `kind` – one of `local`, `git`, `internet`, or `agent`
- Additional fields depending on the kind (see below)

You then connect one or more source IDs to a destination via `KNOWLEDGE_FLOWS`. The destination is defined in `KNOWLEDGE_DESTINATIONS`.

> **Legacy fallback:** The older `KNOWLEDGE_REPOSITORIES` and `KNOWLEDGE_REPO_PATH` environment variables are still supported when the new variables are not set. However, the source/destination/flow model described above is the preferred configuration.

## Configuring Sources and Destinations

Use the following environment variables:

- `KNOWLEDGE_SOURCES` – JSON array of source objects.
- `KNOWLEDGE_DESTINATIONS` – JSON array of destination objects.
- `KNOWLEDGE_FLOWS` – JSON array of flow objects that connect sources to a single destination.

Each source object can be:

- **Local folder** – `{"id":"my-source","name":"My Source","path":"knowledge-bases/my-source"}`
- **Git repository** – `{"id":"git-source","name":"Git Source","url":"https://github.com/org/repo.git","subpath":"docs"}`
- **Internet URL** – `{"id":"web-source","name":"Web Docs","kind":"internet","url":"https://example.com/docs"}`
- **Agent** – `{"id":"agent-source","name":"Agent Knowledge","kind":"agent"}`

Destinations are similar but represent the output location where curated content lives.

### Example: One Flow, Two Local Sources

Suppose you have two local directories with Markdown files:

```text
knowledge-bases/
  product/
  engineering/
```

You want to combine both into a single destination for answering questions. Configure:

```env
KNOWLEDGE_SOURCES=[{"id":"product","name":"Product Docs","path":"knowledge-bases/product"},{"id":"engineering","name":"Engineering Docs","path":"knowledge-bases/engineering"}]
KNOWLEDGE_DESTINATIONS=[{"id":"combined","name":"Combined KB","path":"knowledge-bases/combined"}]
KNOWLEDGE_FLOWS=[{"id":"main","name":"Main KB","sourceIds":["product","engineering"],"destinationId":"combined"}]
```

This means all Markdown from `product` and `engineering` is considered raw source material. The destination `combined` is the curated knowledge base that will be indexed. When you index the flow (`POST /api/knowledge/repositories/index` with `{"flowId":"main"}`), only the destination directory is scanned for answering questions; the sources are used for drafting improvements.

### Example: Remote Git Source, Local Destination

If your raw documentation lives in a GitHub repo and you want to maintain a local curated copy:

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts
KNOWLEDGE_SOURCES=[{"id":"flowerbi","name":"FlowerBI Source","url":"https://github.com/danielearwicker/flowerbi.git","subpath":"src"}]
KNOWLEDGE_DESTINATIONS=[{"id":"flowerbi-docs","name":"FlowerBI Docs","url":"https://github.com/AdamAwan/flowerbi-doc-test.git","subpath":"docs"}]
KNOWLEDGE_FLOWS=[{"id":"flowerbi","name":"FlowerBI KB","sourceIds":["flowerbi"],"destinationId":"flowerbi-docs"}]
```

The source repository is cloned into `MAGPIE_CHECKOUT_ROOT` on startup. The destination is a separate repository where reviewed proposals will be pushed.

### Example: Multiple Source Kinds

You can mix local, git, agent, and internet sources in one flow:

```env
KNOWLEDGE_SOURCES=[{"id":"local-docs","name":"Local Docs","path":"knowledge-bases/internal"},{"id":"github-api","name":"API Docs","url":"https://github.com/org/api-docs.git","subpath":"docs"},{"id":"external-guide","name":"External Guide","kind":"internet","url":"https://example.com/guide.md"},{"id":"agent-knowledge","name":"Agent Knowledge","kind":"agent"}]
KNOWLEDGE_DESTINATIONS=[{"id":"all-knowledge","name":"All Knowledge","path":"knowledge-bases/combined"}]
KNOWLEDGE_FLOWS=[{"id":"full-flow","name":"Full Flow","sourceIds":["local-docs","github-api","external-guide","agent-knowledge"],"destinationId":"all-knowledge"}]
```

### Example: Two Git Repositories + an Agent Source

If you maintain documentation across separate repositories (e.g., user guide and API reference) and also want to include agent‑generated proposals, use:

```env
KNOWLEDGE_SOURCES=[
  {"id":"user-guide","name":"User Guide","kind":"git","url":"https://github.com/org/user-guide.git","subpath":"docs"},
  {"id":"api-ref","name":"API Reference","kind":"git","url":"https://github.com/org/api-ref.git","subpath":"reference"},
  {"id":"agent-feedback","name":"Agent Feedback","kind":"agent"}
]

KNOWLEDGE_DESTINATIONS=[
  {"id":"combined-docs","name":"Combined Docs","kind":"local","path":"knowledge-bases/combined"}
]

KNOWLEDGE_FLOWS=[
  {"id":"full-kb","name":"Full Knowledge Base","sourceIds":["user-guide","api-ref","agent-feedback"],"destinationId":"combined-docs"}
]
```

- `agent` kind requires no additional fields; it tells Magpie to treat agent‑generated proposals as source material.
- The `subpath` in git sources lets you point to a subdirectory within the cloned repo (e.g., `docs` or `src`).

### Example: Single Source, Multiple Destinations (Advanced)

You can also propagate the same source material to multiple curated KBs by defining multiple flows:

```env
KNOWLEDGE_SOURCES=[
  {"id":"core","name":"Core Docs","kind":"local","path":"knowledge-bases/core"}
]

KNOWLEDGE_DESTINATIONS=[
  {"id":"public-kb","name":"Public KB","kind":"local","path":"knowledge-bases/public"},
  {"id":"internal-kb","name":"Internal KB","kind":"local","path":"knowledge-bases/internal"}
]

KNOWLEDGE_FLOWS=[
  {"id":"public","name":"Public Flow","sourceIds":["core"],"destinationId":"public-kb"},
  {"id":"internal","name":"Internal Flow","sourceIds":["core"],"destinationId":"internal-kb"}
]
```

Each flow can be indexed independently:

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"public"}'
```

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"internal"}'
```

## Indexing the Configured Flow

After setting the environment variables and starting the API and watcher (see the main README), index the destination:

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"main"}'
```

Replace `"main"` with your flow's `id`. The API indexes the destination repository/folder. If you have multiple flows, index each one separately.

## Verification

Check that your knowledge is searchable:

```bash
curl -s 'http://localhost:4000/api/knowledge/stats'
```

To search across all indexed content:

```bash
curl -s 'http://localhost:4000/api/knowledge/search?q=your+keyword'
```

You can also verify your configuration is loaded correctly:

```bash
curl -s http://localhost:4000/api/config | jq .
```

Look for `repositories` and the configured flows in the output.

## Additional Notes

- **`KNOWLEDGE_REPOSITORIES` and `KNOWLEDGE_REPO_PATH`** are legacy fallbacks; prefer the explicit source/destination/flow variables.
- **`MAGPIE_CHECKOUT_ROOT`** must be set to a writable local directory when using git sources or destinations.
- **Agent sources** (`{"kind":"agent"}`) are for knowledge that an AI agent (like Claude) feeds back; they are not indexed directly.
- **Internet sources** are fetched at index time; the content is downloaded and treated as raw source.
- Sources with kind `agent` or `internet` are used for drafting proposals but are **not indexed** into the answer corpus.
- **Destination existence**: The destination must already exist as a local folder or git checkout. For git destinations, use the same shape as the git source but with a `url`.

For full details on ingestion and the API, see [ingestion.md](../docs/ingestion.md) and [api.md](../docs/api.md).

## Related

- [Integrations and Data Sources](./integrations-and-data-sources-in-markdown-magpie.md)
- [Knowledge Flows](./managing-knowledge-flows-in-magpie.md)
- [Ingestion](./ingestion.md)