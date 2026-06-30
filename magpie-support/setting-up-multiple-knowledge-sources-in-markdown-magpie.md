---
title: Setting Up Multiple Knowledge Sources
status: draft
---

# Setting Up Multiple Knowledge Sources

Markdown Magpie can ingest content from multiple sources and route them to one or more curated knowledge bases (destinations). This guide walks you through the complete configuration, from environment variables to indexing, with real examples.

## Overview

Each knowledge source is a location where raw Markdown lives. Each destination is a curated knowledge base that the system indexes for answering questions. **Flows** link sources to destinations. You configure all of this in your `.env` file.

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

## Additional Notes

- **`KNOWLEDGE_REPOSITORIES` and `KNOWLEDGE_REPO_PATH`** are legacy fallbacks; prefer the explicit source/destination/flow variables.
- **`MAGPIE_CHECKOUT_ROOT`** must be set to a writable local directory when using git sources or destinations.
- **Agent sources** (`{"kind":"agent"}`) are for knowledge that an AI agent (like Claude) feeds back; they are not indexed directly.
- **Internet sources** are fetched at index time; the content is downloaded and treated as raw source.

For full details on ingestion and the API, see [ingestion.md](../docs/ingestion.md) and [api.md](../docs/api.md).