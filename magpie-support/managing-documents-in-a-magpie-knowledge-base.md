---
title: Managing Documents in a Magpie Knowledge Base
status: draft
---

# Managing Documents in a Magpie Knowledge Base

This guide explains how to add, re-index, and remove documents from your Magpie knowledge base (KB). The KB is the indexed corpus used to answer questions via `/api/ask` and MCP tools. Documents are sourced from configured *flows* (source → destination) and stored in a Git-backed repository.

## How the Knowledge Base Works

Magpie maintains a **destination KB** — a set of Markdown documents in a Git repository that serve as the authoritative answer corpus. Raw sources (source repos, internet pages, agent knowledge) are used to populate or update the destination, but only the destination is indexed for retrieval.

Configured flows in `KNOWLEDGE_FLOWS` define which sources feed into which destination. The destination is cloned (or linked) to a local checkout directory at startup. Indexing reads all `.md` files in the destination, parses frontmatter, splits by headings, and stores sections for hybrid keyword/vector search.

## Adding Documents to the Knowledge Base

New documents are added by **including them in the destination repository** and then re-indexing. Magpie does not provide a separate “upload” endpoint; instead, you:

1. Add your Markdown file(s) to the destination repository (either directly via Git or through a proposal/pull request).
2. Run the index command to make Magpie aware of them.

### Adding via Git (manual)

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

### Adding via a Proposal (recommended for gaps)

When the gap detector finds missing knowledge, the recommended workflow is:

1. A gap cluster appears via `GET /api/gaps/clusters`.
2. Generate a proposal from that cluster (either manually or via the scheduled reconciler).
3. The proposal drafts a Markdown document, commits it to a branch, and opens a pull request.
4. After review and merge, the new document is part of the destination.

This is the primary way to add content that closes knowledge gaps. See [AI Jobs](ai-jobs.md) for the proposal lifecycle.

## Re-indexing the Knowledge Base

After adding, editing, or removing documents, you must re-index to update the search index and embeddings. Re-indexing scans the destination repository’s current state and replaces the indexed sections.

### Manual Re-index via API

Use the `/api/knowledge/repositories/index` endpoint, specifying the flow ID:

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"flowerbi"}'
```

If your deployment uses the older `KNOWLEDGE_REPOSITORIES` variable, pass `repositoryId` instead:

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"repositoryId":"cats"}'
```

The API returns a summary with document and section counts, and triggers background embedding for new sections. Wait for the embedding pass to finish (check API logs for “Embedded N section(s); 0 remaining”) before asking questions.

### Scheduled Re-index

Magpie includes a **source-change-sync** scheduled task that watches each flow’s Git sources and rewrites destination documents when a source has changed. It runs every 10 minutes by default and re-indexes automatically after rewriting. This keeps the KB up-to-date without manual re-index.

See [Architecture](architecture.md) under “Primary Flow” for details.

### Resetting and Full Re-index

For demos or clean slates, the `/api/admin/reset` endpoint clears all indexed data and re-syncs + re-indexes all configured flows:

```bash
curl -X POST http://localhost:4000/api/admin/reset
```

**Warning:** This is destructive and unauthenticated; never expose it in production.

## Removing Documents from the Knowledge Base

To remove a document, you must **delete the file from the destination repository** and then re-index. There is no dedicated API to delete a single indexed section; the index is built from the repository’s current file set.

### Steps to Remove a Document

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

### Removing Multiple Documents

Batch deletions work the same way — remove several files, commit, push, and re-index once.

### Removing Documents via Proposals

If a document has become obsolete or inconsistent, you can also create a **Crunch** proposal that includes file deletions. Crunch is a scheduled or on-demand maintenance pass that can consolidate, split, or delete documents. After the Crunch plan is published and merged, the deletions take effect. See [AI Jobs](ai-jobs.md#crunch--scheduled-knowledge-base-tidying).

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

## Troubleshooting

| Symptom | Likely Cause | Remedy |
|---------|--------------|--------|
| Index returns “0 documents” | Destination checkout not synced or path wrong | Verify `MAGPIE_CHECKOUT_ROOT` and that the destination repo is cloned. Check API startup logs for sync errors. |
| New document not found in search | Index did not run after adding the file | Run index endpoint again. |
| Re-index takes a long time | Embedding pass for many new sections | Wait; embedding runs in background and is idempotent. |
| “local_path_not_allowed” error | Trying to index an arbitrary path without a configured flow | Use a flow ID defined in `KNOWLEDGE_FLOWS`. |
| Changes not reflected after re-index | Browser caching of search results | Use a cache-busting parameter or wait for TTL; re-query the API. |

## References

- [Ingestion documentation](ingestion.md) — full details on source/destination configuration and indexing.
- [HTTP API reference](api.md) — `/api/knowledge/repositories/index`, `/api/knowledge/stats`.
- [Architecture overview](architecture.md) — primary flow, scheduled syncing.
- [AI Jobs & Crunch](ai-jobs.md) — proposal lifecycle and knowledge tidying.