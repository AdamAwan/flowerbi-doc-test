---
title: Managing Knowledge Flows in Magpie
status: draft
---

# Managing Knowledge Flows in Magpie

Knowledge flows connect raw source repositories and curated documentation destinations. They define which data is indexed for question answering, which sources are monitored for updates, and how proposed knowledge changes (gaps, crunches) are published.

## Understanding Flows

A flow is a named pipeline consisting of:
- **One or more sources**: Read-only repositories, internet sites, or agent-provided knowledge.
- **One destination**: The curated knowledge base (KB) that is indexed for `/api/ask` and MCP tools.

The destination is the answer corpus. Sources feed into it, but only the destination is indexed.

## Configuring Flows

Flows are defined in the environment variables `KNOWLEDGE_SOURCES`, `KNOWLEDGE_DESTINATIONS`, and `KNOWLEDGE_FLOWS`. These are set in the `.env` file (or passed to containers).

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts

KNOWLEDGE_SOURCES=[
  {"id":"flowerbi","name":"FlowerBI Source","url":"https://github.com/danielearwicker/flowerbi.git","subpath":"src"}
]
KNOWLEDGE_DESTINATIONS=[
  {"id":"flowerbi-docs","name":"FlowerBI Docs","url":"https://github.com/AdamAwan/flowerbi-doc-test.git","subpath":"docs"}
]
KNOWLEDGE_FLOWS=[
  {"id":"flowerbi","name":"FlowerBI KB","sourceIds":["flowerbi"],"destinationId":"flowerbi-docs"}
]
```

Each entry must have a unique `id`. Supported source kinds:
- `local`: `{"path":"knowledge-bases/cats"}`
- `git`: `{"url":"...","subpath":"docs"}`
- `internet`: `{"kind":"internet","url":"https://example.com/docs"}`
- `agent`: `{"kind":"agent"}`

Remote git sources and destinations are cloned or fast-forward pulled into `MAGPIE_CHECKOUT_ROOT` during API startup.

For backwards compatibility, `KNOWLEDGE_REPOSITORIES` and `KNOWLEDGE_REPO_PATH` are still accepted when `KNOWLEDGE_SOURCES`/`KNOWLEDGE_DESTINATIONS` are not set.

## Indexing a Flow

Once a flow is configured, you must index its destination before users can ask questions. Indexing parses Markdown files, extracts sections, and (optionally) generates embeddings.

### Using the API

```bash
curl -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"flowerbi"}'
```

This indexes only the **destination** of the flow. The response includes the number of documents and sections indexed.

### After Indexing

The API automatically runs background embedding for any section whose vector is missing. In `direct` mode, questions are answered synchronously. In `queue` mode, a watcher must be running to process answer jobs.

## Viewing Flow Status

### Via API

```bash
# List all indexed repositories
curl -s http://localhost:4000/api/knowledge/repositories

# Get document and section counts
curl -s http://localhost:4000/api/knowledge/stats
```

### Via Web Console

Open the web console (default at `http://localhost:3000`) and navigate to the **Knowledge** page. You can see indexed documents, search the KB, and trigger re-indexing.

## Keeping Flows Updated

### Source Change Sync

The `source-change-sync` background job (scheduled by default every 10 minutes) watches each flow's git sources. When a source has new commits, it rewrites the corresponding destination documents and re-indexes them. You can disable this in the Crunch settings page if you prefer manual control.

### Manual Re-index

To force a re-index of a flow (e.g., after changing destination documents by hand), call:

```bash
curl -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"my-flow"}'
```

## The Gap Pipeline and Flows

Every question asked via `/api/ask` or the MCP tool `kb.ask` is logged. Low-confidence answers and user-flagged gaps are clustered per flow. The `gaps-to-pull-requests` reconciler then:
1. Groups related gaps into clusters.
2. Drafts Markdown proposals that fill those gaps.
3. Publishes them to a Git branch in the flow's destination repository.
4. Opens a pull request (if a token for the remote host is configured) and advances the proposal as the PR is merged.

Proposals are always relative to the flow's destination. You can review draft proposals via `GET /api/proposals` or the web console's **Proposals** page.

## Crunch: Scheduled Knowledge Base Tidying

A separate **Crunch** flow runs on a per-flow schedule. It analyses the destination documents, identifies opportunities to split large files or consolidate scattered content, and builds a plan. After operator review, the plan can be published as a branch.

```bash
# Trigger crunch now for a flow
curl -X POST http://localhost:4000/api/crunch/run \
  -H 'content-type: application/json' \
  -d '{"flowId":"flowerbi"}'

# List recent runs
curl -s http://localhost:4000/api/crunch/runs

# Publish a completed crunch plan
curl -X POST http://localhost:4000/api/crunch/runs/:id/publish
```

## Best Practices

- Use separate git repositories for sources and destinations to control access and review cycles.
- Keep the destination repository human‑readable: it is the authoritative knowledge base for your customers.
- If a flow has no sources (e.g., you write documentation directly in the destination repo), omit `sourceIds` and let the gap pipeline propose changes from user questions alone.
- Monitor the `GET /api/gaps/clusters` endpoint to see which missing topics are most frequently asked.
- For production deployments, configure `GITHUB_TOKEN` (or equivalent) so proposals automatically become pull requests.

## Troubleshooting

| Symptom | Likely Cause | Remedy |
|---|---|---|
| `/api/ask` returns low confidence | Destination not indexed or embeddings missing | Re-index the flow and check embedding configuration (see [Ingestion](../ingestion.md)). |
| Proposals not appearing as PRs | No git host token configured | Set `GITHUB_TOKEN` or equivalent and ensure the `pull-request-refresh` scheduler is running. |
| Crunch plan says “no changes needed” | Destination already well‑structured | Configure smaller interval or manually trigger a full analysis. |
| Web console shows no flows | Environment variables not set | Verify `KNOWLEDGE_FLOWS` in `.env` and restart the API. |

## Reference

- [HTTP API Reference](../api.md) — endpoints for managing flows, indexing, and proposals.
- [Ingestion Configuration](../ingestion.md) — detailed configuration of sources, destinations, and flows.
- [Architecture Overview](../architecture.md) — the product loop and how flows fit in.
