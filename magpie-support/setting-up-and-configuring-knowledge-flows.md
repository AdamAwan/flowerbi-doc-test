---
title: Setting Up and Configuring Knowledge Flows
status: draft
---

# Setting Up and Configuring Knowledge Flows

As a new Magpie customer, you need to connect your Markdown documentation sources to a curated knowledge base that Magpie will answer questions from. This is done through **knowledge flows**, which define which raw sources feed into which destination repository, how to index the destination, and how to keep everything in sync.

This guide walks you through the entire setup process, from environment configuration to indexing your first flow.

---

## Prerequisites

- A running Magpie instance (API, database, and optionally the web UI). See the [local development quickstart](../README.md) or production Docker Compose setup.
- One or more Git repositories containing Markdown files. These can be your source docs, your curated knowledge base, or both.
- Environment variables set in your `.env` file (or equivalent).

---

## Step 1: Understand Sources, Destinations, and Flows

Magpie separates **sources** (where raw documentation comes from) from **destinations** (the curated knowledge base that gets indexed for answers). A **flow** links one or more sources to a destination.

| Concept | Description |
|---------|-------------|
| **Source** | A read‑only location of raw Markdown (e.g., your project’s code repository, an internet URL, or an agent knowledge folder). |
| **Destination** | The repository where reviewed, final Markdown lives. This is what Magpie indexes for retrieval and answering. |
| **Flow** | A named connection between sources and a destination. Magpie monitors sources for changes and updates the destination accordingly. |

Sources can be:
- `local` – a folder on the server (e.g., `knowledge-bases/cats`).
- `git` – a remote Git repository (cloned to `MAGPIE_CHECKOUT_ROOT`).
- `internet` – a URL pointing to online documentation.
- `agent` – knowledge harvested from an AI agent (no physical files).

Destinations are always Git repositories (local or remote).

---

## Step 2: Configure Environment Variables

Edit your `.env` file or your container environment to define the checkout root, sources, destinations, and flows.

### Required: Checkout Root

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts
```

This is the local directory where remote repositories are cloned. Create it if it doesn’t exist.

### Define Sources

Use `KNOWLEDGE_SOURCES` as a JSON array. Each source has at minimum an `id` and `name`. The `kind` determines how it is accessed.

```env
KNOWLEDGE_SOURCES=[
  {"id":"flowerbi-src","name":"FlowerBI Source","kind":"git","url":"https://github.com/danielearwicker/flowerbi.git","subpath":"src"},
  {"id":"agent-kb","name":"Agent Knowledge","kind":"agent"}
]
```

### Define Destinations

Use `KNOWLEDGE_DESTINATIONS` as a JSON array. Each destination must have an `id`, `name`, and `url` (Git remote).

```env
KNOWLEDGE_DESTINATIONS=[
  {"id":"flowerbi-docs","name":"FlowerBI Docs","url":"https://github.com/AdamAwan/flowerbi-doc-test.git","subpath":"docs"}
]
```

### Define Flows

Use `KNOWLEDGE_FLOWS` to link sources to a destination. A flow includes a `sourceIds` array and a `destinationId`.

```env
KNOWLEDGE_FLOWS=[
  {"id":"flowerbi","name":"FlowerBI KB","sourceIds":["flowerbi-src","agent-kb"],"destinationId":"flowerbi-docs"}
]
```

> **Note:** For simpler single‑repository setups, you can use the legacy variables `KNOWLEDGE_REPOSITORIES` and `KNOWLEDGE_REPO_PATH`. See the [ingestion documentation](../docs/ingestion.md) for details.

### Add Embeddings (Optional but Recommended for Hybrid Retrieval)

To enable vector search, configure an embeddings provider. For OpenAI‑compatible:

```env
KNOWLEDGE_STORE=postgres
DATABASE_URL=postgres://postgres:postgres@localhost:5432/markdown_magpie
OPENAI_COMPATIBLE_EMBEDDING_MODEL=text-embedding-3-small
OPENAI_COMPATIBLE_BASE_URL=https://api.openai.com/v1
OPENAI_COMPATIBLE_API_KEY=sk-...
```

---

## Step 3: Start Magpie and Verify Configuration

Start the API (and database, if not already running). The API will clone or fast‑forward the configured sources and destinations during startup.

```bash
docker compose up -d postgres
node --env-file=.env scripts/migrate.mjs
MAGPIE_CHECKOUT_ROOT="$PWD/.magpie/checkouts" npm run dev:api
```

Check the health and config endpoints:

```bash
curl http://localhost:4000/api/health
curl http://localhost:4000/api/config
```

The `/api/config` response shows the resolved sources, destinations, flows, and retrieval mode.

---

## Step 4: Index Your Destination Knowledge Base

Indexing reads the destination repository, parses Markdown files, splits them into sections, and populates the search index. It also triggers background embedding of sections if an embeddings provider is configured.

Use the `/api/knowledge/repositories/index` endpoint with your flow ID:

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"flowerbi"}'
```

