---
title: Adding Multiple Sources to Markdown Magpie – Practical Examples
status: draft
---

# Adding Multiple Sources to Markdown Magpie – Practical Examples

Markdown Magpie supports ingesting content from multiple sources into a single knowledge base flow. This guide provides **real, copy‑paste‑ready examples** for configuring two, three, or more sources of different kinds (local folders, git repositories, internet URLs, agent‑sourced content).

## How Sources Work

Sources are defined in the environment variable `KNOWLEDGE_SOURCES` as a JSON array of source objects. Each source has:

- `id` – unique identifier within your configuration
- `name` – human‑readable label
- `kind` – one of `local`, `git`, `internet`, or `agent`
- Additional fields depending on the kind (see below)

You then connect one or more source IDs to a destination via `KNOWLEDGE_FLOWS`. The destination is defined in `KNOWLEDGE_DESTINATIONS`.

## Example 1: Local Folder + Git Repository + Internet Source

This example configures three sources:

- A local Markdown folder (`knowledge-bases/product`)
- A remote git repository (FlowerBI source, cloned into `MAGPIE_CHECKOUT_ROOT`)
- An internet documentation site (fetched at index time)

All three feed into a single curated destination (`kb-output`).

### Environment Variables

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts

KNOWLEDGE_SOURCES=[
  {"id":"local-docs","name":"Local Product Docs","kind":"local","path":"knowledge-bases/product"},
  {"id":"flowerbi","name":"FlowerBI Source","kind":"git","url":"https://github.com/danielearwicker/flowerbi.git","subpath":"src"},
  {"id":"api-docs","name":"API Documentation","kind":"internet","url":"https://example.com/api-docs"}
]

KNOWLEDGE_DESTINATIONS=[
  {"id":"kb-output","name":"Curated KB","kind":"local","path":"knowledge-bases/kb-output"}
]

KNOWLEDGE_FLOWS=[
  {"id":"main","name":"Main Knowledge Flow","sourceIds":["local-docs","flowerbi","api-docs"],"destinationId":"kb-output"}
]
```

> **Note:** The destination must already exist as a local folder or git checkout. For git destinations, use the same shape as the git source example but with a `url`.

### Indexing the Flow

After starting the API and watcher, trigger indexing:

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"main"}'
```

This indexes the **destination** (`kb-output`). The sources are used only during proposal generation and gap detection; they are **not** directly queryable via `/api/ask`.

## Example 2: Two Git Repositories + an Agent Source

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

## Example 3: Single Source, Multiple Destinations (Advanced)

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

## Verifying Your Configuration

After everything is running, check the health endpoint to see resolved sources:

```bash
curl -s http://localhost:4000/api/config | jq .
```

Look for `repositories` and the configured flows.

## Related

- [Integrations and Data Sources](./integrations-and-data-sources-in-markdown-magpie.md)
- [Ingestion](./ingestion.md)
- [Knowledge Flows](./managing-knowledge-flows-in-magpie.md)