---
title: Managing Knowledge Flows
owner: magpie-ops
status: draft
tags: [flows, indexing, proposals, crunch]
review_cycle_days: 90
---

# Managing Knowledge Flows

Knowledge flows are the core pipeline connecting your raw documentation sources to the curated knowledge base that Magpie searches and maintains. This guide covers everything about flows: creating them, indexing content, managing documents, keeping the knowledge base updated, and using the gap pipeline and crunch features.

> For environment variable configuration (sources, destinations, flows, embeddings, etc.), see the [Configuration Reference](configuration-reference.md).

## Understanding Flows

A knowledge flow is a named pipeline that links one or more **sources** (where raw Markdown lives) to a **destination** (the curated repository indexed for answering). The flow drives:
- Cloning/syncing source repositories.
- Indexing the destination for question answering.
- Proposing and publishing updates when gaps are detected.

## Creating and Configuring a Flow

Define your flow using environment variables as described in the [Configuration Reference](configuration-reference.md#knowledge-sources-destinations-and-flows). After setting `KNOWLEDGE_SOURCES`, `KNOWLEDGE_DESTINATIONS`, and `KNOWLEDGE_FLOWS`, restart the API.

## Indexing a Flow

Indexing parses the Markdown files in the flow’s destination, extracts sections, and optionally generates embeddings.

### Using the API

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"my-flow"}'
```

Response:
```json
{
  "repository": "my-destination",
  "documentCount": 12,
  "sectionCount": 48,
  "commitSha": "abc123"
}
```

### Using the Web Console

Navigate to **Knowledge > Repositories**, click **Index** next to your flow.

## Viewing Flow Status

- `GET /api/knowledge/repositories` – list indexed repositories.
- `GET /api/knowledge/stats` – total document and section counts.
- `GET /api/knowledge/documents` – list all documents across flows.

In the web console, the **Knowledge** page shows each flow’s last indexed commit and counts.

## Managing Documents in the Knowledge Base

### Adding Documents

Add Markdown files directly to the destination repository via Git, then re-index. Alternatively, use the gap pipeline to generate a proposal that adds a new document.

**Manual addition:**
```bash
cd $MAGPIE_CHECKOUT_ROOT/my-destination-docs
echo '# New Topic' > docs/new-topic.md
git add . && git commit -m "Add new-topic.md" && git push
```
Then re-index the flow.

**Via proposal:** See [The Gap Pipeline](#the-gap-pipeline-and-flows).

### Removing Documents

Delete the file from the destination repository, commit, push, and re-index. There is no dedicated API for single-document removal.

```bash
cd $MAGPIE_CHECKOUT_ROOT/my-destination-docs
rm docs/old-file.md
git commit -am "Remove old-file.md" && git push
curl -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"my-flow"}'
```

For batch removal, delete multiple files, then re-index. Or use a Crunch proposal that includes deletions.

## Keeping Flows Updated

### Source Change Sync

The `source-change-sync` background job (every 10 minutes) watches source repositories for new commits and rewrites destination documents accordingly. Configure this in Crunch settings.

### Manual Re-index

Force a re-index at any time:
```bash
curl -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"my-flow"}'
```

### Full Reset

For demos, use:
```bash
curl -X POST http://localhost:4000/api/admin/reset
```
**Warning:** Destructive and unauthenticated; never expose in production.

## The Gap Pipeline and Flows

Every question asked via `/api/ask` or MCP `kb.ask` is logged. Low-confidence answers and user-flagged gaps are clustered per flow. The `gaps-to-pull-requests` reconciler then:
1. Groups related gaps into clusters.
2. Drafts Markdown proposals to fill gaps.
3. Publishes to a Git branch in the flow's destination.
4. Opens a PR if a host token is configured.

Review proposals via `GET /api/proposals` or the web console's **Proposals** page.

## Crunch: Scheduled Knowledge Base Tidying

Crunch analyses destination documents, identifies opportunities to split/consolidate/deletion, and builds a plan. After review, the plan can be published as a branch.

```bash
curl -X POST http://localhost:4000/api/crunch/run \
  -H 'content-type: application/json' \
  -d '{"flowId":"my-flow"}'

curl -s http://localhost:4000/api/crunch/runs

curl -X POST http://localhost:4000/api/crunch/runs/:id/publish
```

## Verifying Knowledge Base State

```bash
curl -s http://localhost:4000/api/knowledge/stats
```

## Best Practices

- Use separate Git repos for sources and destinations.
- Keep the destination human‑readable; it’s your authoritative KB.
- For flow without sources, omit `sourceIds`; let the gap pipeline drive proposals.
- Monitor `GET /api/gaps/clusters` for frequently missing topics.
- Configure `GITHUB_TOKEN` for automated PRs in production.
- Use descriptive, unique headings.
- Keep sections focused on a single topic.

## Troubleshooting

| Symptom | Likely Cause | Remedy |
|---|---|---|
| `/api/ask` returns low confidence | Destination not indexed or embeddings missing | Re-index and check embedding config. |
| Proposals not appearing as PRs | No host token | Set `GITHUB_TOKEN` and ensure `pull-request-refresh` scheduler runs. |
| Crunch plan says “no changes needed” | Destination already well‑structured | Configure smaller interval or trigger full analysis. |
| Web console shows no flows | Env vars not set | Verify `KNOWLEDGE_FLOWS` in `.env` and restart API. |
| Index returns “0 documents” | Destination checkout not synced or path wrong | Check `MAGPIE_CHECKOUT_ROOT` and API startup logs. |
| New document not found after adding | Index not run | Run index endpoint again. |
| Re-index takes long | Embedding pass for many new sections | Wait; idempotent background embedding. |
| “local_path_not_allowed” error | Trying to index arbitrary path without a flow | Use a flow ID defined in `KNOWLEDGE_FLOWS`. |
| Changes not reflected after re-index | Browser caching | Use cache-busting or re-query API. |
| Hybrid retrieval not active | Embedding credentials incomplete or `KNOWLEDGE_STORE` not set | Check config. |

## Reference

- [Configuration Reference](configuration-reference.md) – all environment variable details.
- [Quick Start](quick-start.md) – minimal setup guide.
- [Permissions and Access Controls](permissions-and-access-controls-in-markdown-magpie.md) – authentication and authorization.
- [Answer Confidence](understanding-and-improving-answer-confidence-in-markdown-ma.md) – understanding and improving answer quality.