You should receive a response like:

```json
{
  "repository": "flowerbi-docs",
  "documentCount": 12,
  "sectionCount": 48,
  "commitSha": "abc123"
}
```

> **Tip:** If you have multiple flows, repeat this step for each flow ID.

---

## Step 5: Test Your Flow

### Search sections

```bash
curl -s 'http://localhost:4000/api/knowledge/search?q=FlowerBI&limit=3'
```

### Ask a question

```bash
curl -s http://localhost:4000/api/ask \
  -H 'content-type: application/json' \
  -d '{"question":"What is FlowerBI?"}'
```

If embeddings are configured, retrieval mode will be `hybrid` (keyword + vector). Otherwise it falls back to `keyword`.

---

## Step 6: Automate Syncing (Optional)

Magpie can automatically watch source repositories for changes and update the destination. This is controlled by the `source-change-sync` background task, configured in the web UI or via the API.

- Set `KNOWLEDGE_FLOWS` as above.
- Ensure your Git credentials are available (e.g., `GITHUB_TOKEN`).
- The sync happens every 10 minutes by default; adjust the interval in the Crunch settings page.

---

## Administration and Maintenance

### Viewing Indexed Knowledge

```bash
curl http://localhost:4000/api/knowledge/stats
curl http://localhost:4000/api/knowledge/documents
```

### Re‑indexing After Changes

If you update the destination repository directly (not through a flow sync), re‑index it:

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"flowerbi"}'
```

### Resetting for a Fresh Start (Demo Only)

```bash
curl -X POST http://localhost:4000/api/admin/reset
```

> **Warning:** This clears all data. Use only in development or demo environments.

---

## Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| “Failed to sync configured git repositories” | `MAGPIE_CHECKOUT_ROOT` is not writable or missing. Create the directory and ensure write permissions. |
| “No source material found” answers | The destination has not been indexed yet. Run the index endpoint. |
| Hybrid retrieval not active | Check that `KNOWLEDGE_STORE=postgres` and a complete set of embedding credentials are set. Run `/api/config` to see the resolved `retrieval.mode`. |
| `/api/ask` returns 202 (queued) | `AI_EXECUTION_MODE=queue` is set. Switch to `direct` or start a watcher process. |

---

## Next Steps

- Learn about [asking questions and gap detection](../docs/question-logging.md).
- Explore [proposal generation](../docs/ai-jobs.md) for automatically filling knowledge gaps.
- Set up the [MCP server](../docs/mcp.md) for agent‑driven queries.

---

*This guide was based on the Magpie codebase documentation: [README.md](https://github.com/AdamAwan/markdown-magpie#readme), [docs/ingestion.md](https://github.com/AdamAwan/markdown-magpie/blob/main/docs/ingestion.md), [docs/architecture.md](https://github.com/AdamAwan/markdown-magpie/blob/main/docs/architecture.md), and [.claude/skills/run-magpie/SKILL.md](https://github.com/AdamAwan/markdown-magpie/blob/main/.claude/skills/run-magpie/SKILL.md).*