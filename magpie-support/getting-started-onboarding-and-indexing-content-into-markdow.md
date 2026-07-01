---
title: Getting Started: Onboarding and Indexing Content into Markdown Magpie
owner: magpie-ops
status: draft
tags: [getting-started, onboarding, indexing, quickstart]
review_cycle_days: 90
---

> **Note:** This guide consolidates content from the [Quick Start](quick-start.md) document, which has been deprecated. For the most current instructions, refer to this guide. For a quick reference, see the [Quick Start (Legacy Reference)](quick-start.md).

> **Architecture note:** Markdown Magpie uses a queue-only architecture for all generative (chat) AI work: the API never calls a chat model inline. It enqueues a job on a pg-boss queue in Postgres; a separate **watcher** process claims it, invokes the configured provider, and posts the result back over HTTP. Embeddings are the one exception — the API computes them inline (it holds an embedding provider) for indexing and query-time retrieval. The watcher is **required** for any generative work (answers, proposals, drafting, maintenance). Without a running watcher, `POST /api/ask` returns a 202 job that never completes.

# Getting Started: Onboarding and Indexing Content into Markdown Magpie

This guide explains how to get your Markdown content into Markdown Magpie so it can answer questions with citations, detect knowledge gaps, and propose improvements. You will:

- Set up a local development environment.
- Configure knowledge sources and destinations.
- Index your Markdown content.
- Verify that indexing succeeded.

## Prerequisites

