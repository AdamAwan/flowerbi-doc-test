---
title: Onboarding and Indexing Your Content into Magpie
status: draft
---

# Onboarding and Indexing Your Content into Magpie

This guide walks you through the process of getting your Markdown content into Markdown Magpie so that the system can answer questions, detect gaps, and propose improvements. It covers configuration, indexing, and verification.

## Prerequisites

Before you start, ensure the Markdown Magpie stack is running:

- The HTTP API (`@magpie/api`) on port `4000`.
- A Postgres database (with `pgvector`) reachable via `DATABASE_URL`.
- (Optional) An embeddings provider if you want hybrid keyword + vector retrieval. See the [Embedding Configuration](#embedding-configuration) section below.

If you haven’t started the stack yet, follow the [Local Development](../README.md#local-development) instructions in the repo’s main README.

## Step 1: Configure Your Knowledge Sources and Destinations

Markdown Magpie separates raw sources from the curated destination knowledge base. You configure these via environment variables in `.env`.

### Sources

A **source** is where raw Markdown originates. Supported kinds are:

- `local` – a folder on the server (e.g., a bundled knowledge base).
- `git` – a remote Git repository.
- `agent` – a knowledge base maintained by an agent (no external URL).
- `internet` – a public URL (Markdown fetched on demand).

Example source configuration:

```env
KNOWLEDGE_SOURCES=[{"id":"cats","name":"Cat Care Repo","kind":"local","path":"knowledge-bases/cats"}]
```

### Destinations

A **destination** is the curated knowledge base that Magpie indexes for answering questions. It is typically a Git repository where reviewed proposals will be submitted as pull requests.

Example destination:

```env
KNOWLEDGE_DESTINATIONS=[{"id":"cats-docs","name":"Cat Care Docs","url":"https://github.com/your-org/cats-docs.git","subpath":"docs"}]
```

### Flows

A **flow** links one or more sources to a destination. This tells Magpie which sources to use for drafting updates to the destination KB.

```env
KNOWLEDGE_FLOWS=[{"id":"cats","name":"Cat Care KB","sourceIds":["cats"],"destinationId":"cats-docs"}]
```

For backward compatibility, `KNOWLEDGE_REPOSITORIES` and `KNOWLEDGE_REPO_PATH` are still supported but the explicit source/destination/flow model is preferred.

Make sure `MAGPIE_CHECKOUT_ROOT` points to a writable directory where cloned repositories will be stored:

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts
```

The API will clone or fast‑forward pull remote sources and destinations into this directory at startup.

## Step 2: Index the Destination Knowledge Base

Once the API is running and your flows are configured, trigger a reindex of a specific flow’s destination. This is the curated KB that the `/ask` endpoint and MCP tools will query.

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"cats"}'
```

The API will:

1. Walk the destination repository for `.md` files (ignoring `.git` and `node_modules`).
2. Parse frontmatter and split documents into heading‑based sections.
3. Store sections in the in‑memory search index and persist them to Postgres.
4. Kick off a background task to embed any sections whose embedding is `NULL` (if an embeddings provider is configured).

The POST request returns a summary:

```json
{
  "repository": { "id": "cats-docs", "path": ".magpie/checkouts/cats-docs/docs", "commitSha": "abc123" },
  "documentCount": 12,
  "sectionCount": 84
}
```

## Step 3: Verify the Index

Check the system’s overall state:

```bash
curl -s http://localhost:4000/api/knowledge/stats
```

```json
{
  "repositoryCount": 1,
  "documentCount": 12,
  "sectionCount": 84
}
```

Search for a topic:

```bash
curl -s 'http://localhost:4000/api/knowledge/search?q=claws&limit=3'
```

If embeddings are configured, the retrieval mode reported by `GET /api/config` under `retrieval.mode` should be `hybrid`. If only keyword search is active, you’ll see `keyword` with a reason.

Finally, ask a question:

```bash
curl -s -X POST http://localhost:4000/api/ask \
  -H 'content-type: application/json' \
  -d '{"question":"How should I introduce a new cat food?"}'
```

You should receive an answer with citations and a confidence rating.

## Embedding Configuration (Optional)

For hybrid retrieval (vector + keyword), configure an embeddings provider **independently** of the chat provider. The following sets work:

**OpenAI‑compatible** (e.g., OpenAI, DeepSeek, local vLLM):

```env
OPENAI_COMPATIBLE_BASE_URL=https://api.openai.com/v1
OPENAI_COMPATIBLE_API_KEY=sk-...
OPENAI_COMPATIBLE_EMBEDDING_MODEL=text-embedding-3-small
```

**Azure OpenAI**:

```env
AZURE_OPENAI_ENDPOINT=https://myinstance.openai.azure.com
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=text-embedding-3-small
```

Hybrid mode activates automatically when `KNOWLEDGE_STORE=postgres` **and** a complete set of embedding credentials is present. The model must output 1536‑dimensional vectors to be compatible with the existing vector index.

## Next Steps

Once content is onboarded, you can:

- Ask questions via the API, web console, or MCP server.
- View logged questions and knowledge gap candidates at `GET /api/questions` and `GET /api/gaps/candidates`.
- Review gap clusters and generate proposals using the AI job workflow.
- Configure scheduled maintenance (Crunch) to tidy up the knowledge base over time.

For more details, see:

- [Ingestion documentation](./ingestion.md) – deeper explanation of source/destination handling.
- [API Reference](./api.md) – all endpoints for knowledge management.
- [AI Job Contract](./ai-jobs.md) – how proposals and gap resolution work.