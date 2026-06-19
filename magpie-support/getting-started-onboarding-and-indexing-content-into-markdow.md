---
title: Getting Started: Onboarding and Indexing Content into Markdown Magpie
owner: magpie-ops
status: draft
tags: [getting-started, onboarding, indexing]
review_cycle_days: 90
---

# Getting Started: Onboarding and Indexing Content into Markdown Magpie

This guide explains how to get your content into Markdown Magpie so it can answer questions with citations, detect knowledge gaps, and propose improvements. You will:

- Set up a local development environment.
- Configure knowledge sources and destinations.
- Index your Markdown content.
- Verify that indexing succeeded.

## Prerequisites

- Node.js 22+ and npm 10 (if npm 11 fails, use `npx --yes npm@10 ci`).
- Docker and Docker Compose (for Postgres and Redis).
- A Git repository with Markdown files you want to manage.

## 1. Clone and Install

```bash
git clone https://github.com/AdamAwan/markdown-magpie.git
cd markdown-magpie
npm install
```

If you encounter `Exit handler never called!`, use:
```bash
npx --yes npm@10 ci
```

## 2. Configure Environment

Copy the example environment file:

```bash
cp .env.example .env
```

Edit `.env` to set at minimum:

```env
DATABASE_URL=postgres://postgres:postgres@localhost:5432/markdown_magpie
STORAGE_BACKEND=postgres
AI_EXECUTION_MODE=direct
AI_PROVIDER=mock
```

> **Note:** `AI_PROVIDER=mock` uses a deterministic answer generator – no API key needed. For real AI features, see [Chat Providers](../chat-providers.md).

## 3. Start Dependencies (Postgres + Redis)

The Docker Compose file is designed so that a bare `docker compose up` starts only the backing services (Postgres and Redis) without the application containers:

```bash
docker compose up -d
```

Wait for Postgres to be healthy:
```bash
until [ "$(docker inspect -f '{{.State.Health.Status}}' "$(docker compose ps -q postgres)")" = healthy ]; do sleep 2; done
```

## 4. Run Migrations

```bash
npm run db:migrate
```

## 5. Configure Knowledge Flows

Markdown Magpie uses **knowledge flows** to describe which Git repositories act as raw sources and which destination repository holds the curated knowledge base.

Edit `.env` to define at least one flow, for example using the bundled cats knowledge base:

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts
KNOWLEDGE_SOURCES=[{"id":"cats-src","name":"Cats Source","kind":"local","path":"knowledge-bases/cats"}]
KNOWLEDGE_DESTINATIONS=[{"id":"cats-docs","name":"Cats Docs","kind":"local","path":"knowledge-bases/cats"}]
KNOWLEDGE_FLOWS=[{"id":"cats","name":"Cats KB","sourceIds":["cats-src"],"destinationId":"cats-docs"}]
```

> **Explanation:** The source (`cats-src`) is the raw document folder; the destination (`cats-docs`) is the curated KB that will be indexed for question answering. For a simple setup, both can point to the same folder. See [ingestion.md](../ingestion.md) for full configuration options.

## 6. Start the API

Override `MAGPIE_CHECKOUT_ROOT` to a writable local path so the API can clone repositories on startup:

```bash
mkdir -p .magpie/checkouts
MAGPIE_CHECKOUT_ROOT="$PWD/.magpie/checkouts" npm run dev:api &
```

The API will be available at `http://localhost:4000`. Verify with:
```bash
curl localhost:4000/api/health
```

## 7. Index Your Content

With the API running, trigger indexing of a configured flow:

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"cats"}'
```

If using the bundled cats data, you should see output like:
```json
{
  "repository": "cats",
  "documentCount": 8,
  "sectionCount": 32,
  "commitSha": "..."
}
```

> **Note:** The API indexes the **destination** of a flow, not the raw source. The `flowId` must match an entry in `KNOWLEDGE_FLOWS`.

## 8. Verify Indexing

Check that your documents are indexed:

```bash
curl -s http://localhost:4000/api/knowledge/stats
```

Search for a term:

```bash
curl -s 'http://localhost:4000/api/knowledge/search?q=claws'
```

Ask a test question:

```bash
curl -s http://localhost:4000/api/ask \
  -H 'content-type: application/json' \
  -d '{"question":"How should I introduce a new cat food?"}'
```

## 9. (Optional) Start the Web Console

In a separate terminal, start the Next.js web app:

```bash
MAGPIE_DEV_API_PROXY="http://localhost:4000" npm run dev:web
```

Open `http://localhost:3000` to browse the knowledge base, view questions, and manage proposals.

## Important Details

- **Indexing is idempotent.** Running it multiple times updates rather than duplicates.
- **Embeddings are computed in the background** if an embedding provider is configured. See [Embedding Configuration](../ingestion.md#embedding-configuration).
- **Retrieval mode** is `keyword` by default. To enable hybrid (keyword + vector) search, configure Postgres with pgvector and an embedding provider.
- **Destinations are local by default** but can point to remote Git repositories. The API clones them into `MAGPIE_CHECKOUT_ROOT` on startup.

## Troubleshooting

| Problem | Likely cause | Solution |
|---------|--------------|----------|
| `curl localhost:4000/api/health` fails | API not started or port conflict | Check the terminal running the API; kill other processes on port 4000 |
| Indexing returns `400 configured_repository_required` | Multiple flows configured, none specified | Provide a valid `flowId` |
| `/ask` returns low confidence or `no source material` | No indexed content or embedding incomplete | Verify indexing; wait for background embedding to finish |
| `MAGPIE_CHECKOUT_ROOT` not writable | Override not set | Use `MAGPIE_CHECKOUT_ROOT="$PWD/.magpie/checkouts"` before starting the API |

## Next Steps

- Learn about [knowledge gap detection and proposals](../question-logging.md).
- Set up [real AI providers](../chat-providers.md) instead of `mock`.
- Configure automated [Crunch](../ai-jobs.md#crunch) for knowledge base tidying.

---

**References**
- [Repository README](../../README.md) – full local development instructions.
- [Markdown Ingestion](../ingestion.md) – detailed source/destination/flow configuration.
- [HTTP API Reference](../api.md) – all API endpoints for indexing and searching.
- [Architecture](../architecture.md) – high-level system design and provider strategy.
