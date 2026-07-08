---
title: Getting Started: Onboarding and Indexing Content into Markdown Magpie
owner: magpie-ops
status: draft
tags: [getting-started, onboarding, indexing, quickstart]
review_cycle_days: 90
---

> **Note:** This guide consolidates content from the [Quick Start](quick-start.md) and the [Managing Knowledge Flows](managing-knowledge-flows-in-markdown-magpie.md) documents, which have been deprecated. For the most current instructions, refer to this guide. For a quick reference, see the [Quick Start (Legacy Reference)](quick-start.md) and the [Knowledge Flows Reference](managing-knowledge-flows-in-markdown-magpie.md).

> **Architecture note:** Markdown Magpie uses a queue-only architecture for all generative (chat) AI work: the API never calls a chat model inline. It enqueues a job on a pg-boss queue in Postgres; a separate **watcher** process claims it, invokes the configured provider, and posts the result back over HTTP. Embeddings are the one exception — the API computes them inline (it holds an embedding provider) for indexing and query-time retrieval. The watcher is **required** for any generative work (answers, proposals, drafting, maintenance). Without a running watcher, `POST /api/ask` returns a 202 job that never completes.

# Getting Started: Onboarding and Indexing Content into Markdown Magpie

This guide explains how to get your Markdown content into Markdown Magpie so it can answer questions with citations, detect knowledge gaps, and propose improvements. You will:

- Set up a local development environment.
- Configure knowledge sources and destinations.
- Index your Markdown content.
- Verify that indexing succeeded.
- Learn how to manage and maintain your knowledge base over time.

## Prerequisites

