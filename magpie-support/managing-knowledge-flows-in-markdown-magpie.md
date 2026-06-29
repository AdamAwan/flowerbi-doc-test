---
title: Managing Knowledge Flows in Markdown Magpie
status: draft
---

# Managing Knowledge Flows in Markdown Magpie

Knowledge flows are the core pipeline that connects your raw documentation sources to a curated knowledge base. They define **what** content is read, **how** it is processed, and **where** the resulting knowledge is stored and served. This guide covers setting up flows, indexing content, managing documents, and maintaining the knowledge base over time.

## Understanding Flows

A knowledge flow is a named pipeline that links one or more **sources** (where your raw Markdown lives) to a **destination** (the curated repository that Magpie searches and maintains). The flow is used by the system to:

- Clone or sync source repositories.
- Index the destination repository for answering questions.
- Propose and publish updates to the destination when gaps are detected.

| Concept | Description |
|---------|-------------|
| **Source** | A read‑only location of raw Markdown (e.g., your project’s code repository, an internet URL, or an agent knowledge folder). |
| **Destination** | The repository where reviewed, final Markdown lives. This is what Magpie indexes for retrieval and answering. |
| **Flow** | A named connection between sources and a destination. Magpie monitors sources for changes and updates the destination accordingly. |

## Configuring a Knowledge Flow

Flows are defined in the environment variables `KNOWLEDGE_SOURCES`, `KNOWLEDGE_DESTINATIONS`, and `KNOWLEDGE_FLOWS`. These are set in the `.env` file (or passed to containers).

### Required: Checkout Root

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts
```

This is the local directory where remote repositories are cloned. Create it if it doesn’t exist.

### Define Sources

Use `KNOWLEDGE_SOURCES` as a JSON array. Each source has at minimum an `id` and `name`. The `kind` determines how it is accessed.

```env
KNOWLEDGE_SOURCES=[
  {"id":"flowerbi-src","name":"FlowerBI Source","kind":"git","url":"https://github.com/example/flowerbi.git","subpath":"src"},
  {"id":"external-guide","name":"External Guide","kind":"internet","url":"https://example.com/guide.md"}
]
```

Supported source kinds:
- `local` – a folder on the server (e.g., `knowledge-bases/product`).
- `git` – a remote Git repository, optionally with a `subpath` to a subfolder.
- `internet` – a plain HTTP/HTTPS URL that returns Markdown.
- `agent` – the agent’s own knowledge (no URL needed).

### Define Destinations

Use `KNOWLEDGE_DESTINATIONS` as a JSON array. Each destination must have an `id`, `name`, and `url` (Git remote).

```env
KNOWLEDGE_DESTINATIONS=[
  {"id":"flowerbi-docs","name":"FlowerBI Docs","url":"https://github.com/example/flowerbi-docs.git","subpath":"docs"}
]
```

Destinations are always writable Git repositories that Magpie will publish proposals and edits to.

### Define Flows

Use `KNOWLEDGE_FLOWS` to link sources to a destination. A flow includes a `sourceIds` array and a `destinationId`.

```env
KNOWLEDGE_FLOWS=[
  {"id":"flowerbi","name":"FlowerBI Knowledge Base","sourceIds":["flowerbi-src","external-guide"],"destinationId":"flowerbi-docs"}
]
```

For backwards compatibility, `KNOWLEDGE_REPOSITORIES` and `KNOWLEDGE_REPO_PATH` are still accepted when `KNOWLEDGE_SOURCES`/`KNOWLEDGE_DESTINATIONS` are not set.

### Add Embeddings (Optional but Recommended)

To enable hybrid retrieval, configure an embedding provider:

```env
KNOWLEDGE_STORE=postgres
DATABASE_URL=postgres://postgres:postgres@localhost:5432/markdown_magpie
OPENAI_COMPATIBLE_EMBEDDING_MODEL=text-embedding-3-small
OPENAI_COMPATIBLE_BASE_URL=https://api.openai.com/v1
OPENAI_COMPATIBLE_API_KEY=sk-...
```

## The Watcher and AI Job Execution

All AI work in Magpie is modeled as jobs on a pg-boss queue in Postgres, not as a hard dependency on one model vendor. **The API never calls a model inline.** It enqueues a job; a separate **watcher** process claims it, invokes the configured provider, and posts the result back over HTTP. The API and watcher share only the HTTP API and the managed-checkout volume — the watcher has no direct database access.

### Job States

A job moves through these states (mirroring pg-boss):

`created` → `active` → `completed` (terminal). Other states: `retry` (queued for another attempt after a recoverable failure), `failed` (terminal, retries exhausted), `cancelled` (terminal, cancelled by an operator), `blocked` (waiting on a dependency / singleton key).

### Watcher Capabilities

A watcher advertises a **capability** for each provider whose credentials are present in its environment, plus `maintenance` (always available). The API only routes a job to a capability a running watcher actually offers, so a job stays queued until a capable watcher is running.

| Capability | Required env |
| --- | --- |
| `openai-compatible` | `OPENAI_COMPATIBLE_BASE_URL`, `OPENAI_COMPATIBLE_API_KEY`, `OPENAI_COMPATIBLE_MODEL` |
| `azure-openai` | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_CHAT_DEPLOYMENT` |
| `codex` | `CODEX_CLI_PATH` (defaults to `codex` on `PATH`) |
| `claude` | `CLAUDE_CLI_PATH` (defaults to `claude` on `PATH`) |
| `github` | `GITHUB_TOKEN`, `MAGPIE_GIT_AUTHOR_NAME`, `MAGPIE_GIT_AUTHOR_EMAIL` |
| `maintenance` | (none) |

