---
title: Knowledge Source and Destination Configuration
status: draft
---

# Environment Variables for Configuring Multiple Sources

To configure multiple knowledge sources, destinations, and flows, use the following environment variables:

- `KNOWLEDGE_SOURCES` — JSON array defining read sources.
- `KNOWLEDGE_DESTINATIONS` — JSON array defining curated KB destinations.
- `KNOWLEDGE_FLOWS` — JSON array defining links between sources and destinations.

Each entry in `KNOWLEDGE_SOURCES` must have an `id`, `name`, and the connection properties specific to its kind. Supported kinds: `local`, `git`, `internet`, `agent`.

Similarly, `KNOWLEDGE_DESTINATIONS` defines where reviewed proposals are written. They follow the same shape but usually point to a git repository or local folder.

`KNOWLEDGE_FLOWS` connects a set of source IDs to a single destination ID, plus a unique `id` and `name` for the flow.

## Example: Two Git Sources and One Destination

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts
KNOWLEDGE_SOURCES=[{"id":"flowerbi","name":"FlowerBI Source","url":"https://github.com/danielearwicker/flowerbi.git","subpath":"src"},{"id":"agent","name":"Agent Knowledge","kind":"agent"}]
KNOWLEDGE_DESTINATIONS=[{"id":"flowerbi-docs","name":"FlowerBI Docs","url":"https://github.com/AdamAwan/flowerbi-doc-test.git","subpath":"docs"}]
KNOWLEDGE_FLOWS=[{"id":"flowerbi","name":"FlowerBI KB","sourceIds":["flowerbi","agent"],"destinationId":"flowerbi-docs"}]
```

This configures two sources (a git repo and an agent knowledge source) and one destination. The flow uses both sources to feed the destination.

## Example: Local Folder Source

```env
KNOWLEDGE_SOURCES=[{"id":"docs","name":"Product Docs","path":"knowledge-bases/product"}]
KNOWLEDGE_DESTINATIONS=[{"id":"docs","name":"Product Docs","path":"knowledge-bases/product"}]
KNOWLEDGE_FLOWS=[{"id":"docs","name":"Product Docs","sourceIds":["docs"],"destinationId":"docs"}]
```

## Format

Each JSON object must include `id` (unique identifier), `name` (human-readable label), and the appropriate key for the source kind:

- `local`: requires `path` (relative or absolute).
- `git`: requires `url` and optional `subpath`.
- `internet`: requires `url` and `kind: "internet"`.
- `agent`: use `kind: "agent"` (no path or url).

For destinations, only `local` and `git` are supported — destinations cannot be `internet` or `agent`.

## Backwards Compatibility

If `KNOWLEDGE_SOURCES` and `KNOWLEDGE_DESTINATIONS` are not set, the system falls back to legacy variables `KNOWLEDGE_REPOSITORIES` and `KNOWLEDGE_REPO_PATH`. This is deprecated.

## Reference

- Full documentation of the ingestion model is in [docs/ingestion.md](./docs/ingestion.md).
- Environment variable list: see `.env.example` in the repository root.