- Node.js 22+ and npm 10 (if npm 11 fails, use `npx --yes npm@10 ci`).
- Docker and Docker Compose (for Postgres and optional Redis).
- A Git repository with Markdown files you want to manage.
- The HTTP API (`@magpie/api`) on port 4000.
- A Postgres database (with `pgvector`) reachable via `DATABASE_URL`.
- (Optional) An embeddings provider if you want hybrid keyword + vector retrieval. See [Embedding Configuration](#embedding-configuration) below.
- **Watcher (required for all generative work):** Because the API never calls a chat model inline, you must run the watcher process (see [Step 7](#7-start-the-watcher)). Without it, questions will be enqueued but never answered.

> **Authentication note:** Authentication **fails closed** — it is required unless you explicitly set `AUTH_REQUIRED=false`. When auth is enabled (the default), the API refuses to start if Auth0 is not fully configured (a missing or placeholder `AUTH0_AUDIENCE` aborts startup, with an error such as "Auth0 not configured"). For local development, override `AUTH_REQUIRED=false` in your environment or `.env`. See the [Troubleshooting](#troubleshooting) section for related issues.

> **Note:** Redis is **not required** for local development. The queue uses Postgres via pg-boss. The `QUEUE_URL` variable in `.env.example` is legacy and can be left blank. Docker Compose starts Postgres only by default; if you need Redis, add it to your compose file.

If you haven't started the stack yet, follow the [Local Development](../README.md#local-development) instructions in the repo's main README.

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

> **Note:** `AI_PROVIDER=openai-compatible` requires `OPENAI_COMPATIBLE_BASE_URL`, `OPENAI_COMPATIBLE_API_KEY`, and `OPENAI_COMPATIBLE_MODEL` to be set. For other provider options, see [Chat Providers](#chat-providers). The `mock` provider has been removed.

The watcher will need the environment variables matching the chosen `AI_PROVIDER`. Below are the required variables for each provider:

| Provider | Required Watcher Environment Variables |
|----------|----------------------------------------|
| `openai-compatible` | `OPENAI_COMPATIBLE_BASE_URL`, `OPENAI_COMPATIBLE_API_KEY`, `OPENAI_COMPATIBLE_MODEL` |
| `azure-openai` | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_CHAT_DEPLOYMENT` |
| `codex` | `CODEX_CLI_PATH` (defaults to `codex` on `PATH`) |
| `claude` | `CLAUDE_CLI_PATH` (defaults to `claude` on `PATH`) |

Without the correct credentials, the watcher will not advertise the capability and jobs will remain queued.

## 3. Start Dependencies (Postgres + optional Redis)

The Docker Compose file is designed so that a bare `docker compose up` starts only Postgres (the backing service) without the application containers. Redis is not required for core functionality—the queue uses Postgres via pg-boss. If you prefer not to run Redis, you can comment out the Redis service in `docker-compose.yml` or simply ignore it.

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

### Understanding Flows

A knowledge flow is a named pipeline that links one or more sources to a destination. The flow is used by the system to clone or sync source repositories, index the destination for answering questions, and propose updates when gaps are detected.

| Concept | Description |
|---------|-------------|
| **Source** | A read‑only location of raw Markdown (e.g., your project's code repository, an internet URL, or an agent knowledge folder). |
| **Destination** | The repository where reviewed, final Markdown lives. This is what Magpie indexes for retrieval and answering. |
| **Flow** | A named connection between sources and a destination. Magpie monitors sources for changes and updates the destination accordingly. |

### Sources

A **source** is where raw Markdown originates. Supported kinds are:

| Kind | Configuration | Example |
|------|---------------|---------|
| `local` | `{ "path": "knowledge-bases/cats" }` | A folder inside the repository. |
| `git` | `{ "url": "https://...", "subpath": "docs" }` | Clone a remote repository, optionally using a subfolder. |
| `internet` | `{ "kind": "internet", "url": "https://example.com/docs" }` | Fetch Markdown from a public URL. |
| `agent` | `{ "kind": "agent" }` | Content provided by an AI agent (e.g., from Claude or Codex). |

Sources with kind `agent` or `internet` are used for drafting proposals but are **not indexed** into the answer corpus. Only destinations (curated knowledge bases) are indexed for answering questions.

Example source configuration:

```env
KNOWLEDGE_SOURCES=[{"id":"product","name":"Product Docs","kind":"local","path":"knowledge-bases/product"}]
```

For a single source, `KNOWLEDGE_SOURCES` can be replaced by the alias `SOURCE` (same structure). For backwards compatibility, the singular aliases `KNOWLEDGE_SOURCE` and `KNOWLEDGE_DESTINATION` are also accepted when only one source or destination is needed.

### Destinations

A **destination** is the curated knowledge base that Magpie indexes for answering questions. It is typically a Git repository where reviewed proposals will be submitted as pull requests.

Example destination:

```env
KNOWLEDGE_DESTINATIONS=[{"id":"product-docs","name":"Product Docs","url":"https://github.com/your-org/product-docs.git","subpath":"docs"}]
```

Destinations are always writable Git repositories that Magpie will publish proposals and edits to.

For a single destination, `KNOWLEDGE_DESTINATIONS` can be replaced by the alias `DESTINATION` (same structure).

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

> **Troubleshooting startup:** If the API immediately exits after printing "syncing configured git checkouts", a private remote source may need credentials (non-interactive clone fails) or `MAGPIE_CHECKOUT_ROOT` may be non-writable. Ensure `git clone <source-url>` succeeds standalone, and verify the checkout directory is writable.

## 7. Start the Watcher

The watcher must be running for any generative AI work (answering questions, drafting proposals, publishing, and maintenance jobs). Start it in a separate terminal:

```bash
AUTH_REQUIRED=false WATCHER_API_CLIENT_ID= WATCHER_API_CLIENT_SECRET= \
  MAGPIE_CHECKOUT_ROOT="$PWD/.magpie/checkouts" npm run dev:watcher
```

> Without the watcher, `POST /api/ask` will return `202` and the question will never be answered. The watcher advertises capabilities based on the provider credentials in its environment; ensure the chosen provider's environment variables are set.

### Watcher Capabilities

A watcher advertises a **capability** for each provider or service whose credentials are present in its environment, plus `maintenance` (always available). The API only routes a job to a capability a running watcher actually offers, so a job stays queued until a capable watcher is running.

| Capability | Required env |
| --- | --- |
| `openai-compatible` | `OPENAI_COMPATIBLE_BASE_URL`, `OPENAI_COMPATIBLE_API_KEY`, `OPENAI_COMPATIBLE_MODEL` |
| `azure-openai` | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_CHAT_DEPLOYMENT` |
| `codex` | `CODEX_CLI_PATH` (defaults to `codex` on `PATH`) |
| `claude` | `CLAUDE_CLI_PATH` (defaults to `claude` on `PATH`) |
| `local-git` | `MAGPIE_GIT_AUTHOR_NAME`, `MAGPIE_GIT_AUTHOR_EMAIL` (and git on `PATH`) |
| `github` | `GITHUB_TOKEN`, `MAGPIE_GIT_AUTHOR_NAME`, `MAGPIE_GIT_AUTHOR_EMAIL` |
| `maintenance` | (none) |

For CLI-based providers (`codex` and `claude`), you can optionally choose the model used by the watcher by setting `CODEX_CLI_MODEL` or `CLAUDE_CLI_MODEL`. When set, the watcher appends `--model <value>` to the CLI invocation, overriding the CLI's default model.

Note: A watcher with `github` credentials also satisfies `local-git` (it has git + author identity), so it can publish to both remote and local destinations. `local-git` alone publishes only to `file://` destinations (push, no PR).

> **Important for maintenance jobs:** Markdown Magpie includes maintenance orchestrators — such as gap‑closure verification (`verify_gap_closure`), patrol tidying, and the gap reconciler — that claim a watcher and then *block* while waiting on follow‑up AI jobs they enqueued. Because a watcher runs **one job at a time**, those follow‑ups can only be picked up by a **second** watcher. With a single watcher the orchestration self‑starves and times out. The console warns when only one watcher is connected. For local development with maintenance features, start a second watcher in another terminal:
>
> ```bash
> AUTH_REQUIRED=false WATCHER_API_CLIENT_ID= WATCHER_API_CLIENT_SECRET= \
>   MAGPIE_CHECKOUT_ROOT="$PWD/.magpie/checkouts" npm run dev:watcher
> ```

### Client Flow

The standard request/await pattern for any job-backed endpoint:

1. `POST` the work — the API returns **`202`** with the created job and links.
2. `GET /api/jobs/:id/wait` — long-polls. Returns **`200`** once the job is terminal, or **`202`** if it is still running.
3. `GET /api/jobs/:id` — fetch the job snapshot at any time without blocking.

Job states: `created` → `active` → `completed` (terminal). Other states: `retry` (queued for another attempt after a recoverable failure), `failed` (terminal, retries exhausted), `cancelled` (terminal, cancelled by an operator), `blocked` (waiting on a dependency / singleton key).

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

### Viewing Flow Status

Additional API endpoints for monitoring:

- `GET /api/knowledge/repositories` – list indexed repositories (paginated, default limit 50, capped at 200). Each response includes a `total` field with the full count.
- `GET /api/knowledge/stats` – get total document and section counts.
- `GET /api/knowledge/documents` – list indexed documents across flows (paginated, default limit 50, capped at 200). Each response includes a `total` field with the full count.
- `GET /api/config` – reports the runtime configuration, including the active retrieval mode (`hybrid` or `keyword`) and AI provider. The retrieval mode is also shown with a plain-language `reason`.

```bash
curl -s http://localhost:4000/api/config | jq .retrieval
# {"mode":"hybrid","reason":"Postgres + embedding credentials configured"}
```

## 11. (Optional) Start the Web Console

In a separate terminal, start the Next.js web app:

```bash
MAGPIE_DEV_API_PROXY="http://localhost:4000" npm run dev:web
```

Open `http://localhost:3000` to browse the knowledge base, view questions, and manage proposals.

> **Note:** If you see a blank page with failing chunk requests, the most likely cause is that `MAGPIE_DEV_API_PROXY` was not set before starting the web server. The Next dev server does not read the repo-root `.env`, so the variable must be set in the shell or in `apps/web/.env.local`. Without it, the browser tries same-origin API calls on port 3000 and saturates the connection limit.

## 12. (Optional) Run the MCP Server

The MCP server (`@magpie/mcp`) lets AI agents and MCP-aware clients query the knowledge base through tools such as `kb_ask`, `kb_search`, `kb_feedback`, `kb_flows`, `kb_outline`, and `kb_seed`. It is a thin client over the HTTP API — it holds no state and proxies every request to the API at `API_BASE_URL`. The API and a watcher must be running before `kb_ask` can answer questions.

### stdio Transport (local subprocess)

Build and run the stdio server:

```bash
npm run build -w @magpie/mcp
API_BASE_URL=http://localhost:4000 node apps/mcp/dist/main.js
```

The server communicates over stdin/stdout using JSON-RPC messages (one JSON line per message, no embedded newlines). Logging goes to stderr. A project-scoped `.mcp.json` at the repository root registers the server with Claude Code automatically.

### Streamable HTTP Transport (network)

Run the HTTP server on a configurable port (default `4001`):

```bash
API_BASE_URL=http://localhost:4000 npm run start:http -w @magpie/mcp
```

The MCP endpoint is at `http://localhost:4001/mcp`. The server also exposes a health check at `http://localhost:4001/health`. The port is set by `MCP_HTTP_PORT` (default `4001`) and the bind host by `MCP_HTTP_HOST` (default `127.0.0.1`; set to `0.0.0.0` to expose on all interfaces).

### Available Tools

| Tool | Description | Required scope |
|------|-------------|----------------|
| `kb_search` | Search indexed Markdown sections by keyword. | `read:knowledge` |
| `kb_ask` | Ask a question and get a cited answer from the knowledge base. | `ask:knowledge` |
| `kb_feedback` | Flag an answer as helpful, unhelpful, or a knowledge gap. | `feedback:questions` |
| `kb_flows` | List the knowledge flows a question can be routed to. | `read:knowledge` |
| `kb_outline` | Propose a seed outline (list of documents to author) for a topic, grounded in a flow's existing docs. | `manage:jobs` |
| `kb_seed` | Seed a flow with initial content: draft documents straight into proposals, skipping the gap pipeline. | `manage:jobs` |

For authentication details, per-tool scopes, and client setup instructions (Claude Code, Claude Desktop, VS Code, Cursor, Continue), see [docs/mcp.md](../docs/mcp.md).

When `AUTH_REQUIRED=true`, the HTTP transport acts as an OAuth protected resource, validating inbound tokens against Auth0. The stdio transport presents `MCP_AUTH_TOKEN` to the API. Both fail fast at startup if required credentials are missing.

## 13. Bootstrap Content with Flow Seeding

The fastest way to populate a new knowledge base is **flow seeding**: you submit a list of documents to author (each a title plus the points it should cover), and the system drafts each one straight into a proposal → pull request, bypassing the gap-clustering pipeline entirely. This is the recommended on-ramp for a new flow or a new area of knowledge (e.g. a new feature).

### Direct API Call

```bash
curl -X POST http://localhost:4000/api/flows/my-flow/seed \
  -H 'content-type: application/json' \
  -d '{
    "items": [
      {
        "title": "Billing overview",
        "coverage": ["what billing is", "the available plans"]
      },
      {
        "coverage": ["refund policy", "how to request a refund"]
      }
    ]
  }'
```

Each item's `coverage` field (the points the document must cover) is required (≥1). `title` and `targetPath` are optional. An optional `questions` array supplies motivating prompts for context. The endpoint requires the `manage:jobs` scope and `manage` capability on the target flow.

The API enqueues one `draft_seed_document` job per item and returns `{ "ok": true, "jobIds": [...] }`. Drafting is source-grounded: the executing agent explores the flow's source repositories (git/local/internet/agent) to find supporting material. Coverage points the sources do not support are omitted from the authored document and recorded on the proposal's rationale. On completion, each document is created as a clusterless proposal that goes through the same reconcile gate as gap drafts (folds into overlapping open PRs, or self-publishes as its own PR), so seeding still ends at a reviewable pull request.

### Using the Web Console

The console's **Seed / add an area** page drives the full workflow: pick a flow, enter a topic, generate an outline, edit the proposed documents, then seed. See the Seed section in the sidebar at `http://localhost:3000/seed`.

### Using the MCP Tools

The MCP server exposes `kb_outline` (proposes documents for a topic, grounded in the flow's existing docs) and `kb_seed` (drafts the items straight into proposals). The typical two-step flow is:

1. `kb_outline` with `{ "flow": "my-flow", "topic": "Refund handling" }` — returns proposed `SeedItem[]`.
2. Review and edit the items, then call `kb_seed` with `{ "flow": "my-flow", "items": [...] }`.

This bypasses the need to write coverage points by hand.

### Generating the Outline

If you do not have a precise list of items, you can let the system propose one:

```bash
curl -X POST http://localhost:4000/api/flows/my-flow/outline \
  -H 'content-type: application/json' \
  -d '{
    "topic": "Refund handling",
    "notes": "focus on partial refunds"
  }'
```

The API grounds the outline job in the flow's existing documents (retrieved inline for the topic) so the model proposes items that fit the current structure and do not restate what is already covered. The job returns `{ items: SeedItem[], rationale }`. It **only proposes** — nothing is drafted or seeded. Review and edit the returned items, then submit them via the seed endpoint above or the `kb_seed` MCP tool.

## 14. Switching AI Provider at Runtime

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

## 15. Embedding Configuration (Optional)

For hybrid retrieval (keyword + vector), configure an embeddings provider **independently** of the chat provider. The following sets work:

**OpenAI‑compatible** (e.g., OpenAI, DeepSeek, local vLLM):

```env
OPENAI_COMPATIBLE_BASE_URL=https://api.openai.com/v1
OPENAI_COMPATIBLE_API_KEY=sk-...
OPENAI_COMPATIBLE_EMBEDDING_MODEL=text-embedding-3-small
```

If `OPENAI_COMPATIBLE_EMBEDDING_BASE_URL` and `OPENAI_COMPATIBLE_EMBEDDING_API_KEY` are left blank, the system falls back to the common chat values (`OPENAI_COMPATIBLE_BASE_URL` and `OPENAI_COMPATIBLE_API_KEY`).

**Azure OpenAI**:

```env
AZURE_OPENAI_ENDPOINT=https://myinstance.openai.azure.com
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=text-embedding-3-small
```

Hybrid mode activates automatically when `KNOWLEDGE_STORE=postgres` **and** a complete set of embedding credentials is present. The model must output 1536‑dimensional vectors to be compatible with the existing vector index.

Note: `EMBEDDING_PROVIDER` is informational only — it is surfaced in `/api/config` for display and does not enable embeddings. Setting `OPENAI_COMPATIBLE_EMBEDDING_MODEL` is what enables OpenAI-compatible embeddings.

## 16. Managing Documents in the Knowledge Base

The knowledge base is the indexed corpus used to answer questions. Documents are sourced from configured flows and stored in a Git-backed destination repository.

### Adding Documents

New documents are added by **including them in the destination repository** and then re-indexing. Magpie does not provide a separate "upload" endpoint; instead, you:

1. Add your Markdown file(s) to the destination repository (either directly via Git, through a proposal/pull request, or by flow seeding).
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
2. Generate a proposal from that cluster. You can either post the cluster ID to the API or use the scheduled reconciler:
   ```bash
   curl -X POST http://localhost:4000/api/proposals/from-gap \
     -H 'content-type: application/json' \
     -d '{"summary":"No source material found for: How do I trim claws?","destinationId":"cats-docs"}'
   ```
   The API enqueues a `draft_markdown_proposal` job. When the watcher completes it, the generated Markdown is stored as a proposal.
3. Review the draft proposal via `GET /api/proposals` or the web console's **Proposals** page.
4. Once `ready`, publish it: the system commits to a branch and (if the destination is a GitHub remote) opens a pull request.
5. After review and merge, the new document is part of the destination.

#### Adding via Flow Seeding (recommended for new areas)

To bootstrap a new area of knowledge — or an entire new flow — use the seed endpoint as described in [Section 13](#13-bootstrap-content-with-flow-seeding). Each item is drafted into a proposal → pull request, skipping the gap pipeline entirely.

### Removing Documents

To remove a document, you must **delete the file from the destination repository** and then re-index.

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

If a document has become obsolete or inconsistent, the scheduled **correctness patrol** can detect duplicates, contradictions, and overgrown documents. When it finds a problem, it creates a proposal with a multi-file changeset (dedupe, split, or correct) that can be reviewed and published as a pull request.

## 17. Keeping Flows Updated

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

**Warning:** This is destructive. It requires the `manage:admin` scope and the `admin` capability (a super-admin role); never expose it without these restrictions in production.

## 18. The Gap Pipeline and Proposals

Every question asked via `/api/ask` or the MCP tool `kb_ask` is logged. Low-confidence answers and user-flagged gaps are clustered per flow. The `gaps-to-pull-requests` reconciler then:
0. **Prunes resolved gaps** from active clusters: a gap is resolved by `(question, summary)` when its proposal merges and passes gap-closure verification, but a prior reshape may have moved that gap into a cluster other than the one the merge freezes. So each tick deactivates the cluster membership of any gap now resolved, and freezes any active cluster left with no still-open members — keeping "active membership" to mean "this gap belongs to this cluster *and* is still open", so a covered gap never re-surfaces as a cluster member or gets re-drafted.
1. Groups related gaps into clusters. Optionally reshapes clusters (merge/split) via a provider‑partitioned AI job — if no chat watcher is available, reshape is skipped.
2. Drafts Markdown proposals that fill those gaps (scoped to a cluster's still-open gaps only).
3. Publishes them to a Git branch in the flow's destination repository.
4. Opens a pull request (if the destination is a GitHub remote) and advances the proposal as the PR is merged.
5. Checks open pull requests: when a PR merges, the system re-indexes the knowledge base and enqueues a `verify_gap_closure` job to re-ask the triggering questions; the gaps are resolved only if the merged document confidently answers those questions. If verification fails, the gaps stay open for another draft. If a PR closes without merging, the proposal is marked `rejected`.

Proposals are always relative to the flow's destination. You can review draft proposals via `GET /api/proposals` or the web console's **Proposals** page.
The reconciler maintains persisted gap clusters, each with an `id`, `title`, `questionIds`, `count`, and optional `rationale`. Clusters are surfaced via `GET /api/gaps/clusters` and provide a fast read without model calls. Clustering happens in the background reconciler, not on request.

### Proposal Lifecycle

A proposal moves through these statuses: `draft`, `ready`, `branch-pushed`, `pr-opened`, `merged`, `rejected`, `superseded`. Once a proposal is `ready`, it can be published:

```bash
POST /api/proposals/:id/status
{ "status": "ready" }

POST /api/proposals/:id/publish
```

Publication is enqueue-only. The watcher commits the Markdown to a `magpie/proposal-*` branch and pushes it. For a GitHub destination it then opens a pull request; for a local-git (file://) destination it stops at the pushed branch (no PR to open) and the human reviewer uses the console's **Accept** (merge) or **Bin** (reject) action — Accept merges the branch into the default branch, resolves gaps, and re-indexes; Bin marks the proposal rejected, freezes its gap cluster, and deletes the review branch. The proposal records the branch, commit SHA, and (for GitHub) PR URL. If no host token is available, it degrades gracefully to a pushed branch.

### Verification and Reopening

When a proposal is merged, `verifyGapClosure` re-asks each triggering question to confirm whether the merged content resolves the gap.

- If a re-ask returns a confident answer citing the merged document, the gap is resolved immediately for that question — even if other questions in the same cluster are still open.
- If a re-ask is still open (low confidence or not citing the merged doc), a **`verification`** gap row is recorded. The summary filed under is determined from the proposal's persisted cluster membership (which carries the per-question association) — so the reopen is attached to the correct gap for that specific question, not the question's oldest open gap or another question's gap. If there is no cluster, the system intersects the question's own still-open gap summaries with the proposal's recorded summaries, falling back to the question text. This ensures the reopen dedups with the existing gap in candidate clustering and avoids misfiling on multi-gap or multi-question scenarios.
- After two failed verifications for the same question (`CLOSURE_RETRY_CAP`), its gap is marked `needs_attention` and the reconciler stops re-drafting it.
- The verification gap's note is passed to the drafter as `resubmissionNotes` when the gap is re-drafted, so the model sees why its previous attempt did not close the gap.

## 19. Patrol Maintenance: Scheduled Knowledge Base Tidying

Rather than a single whole-knowledge-base crunch, Magpie now runs several **patrol** lenses on a rolling cursor. Each scheduled tick selects the least-recently-checked documents in a flow and runs one or more lenses over them:

- **Correctness patrol** – verify, dedupe, split. The verify, dedupe, and split lenses propose corrections, consolidations, and splits.
  - **Verify** – checks document claims against source material; flags unprovable statements.
  - **Correct** – automatically rewrites flagged documents.
  - **Dedupe** – finds near-duplicates and reconciles pairs.
  - **Split** – breaks overgrown documents into focused pieces.
- **Editorial patrol** – expand thin docs. Rolls its own cursor across this flow's knowledge-base documents, sending the least-recently-improved ones to an agent that explores the flow's source repositories so fine-but-thin documents grow source-backed coverage. Separate from the Correctness patrol: it proposes editorial expansion, not correctness or structural fixes.

Each patrol produces a `MaintenanceRun` record surfaced in the Activity page. Proposals created by patrols are clusterless and go through the same reconcile gate (fold into existing open PRs on overlap, publish as own PR otherwise). The correctness patrol is conservative (only acts when something is demonstrably wrong), while the editorial patrol is proactive (grows fine-but-thin docs).

Patrol schedules can be enabled/disabled per flow and cron expression via the **Schedules** page in the web console.

### Patrol Trigger

The patrol uses a rolling cursor: each tick picks the N least-recently-checked files (or files past a staleness threshold) and runs the lenses on them. This rotates through the whole KB over days with bounded cost per tick, and naturally re-visits files as they age.

## Best Practices

- Use separate git repositories for sources and destinations to control access and review cycles.
- Keep the destination repository human‑readable: it is the authoritative knowledge base for your customers.
- If a flow has no sources (e.g., you write documentation directly in the destination repo), omit `sourceIds` and let the gap pipeline propose changes from user questions alone.
- Monitor the `GET /api/gaps/clusters` endpoint to see which missing topics are most frequently asked.
- For production deployments, configure `GITHUB_TOKEN` (or equivalent) so proposals automatically become pull requests.
- Use descriptive, unique headings that summarize section content.
- Keep sections focused on a single topic.
- Avoid extremely long sections; break them into smaller, well-named subsections.

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
| API fails to start with "Auth0 not configured" error | `AUTH_REQUIRED` is true (default) but `AUTH0_AUDIENCE` is missing or invalid | Set `AUTH_REQUIRED=false` for local development, or configure Auth0 variables correctly |
| `/ask` returns low confidence or `no source material` | No indexed content or embedding incomplete | Verify indexing; wait for background embedding to finish |
| `/ask` returns `202` but never completes | The watcher is not running. | Start the watcher (step 7) and retry the question. |
| `/ask` returns low confidence even after indexing | Embedding pass not finished; retrieval mode is keyword | Wait for background embedding or configure hybrid retrieval |
| Watcher logs `Capability … not ready` | Environment variables for the chosen provider are incorrect | Check provider env vars, see the table in step 2 |
| `401` on API calls | Authentication enabled but no credentials | Set `AUTH_REQUIRED=false` in `.env` or as environment variable |
| `MAGPIE_CHECKOUT_ROOT` not writable | Override not set | Use `MAGPIE_CHECKOUT_ROOT="$PWD/.magpie/checkouts"` before starting the API |
| "local_path_not_allowed" error | Trying to index an arbitrary path without a configured flow | Use a flow ID defined in `KNOWLEDGE_FLOWS` |
| Changes not reflected after re-index | Browser caching of search results | Use a cache-busting parameter or wait for TTL; re-query the API |
| Bootstrap fails with permission error | Override `MAGPIE_CHECKOUT_ROOT` to a writable local path | Use the command from step 6 |
| "Failed to sync configured git repositories" | `MAGPIE_CHECKOUT_ROOT` is not writable or missing; a private remote source requires credentials | Create the directory and ensure write permissions; verify `git clone <source-url>` works standalone |
| Hybrid retrieval not active | Embedding credentials incomplete or `KNOWLEDGE_STORE` not set | Check that `KNOWLEDGE_STORE=postgres` and a complete set of embedding credentials are set |
| `/api/ask` returns 202 (queued) | This is normal; the answer is being processed | Poll `GET /api/jobs/<id>/wait` until the job completes |
| Index returns "0 documents" | Destination checkout not synced or path wrong; or knowledge config silently dropped | Verify `MAGPIE_CHECKOUT_ROOT` and that the destination repo is cloned. Check API startup logs for sync errors. Also inspect `KNOWLEDGE_SOURCES`, `KNOWLEDGE_DESTINATIONS`, and `KNOWLEDGE_FLOWS` — a stray `==` or a flow whose `sourceIds` don't match any source will be silently dropped. |
| New document not found in search | Index did not run after adding the file | Run index endpoint again. |
| Re-index takes a long time | Embedding pass for many new sections | Wait; embedding runs in background and is idempotent. |
| Knowledge config silently dropped (syncing count 0) | Malformed environment variable (e.g., `KNOWLEDGE_SOURCES==[...]` with a stray `==`) or flow/source id mismatch | Check the exact syntax of `KNOWLEDGE_SOURCES`, `KNOWLEDGE_DESTINATIONS`, and `KNOWLEDGE_FLOWS`. Ensure each flow's `sourceIds` reference existing source ids and `destinationId` references an existing destination. |
| Web UI: blank page, all `_next/static/chunks/*.js` fail | `MAGPIE_DEV_API_PROXY` not set; browser saturates same-origin connection limit | Set `MAGPIE_DEV_API_PROXY=http://localhost:4000` in the shell or in `apps/web/.env.local` before starting the web dev server. |
| Proposals not appearing as PRs | No git host token configured | Set `GITHUB_TOKEN` or equivalent and ensure the `refresh_flow_snapshot` scheduler is running. |
| Patrol plan says "no changes needed" | Destination already well‑structured | Configure smaller interval or manually trigger a full analysis. |
| Web console shows no flows | Environment variables not set | Verify `KNOWLEDGE_FLOWS` in `.env` and restart the API. |

## Next Steps

- Learn about [knowledge gap detection and proposals](managing-knowledge-flows-in-markdown-magpie.md#the-gap-pipeline-and-flows).
- Set up [real AI providers](integrations-and-connecting-data-sources.md#ai-provider-integrations) instead of `mock`.
- Configure [hybrid retrieval with embeddings](configuration-reference.md#embedding-provider-configuration) for improved answer quality.
- Configure automated [patrol maintenance](managing-knowledge-flows-in-markdown-magpie.md#patrol-maintenance-scheduled-knowledge-base-tidying) for knowledge base tidying.
- Review the [permissions and access controls](permissions-and-access-controls-in-markdown-magpie.md).
- Review the [Configuration Reference](configuration-reference.md) for comprehensive documentation of all environment variables.
- Understand the [architecture and job model](architecture.md).
- Try the [full Docker Compose stack](README.md#docker-compose) for a self-contained demo.
- Connect AI agents via the [MCP server](docs/mcp.md) for direct knowledge base queries.

---

**References**
- [Repository README](../../README.md) – full local development instructions.
- [Markdown Ingestion](managing-knowledge-flows-in-markdown-magpie.md) – detailed source/destination/flow configuration.
- [HTTP API Reference](managing-knowledge-flows-in-markdown-magpie.md#api-endpoints) – all API endpoints for indexing and searching.
- [Architecture](managing-knowledge-flows-in-markdown-magpie.md) – high-level system design and provider strategy.
- [Configuration Reference](configuration-reference.md) – comprehensive documentation of all environment variables.
- [MCP Server Reference](docs/mcp.md) – MCP transports, tools, auth, and client setup.
- [AI Job Contract](docs/ai-jobs.md) – queued AI jobs, seeding, and the watcher model.