### Client Flow

The standard request/await pattern for any job-backed endpoint:

1. `POST` the work — the API returns **`202`** with the created job and links.
2. `GET /api/jobs/:id/wait` — long-polls. Returns **`200`** once the job is terminal, or **`202`** if it is still running.
3. `GET /api/jobs/:id` — fetch the job snapshot at any time without blocking.

## Indexing a Flow

Once a flow is configured, you must index its destination before users can ask questions. Indexing parses Markdown files, extracts sections, and (optionally) generates embeddings.

### Using the API

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"flowerbi"}'
```

This indexes only the **destination** of the flow. The response includes the number of documents and sections indexed:

```json
{
  "repository": "flowerbi-docs",
  "documentCount": 12,
  "sectionCount": 48,
  "commitSha": "abc123"
}
```

### Using the Web Console

Navigate to **Knowledge > Repositories** in the web console. Click the **Index** button next to your flow to trigger indexing.

### After Indexing

The API automatically runs background embedding for any section whose vector is missing. Embedding runs inside the API process — no separate watcher is needed for this step. When embeddings are configured, retrieval becomes **hybrid** (a pgvector nearest-neighbour search fused with keyword scoring via Reciprocal Rank Fusion). Otherwise, keyword-only scoring is used. The active retrieval mode is reported by `GET /api/config`.

## Viewing Flow Status

### API Endpoints

- `GET /api/knowledge/repositories` – list all indexed repositories (each corresponds to a flow’s destination).
- `GET /api/knowledge/stats` – get total document and section counts.
- `GET /api/knowledge/documents` – list all indexed documents across flows.

### Web Console

The **Knowledge** page shows each flow’s index status, including last indexed commit and counts.

## Managing Documents in the Knowledge Base

The knowledge base is the indexed corpus used to answer questions. Documents are sourced from configured flows and stored in a Git-backed destination repository.

### Adding Documents

New documents are added by **including them in the destination repository** and then re-indexing. Magpie does not provide a separate “upload” endpoint; instead, you:

1. Add your Markdown file(s) to the destination repository (either directly via Git or through a proposal/pull request).
2. Run the index command to make Magpie aware of them.

#### Adding via Git (manual)

Edit the destination repository locally or push a new branch:

```bash
# Navigate to the checkout (e.g., inside MAGPIE_CHECKOUT_ROOT/<destination-id>)
cd /path/to/magpie-checkouts/flowerbi-docs
# Add a new file
echo '# New Topic' > docs/new-topic.md
git add docs/new-topic.md
git commit -m "Add new-topic.md"
git push
```

#### Adding via a Proposal (recommended for gaps)

When the gap detector finds missing knowledge, the recommended workflow is:

1. A gap cluster appears via `GET /api/gaps/clusters`.
2. Generate a proposal from that cluster (either manually or via the scheduled reconciler).
3. The proposal drafts a Markdown document, commits it to a branch, and opens a pull request.
4. After review and merge, the new document is part of the destination.

### Removing Documents

To remove a document, you must **delete the file from the destination repository** and then re-index. There is no dedicated API to delete a single indexed section; the index is built from the repository’s current file set.

1. Delete the Markdown file from the destination checkout:

```bash
cd /path/to/magpie-checkouts/flowerbi-docs
rm docs/unwanted-topic.md
git commit -am "Remove unwanted-topic.md"
git push
```

2. Re-index the flow:

```bash
curl -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"flowerbi"}'
```

After re-indexing, the deleted document no longer appears in search results or answer citations.

#### Removing Multiple Documents

Batch deletions work the same way — remove several files, commit, push, and re-index once.

#### Removing Documents via Patrol Maintenance

If a document has become obsolete or inconsistent, the scheduled **fix-patrol** can detect duplicates, contradictions, and overgrown documents. When it finds a problem, it creates a proposal with a multi-file changeset (dedupe, split, or correct) that can be reviewed and published as a pull request.

## Keeping Flows Updated

### Source Change Sync

The `source-change-sync` background job (scheduled by default every 10 minutes) watches each flow's git sources. When a source has new commits, it rewrites the corresponding destination documents and re-indexes them. This is an **event-driven** job: the trigger is the source commit, and its scope is limited to the diff plus the documents that cite that area. Source-sync changes now flow through the same proposal model as gap drafts, so they can fold into existing open PRs. You can disable this in the Schedules settings page if you prefer manual control.

### Manual Re-index

To force a re-index of a flow (e.g., after changing destination documents by hand), call:

```bash
curl -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"my-flow"}'
```

### Resetting and Full Re-index

For demos or clean slates, the `/api/admin/reset` endpoint clears all indexed data and re-syncs + re-indexes all configured flows:

```bash
curl -X POST http://localhost:4000/api/admin/reset
```

**Warning:** This is destructive and unauthenticated; never expose it in production.

## The Gap Pipeline and Flows

Every question asked via `/api/ask` or the MCP tool `kb.ask` is logged. Low-confidence answers and user-flagged gaps are clustered per flow. The `gaps-to-pull-requests` reconciler then:
1. Groups related gaps into clusters.
2. Drafts Markdown proposals that fill those gaps.
3. Publishes them to a Git branch in the flow's destination repository.
4. Opens a pull request (if a token for the remote host is configured) and advances the proposal as the PR is merged.

Proposals are always relative to the flow's destination. You can review draft proposals via `GET /api/proposals` or the web console's **Proposals** page.

### Proposal Lifecycle

A proposal moves through these statuses: `draft`, `ready`, `branch-pushed`, `pr-opened`, `merged`, `rejected`. Once a proposal is `ready`, it can be published:

```bash
POST /api/proposals/:id/status
{ "status": "ready" }