- Node.js 22+ and npm 10 (if npm 11 fails, use `npx --yes npm@10 ci`).
- Docker and Docker Compose (for Postgres and optional Redis).
- A Git repository with Markdown files you want to manage.
- The HTTP API (`@magpie/api`) on port 4000.
- A Postgres database (with `pgvector`) reachable via `DATABASE_URL`.
- (Optional) An embeddings provider if you want hybrid keyword + vector retrieval. See [Embedding Configuration](#embedding-configuration) below.
- **Watcher (required for all generative work):** Because the API never calls a chat model inline, you must run the watcher process (see [Step 7](#7-start-the-watcher)). Without it, questions will be enqueued but never answered.

> **Note:** Redis is **not required** for local development. The queue uses Postgres via pg-boss. The `QUEUE_URL` variable in `.env.example` is legacy and can be left blank. Docker Compose starts Postgres only by default; if you need Redis, add it to your compose file.

If you haven’t started the stack yet, follow the [Local Development](../README.md#local-development) instructions in the repo’s main README.

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
AI_PROVIDER=openai-compatible
```

> **Note:** `AI_PROVIDER=openai-compatible` requires `OPENAI_COMPATIBLE_BASE_URL`, `OPENAI_COMPATIBLE_API_KEY`, and `OPENAI_COMPATIBLE_MODEL` to be set. For other provider options, see [Chat Providers](integrations-and-connecting-data-sources.md#ai-provider-integrations). The `mock` provider has been removed.

The watcher will need the environment variables matching the chosen `AI_PROVIDER`. Below are the required variables for each provider:

| Provider | Required Watcher Environment Variables |
|----------|----------------------------------------|
| `openai-compatible` | `OPENAI_COMPATIBLE_BASE_URL`, `OPENAI_COMPATIBLE_API_KEY`, `OPENAI_COMPATIBLE_MODEL` |
| `azure-openai` | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_CHAT_DEPLOYMENT` |
| `codex` | `CODEX_CLI_PATH` (defaults to `codex` on `PATH`) |
| `claude` | `CLAUDE_CLI_PATH` (defaults to `claude` on `PATH`) |

Without the correct credentials, the watcher will not advertise the capability and jobs will remain queued.

## 3. Start Dependencies (Postgres + optional Redis)

The Docker Compose file is designed so that a bare `docker compose up` starts only the backing services (Postgres and Redis) without the application containers. Redis is not required for core functionality—the queue uses Postgres via pg-boss. If you prefer not to run Redis, you can comment out the Redis service in `docker-compose.yml` or simply ignore it.

```bash
docker compose up -d
```

Wait for Postgres to be healthy:

```bash
until [ "$(docker inspect -f '{{.State.Health.Status}}' "$(docker compose ps -q postgres)")" = healthy ]; do sleep 2; done
```

> **Note:** By default, the queue uses Postgres via pg‑boss, so Redis is optional.

## 4. Run Migrations

```bash
npm run db:migrate
```

## 5. Configure Knowledge Flows

Markdown Magpie uses **knowledge flows** to describe which Git repositories act as raw sources and which destination repository holds the curated knowledge base. Flows link **sources** (where raw Markdown originates) to a **destination** (the curated KB that gets indexed for answers).

### Sources

A **source** is where raw Markdown originates. Supported kinds are:

- `local` – a folder on the server (e.g., a bundled knowledge base).
- `git` – a remote Git repository.
- `agent` – a knowledge base maintained by an agent (no external URL).
- `internet` – a public URL (Markdown fetched on demand).

Example source configuration:

```env
KNOWLEDGE_SOURCES=[{"id":"product","name":"Product Docs","kind":"local","path":"knowledge-bases/product"}]
```

### Destinations

A **destination** is the curated knowledge base that Magpie indexes for answering questions. It is typically a Git repository where reviewed proposals will be submitted as pull requests.

Example destination:

```env
KNOWLEDGE_DESTINATIONS=[{"id":"product-docs","name":"Product Docs","url":"https://github.com/your-org/product-docs.git","subpath":"docs"}]
```

### Flows

A **flow** links one or more sources to a destination. This tells Magpie which sources to use for drafting updates to the destination KB.

```env
KNOWLEDGE_FLOWS=[{"id":"product","name":"Product KB","sourceIds":["product"],"destinationId":"product-docs"}]
```

For backward compatibility, `KNOWLEDGE_REPOSITORIES` and `KNOWLEDGE_REPO_PATH` are still supported but the explicit source/destination/flow model is preferred.

> **Example with a local source (for testing):** If you prefer a quick local setup without a remote Git repository, you can use the following configuration:
>
> ```env
> MAGPIE_CHECKOUT_ROOT=.magpie/checkouts
> KNOWLEDGE_SOURCES=[{"id":"cats","name":"Cat Care Repo","kind":"local","path":"knowledge-bases/cats"}]
> KNOWLEDGE_DESTINATIONS=[{"id":"cats-docs","name":"Cat Care Docs","url":"https://github.com/your-org/cats-docs.git","subpath":"docs"}]
> KNOWLEDGE_FLOWS=[{"id":"cats","name":"Cat Care KB","sourceIds":["cats"],"destinationId":"cats-docs"}]
> ```
>
> Then create the `knowledge-bases/cats` folder and add some Markdown files. This is useful for trying out the system without needing a remote Git repository.

Make sure `MAGPIE_CHECKOUT_ROOT` points to a writable directory where cloned repositories will be stored:

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts
```

The API will clone or fast‑forward pull remote sources and destinations into this directory at startup.

## 6. Start the API

Start the API (and database, if not already running). The API will clone or fast‑forward the configured sources and destinations during startup.

```bash
mkdir -p .magpie/checkouts
MAGPIE_CHECKOUT_ROOT="$PWD/.magpie/checkouts" npm run dev:api &
```

The API will be available at `http://localhost:4000`. Verify with:

```bash
curl localhost:4000/api/health
```

## 7. Start the Watcher

The watcher must be running for any generative AI work (answering questions, drafting proposals, publishing, and maintenance jobs). Start it in a separate terminal:

```bash
AUTH_REQUIRED=false WATCHER_API_CLIENT_ID= WATCHER_API_CLIENT_SECRET= \
  MAGPIE_CHECKOUT_ROOT="$PWD/.magpie/checkouts" npm run dev:watcher
```

> Without the watcher, `POST /api/ask` will return `202` and the question will never be answered. The watcher advertises capabilities based on the provider credentials in its environment; ensure the chosen provider's environment variables are set.

## 8. Index Your Content

With the API running, trigger indexing of a configured flow:

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"product"}'
```

For the cats example, use:
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

After indexing, background embedding runs automatically. For the best search and answer quality, wait until the API logs `Embedded N section(s); 0 remaining` before asking questions. With a configured provider, answers are still returned even without embeddings, but confidence scores may be lower.

The POST request returns a summary:

```json
{
  "repository": "product",
  "documentCount": 8,
  "sectionCount": 32,
  "commitSha": "..."
}
```

> **Note:** The API indexes the **destination** of a flow, not the raw source. The `flowId` must match an entry in `KNOWLEDGE_FLOWS`.

## 9. Ask a Question

The API is **enqueue-only** for questions. It records the question, returns `202` with a job ID, and enqueues an `answer_question` job for the watcher. To get the answer, poll the job endpoint:

```bash
curl -s http://localhost:4000/api/ask \
  -H 'content-type: application/json' \
  -d '{"question":"What topics does my knowledge base cover?"}'
# Response includes job.id and questionId

JOB_ID="..."  # from the response's job.id
curl -s "http://localhost:4000/api/jobs/$JOB_ID/wait"
curl -s "http://localhost:4000/api/questions/<question-id>"
```

The `wait` endpoint long-polls: returns **`200`** when the job is complete, **`202`** if still running (re-issue the call). The question ID is returned as `questionId` in the 202 response.

## 10. Verify Indexing

Check that your documents are indexed:

```bash
curl -s http://localhost:4000/api/knowledge/stats
```

```json
{
  "repositoryCount": 1,
  "documentCount": 8,
  "sectionCount": 32
}
```

Search for a term:

```bash
curl -s 'http://localhost:4000/api/knowledge/search?q=setup'
```

Ask a test question (see Step 9).

## 11. (Optional) Start the Web Console

In a separate terminal, start the Next.js web app:

```bash
MAGPIE_DEV_API_PROXY="http://localhost:4000" npm run dev:web
```

Open `http://localhost:3000` to browse the knowledge base, view questions, and manage proposals.

## Switching AI Provider at Runtime

After startup, you can change the active AI provider without restarting the API by calling `POST /api/config`. This is useful for testing different providers quickly. The request accepts either a flat or nested shape:

```bash
curl -s -X POST http://localhost:4000/api/config \
  -H 'content-type: application/json' \
  -d '{"aiProvider":"azure-openai"}'
```

```bash
curl -s -X POST http://localhost:4000/api/config \
  -H 'content-type: application/json' \
  -d '{"ai":{"provider":"azure-openai"}}'
```

The API returns the updated configuration (same shape as `GET /api/config`). The provider must be one of: `openai-compatible`, `azure-openai`, `codex`, `claude`. It will fail with `400` if the provider is not configured (i.e., its environment variables are missing).

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

## Important Details

- **Indexing is idempotent.** Running it multiple times updates rather than duplicates.
- **Embeddings are computed in the background** if an embedding provider is configured.
- **Retrieval mode** is `keyword` by default. To enable hybrid (keyword + vector) search, configure Postgres with pgvector and an embedding provider.
- **No bundled knowledge base is provided.** The `knowledge-bases/` directory is intentionally empty; configure your own sources and destinations.
- **Redis is not required** – the queue uses Postgres via pg-boss, so you can safely omit or disable the Redis container.
- **Watcher is required** for all generative work; without it, jobs are queued but never completed.

## Troubleshooting

| Problem | Likely cause | Solution |
|---------|--------------|----------|
| `curl localhost:4000/api/health` fails | API not started or port conflict | Check the terminal running the API; kill other processes on port 4000 |
| Indexing returns `400 configured_repository_required` | Multiple flows configured, none specified | Provide a valid `flowId` defined in `KNOWLEDGE_FLOWS` |
| `/ask` returns low confidence or `no source material` | No indexed content or embedding incomplete | Verify indexing; wait for background embedding to finish |
| `/ask` returns `202` but never completes | The watcher is not running. | Start the watcher (step 7) and retry the question. |
| `/ask` returns low confidence even after indexing | Embedding pass not finished; retrieval mode is keyword | Wait for background embedding or configure hybrid retrieval |
| Watcher logs `Capability … not ready` | Environment variables for the chosen provider are incorrect | Check provider env vars, see the table in step 2 |
| `401` on API calls | Authentication enabled but no credentials | Set `AUTH_REQUIRED=false` in `.env` or as environment variable |
| `MAGPIE_CHECKOUT_ROOT` not writable | Override not set | Use `MAGPIE_CHECKOUT_ROOT="$PWD/.magpie/checkouts"` before starting the API |
| “local_path_not_allowed” error | Trying to index an arbitrary path without a configured flow | Use a flow ID defined in `KNOWLEDGE_FLOWS` |
| Changes not reflected after re-index | Browser caching of search results | Use a cache-busting parameter or wait for TTL; re-query the API |
| Bootstrap fails with permission error | Override `MAGPIE_CHECKOUT_ROOT` to a writable local path | Use the command from step 6 |
| “Failed to sync configured git repositories” | `MAGPIE_CHECKOUT_ROOT` is not writable or missing | Create the directory and ensure write permissions |
| Hybrid retrieval not active | Embedding credentials incomplete or `KNOWLEDGE_STORE` not set | Check that `KNOWLEDGE_STORE=postgres` and a complete set of embedding credentials are set |
| `/api/ask` returns 202 (queued) | This is normal; the answer is being processed | Poll `GET /api/jobs/<id>/wait` until the job completes |
| Index returns “0 documents” | Destination checkout not synced or path wrong | Verify `MAGPIE_CHECKOUT_ROOT` and that the destination repo is cloned. Check API startup logs for sync errors. |
| New document not found in search | Index did not run after adding the file | Run index endpoint again. |
| Re-index takes a long time | Embedding pass for many new sections | Wait; embedding runs in background and is idempotent. |

## Next Steps

- Learn about [knowledge gap detection and proposals](managing-knowledge-flows-in-markdown-magpie.md#the-gap-pipeline-and-flows).
- Set up [real AI providers](integrations-and-connecting-data-sources.md#ai-provider-integrations) instead of `mock`.
- Configure [hybrid retrieval with embeddings](configuration-reference.md#embedding-provider-configuration) for improved answer quality.
- Configure automated [patrol maintenance](managing-knowledge-flows-in-markdown-magpie.md#patrol-maintenance-scheduled-knowledge-base-tidying) for knowledge base tidying.
- Review the [permissions and access controls](permissions-and-access-controls-in-markdown-magpie.md).
- Review the [Configuration Reference](configuration-reference.md) for comprehensive documentation of all environment variables.
- Understand the [architecture and job model](architecture.md).
- Try the [full Docker Compose stack](README.md#docker-compose) for a self-contained demo.

---

**References**
- [Repository README](../../README.md) – full local development instructions.
- [Markdown Ingestion](managing-knowledge-flows-in-markdown-magpie.md) – detailed source/destination/flow configuration.
- [HTTP API Reference](managing-knowledge-flows-in-markdown-magpie.md#api-endpoints) – all API endpoints for indexing and searching.
- [Architecture](managing-knowledge-flows-in-markdown-magpie.md) – high-level system design and provider strategy.
- [Configuration Reference](configuration-reference.md) – comprehensive documentation of all environment variables.