POST /api/proposals/:id/publish
```

Publication is enqueue-only. The watcher commits the Markdown to a `magpie/proposal-*` branch, pushes it, and opens a pull request. The proposal records the branch, commit SHA, and PR URL. If no host token is available, it degrades gracefully to a pushed branch.

### Gap Clusters

The reconciler maintains persisted gap clusters, each with an `id`, `title`, `questionIds`, `count`, and optional `rationale`. Clusters are surfaced via `GET /api/gaps/clusters` and provide a fast read without model calls. Clustering happens in the background reconciler, not on request.

## Patrol Maintenance: Scheduled Knowledge Base Tidying

Rather than a single whole-knowledge-base crunch, Magpie now runs several **patrol** lenses on a rolling cursor. Each scheduled tick selects the least-recently-checked documents in a flow and runs one or more lenses over them:

- **Fix-patrol** – correctness and structural maintenance:
  - **Verify** – checks document claims against source material; flags unprovable statements.
  - **Correct** – automatically rewrites flagged documents.
  - **Dedupe** – finds near-duplicates and reconciles pairs.
  - **Split** – breaks overgrown documents into focused pieces.
- **Improve-patrol** – editorial growth; source-grounded expansions for fine-but-thin documents.

Each patrol produces a `MaintenanceRun` record surfaced in the Schedules and Activity pages. Proposals created by patrols are clusterless and go through the same reconcile gate (fold into existing open PRs on overlap, publish as own PR otherwise). The fix-patrol and improve-patrol are separate jobs: fix-patrol is conservative (only acts when something is demonstrably wrong), while improve-patrol is proactive (grows fine-but-thin docs).

Patrol schedules can be enabled/disabled per flow and cron expression via the **Schedules** page in the web console.

### Patrol Trigger

The patrol uses a rolling cursor: each tick picks the N least-recently-checked files (or files past a staleness threshold) and runs the lenses on them. This rotates through the whole KB over days with bounded cost per tick, and naturally re-visits files as they age.

## Verifying the Knowledge Base State

After any index operation, query the stats endpoint to confirm changes:

```bash
curl -s http://localhost:4000/api/knowledge/stats
# {"repositoryCount":1,"documentCount":15,"sectionCount":72}
```

List all indexed documents:

```bash
curl -s http://localhost:4000/api/knowledge/documents
```

## Best Practices

- Use separate git repositories for sources and destinations to control access and review cycles.
- Keep the destination repository human‑readable: it is the authoritative knowledge base for your customers.
- If a flow has no sources (e.g., you write documentation directly in the destination repo), omit `sourceIds` and let the gap pipeline propose changes from user questions alone.
- Monitor the `GET /api/gaps/clusters` endpoint to see which missing topics are most frequently asked.
- For production deployments, configure `GITHUB_TOKEN` (or equivalent) so proposals automatically become pull requests.
- Use descriptive, unique headings that summarize section content.
- Keep sections focused on a single topic.
- Avoid extremely long sections; break them into smaller, well-named subsections.

## Troubleshooting

| Symptom | Likely Cause | Remedy |
|---|---|---|
| `/api/ask` returns low confidence | Destination not indexed or embeddings missing | Re-index the flow and check embedding configuration. |
| Proposals not appearing as PRs | No git host token configured | Set `GITHUB_TOKEN` or equivalent and ensure the `refresh_pull_requests` scheduler is running. |
| Patrol plan says “no changes needed” | Destination already well‑structured | Configure smaller interval or manually trigger a full analysis. |
| Web console shows no flows | Environment variables not set | Verify `KNOWLEDGE_FLOWS` in `.env` and restart the API. |
| Index returns “0 documents” | Destination checkout not synced or path wrong | Verify `MAGPIE_CHECKOUT_ROOT` and that the destination repo is cloned. Check API startup logs for sync errors. |
| New document not found in search | Index did not run after adding the file | Run index endpoint again. |
| Re-index takes a long time | Embedding pass for many new sections | Wait; embedding runs in background and is idempotent. |
| “local_path_not_allowed” error | Trying to index an arbitrary path without a configured flow | Use a flow ID defined in `KNOWLEDGE_FLOWS`. |
| Changes not reflected after re-index | Browser caching of search results | Use a cache-busting parameter or wait for TTL; re-query the API. |
| “Failed to sync configured git repositories” | `MAGPIE_CHECKOUT_ROOT` is not writable or missing | Create the directory and ensure write permissions. |
| Hybrid retrieval not active | Embedding credentials incomplete or `KNOWLEDGE_STORE` not set | Check that `KNOWLEDGE_STORE=postgres` and a complete set of embedding credentials are set. |
| `/api/ask` returns 202 (queued) and job never completes | No watcher running, or watcher does not advertise a capable provider | Start the watcher process and confirm its provider credentials match `AI_PROVIDER`. |

## Reference

- [HTTP API Reference](integrations-and-connecting-data-sources.md) — endpoints for managing flows, indexing, and proposals.
- [Ingestion Configuration](integrations-and-connecting-data-sources.md) — detailed configuration of sources, destinations, and flows.
- [Architecture Overview](integrations-and-connecting-data-sources.md) — the product loop and how flows fit in.
