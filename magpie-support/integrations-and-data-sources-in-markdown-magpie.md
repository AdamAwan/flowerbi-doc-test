---
title: Integrations and Data Sources in Markdown Magpie
status: draft
---

# Integrations and Data Sources

Markdown Magpie connects to a variety of external systems for knowledge ingestion, AI providers, deployment, and agent collaboration. This guide explains each integration point and how to configure them.

## Data Sources (Knowledge Ingestion)

Markdown Magpie ingests Markdown content from *sources* and writes reviewed proposals to *destinations*. Both are configured using environment variables.

### Source & Destination Configuration

Define sources and destinations in environment variables as JSON arrays:

| Variable | Description |
|---|---|
| `KNOWLEDGE_SOURCES` | Array of source objects where Markdown content is read from. |
| `KNOWLEDGE_DESTINATIONS` | Array of destination objects where curated knowledge and proposals are written. |
| `KNOWLEDGE_FLOWS` | Links sources to a destination for a given knowledge base. |
| `MAGPIE_CHECKOUT_ROOT` | Local path where git repositories are cloned (default `.magpie/checkouts`). |

For backward compatibility, `KNOWLEDGE_REPOSITORIES` and `KNOWLEDGE_REPO_PATH` are still supported when the new variables are not set.

Example configuration:

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts
KNOWLEDGE_SOURCES=[{"id":"flowerbi","name":"FlowerBI Source","url":"https://github.com/danielearwicker/flowerbi.git","subpath":"src"}]
KNOWLEDGE_DESTINATIONS=[{"id":"flowerbi-docs","name":"FlowerBI Docs","url":"https://github.com/AdamAwan/flowerbi-doc-test.git","subpath":"docs"}]
KNOWLEDGE_FLOWS=[{"id":"flowerbi","name":"FlowerBI KB","sourceIds":["flowerbi"],"destinationId":"flowerbi-docs"}]
```

### Supported Source Kinds

| Kind | Configuration | Example |
|---|---|---|
| `local` | `{ "path": "knowledge-bases/cats" }` | A folder inside the repository. |
| `git` | `{ "url": "https://...", "subpath": "docs" }` | Clone a remote repository, optionally using a subfolder. |
| `internet` | `{ "kind": "internet", "url": "https://example.com/docs" }` | Fetch Markdown from a public URL. |
| `agent` | `{ "kind": "agent" }` | Content provided by an AI agent (e.g., from Claude or Codex). |

Sources with kind `agent` or `internet` are used for drafting proposals but are **not indexed** into the answer corpus. Only destinations (curated knowledge bases) are indexed for answering questions.

### Indexing a Data Source

After configuration, index a flow's destination with the HTTP API:

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"flowerbi"}'
```

The API walks the clone, parses Markdown and frontmatter, splits sections, stores them in PostgreSQL, and (if an embeddings provider is set) generates vector embeddings for hybrid search.

## AI Providers (Chat & Embeddings)

Markdown Magpie keeps AI provider logic behind pluggable adapters. You can configure different providers for answering questions and for generating embeddings.

### Chat Providers

Set `AI_PROVIDER` to control how answers are synthesised. The API enqueues all generative work as jobs on a pg-boss queue; a separate watcher process claims and completes them. The system is queue-only — there is no synchronous 'direct' mode; the watcher is required for any AI work.

| Provider | Environment Variables | Notes |
|---|---|---|
| `openai-compatible` | `AI_PROVIDER=openai-compatible`, `OPENAI_COMPATIBLE_BASE_URL`, `OPENAI_COMPATIBLE_API_KEY`, `OPENAI_COMPATIBLE_MODEL` | Works with any OpenAI-compatible API (OpenAI, DeepSeek, OpenRouter, etc.). |
| `azure-openai` | `AI_PROVIDER=azure-openai`, `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_CHAT_DEPLOYMENT`, `AZURE_OPENAI_API_VERSION` | Azure OpenAI service. |
| `codex` | `AI_PROVIDER=codex`, `CODEX_CLI_PATH`, `CODEX_CLI_ARGS`, `CODEX_CLI_PROMPT_MODE` | Uses a local Codex CLI. |
| `claude` | `AI_PROVIDER=claude`, `CLAUDE_CLI_PATH`, `CLAUDE_CLI_ARGS`, `CLAUDE_CLI_PROMPT_MODE` | Uses a local Claude CLI. |

> **Note:** The `mock` provider has been removed. For the current list of supported providers and their required variables, see the [Configuration Reference](configuration-reference.md).

Switch providers at runtime via `POST /api/config`. The provider must be configured in environment variables for the change to be accepted.

### Embedding Providers

Embeddings are required for hybrid (keyword + vector) retrieval. Configure them independently of chat providers:

| Provider | Required Variables |
|---|---|
| OpenAI-compatible embeddings | `OPENAI_COMPATIBLE_EMBEDDING_MODEL`, plus base URL (`OPENAI_COMPATIBLE_EMBEDDING_BASE_URL`, falls back to `OPENAI_COMPATIBLE_BASE_URL`) and API key (`OPENAI_COMPATIBLE_EMBEDDING_API_KEY`, falls back to `OPENAI_COMPATIBLE_API_KEY`). |
| Azure OpenAI embeddings | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_EMBEDDING_DEPLOYMENT`. |

To enable hybrid retrieval, you must also have `KNOWLEDGE_STORE=postgres` and `DATABASE_URL` set. The system automatically activates hybrid mode when both the database and a complete set of embedding credentials are present.

## Git Integration (Sync & Proposals)

Markdown Magpie syncs git repositories on startup and uses them for proposal branches.

### Key features

- **Source sync**: Sources and destinations defined with `url` are cloned or fast-forwarded into `MAGPIE_CHECKOUT_ROOT` during API startup and on the `source-change-sync` background task.
- **Proposal branches**: Proposals are committed to a new branch (`magpie/proposal-*`) using the `local-git` publisher. If a `GITHUB_TOKEN` is provided, the branch is also raised as a pull request.
- **Patrol proposals**: Maintenance patrol operations create proposals that are published to `magpie/proposal-*` branches, following the same lifecycle.
- **Crunch branches**: Knowledge-base tidy operations are published to `magpie/crunch-*` branches (legacy; patrol proposals are the current approach).

### Pull Request Provider

The `gaps-to-pull-requests` reconciler automatically advances proposals through the lifecycle: draft → ready → branch-pushed → pr-opened → merged → rejected. It polls open PRs, and when a PR merges it re-indexes the destination and enqueues a gap-closure verification — the triggering questions are re-asked, and only if the merged document confidently answers them are the gaps resolved. If a PR closes without merging, the proposal is marked rejected.

| Provider | Environment Variables |
|---|---|
| GitHub | `GITHUB_TOKEN` |
| GitLab | `GITLAB_TOKEN` (planned) |
| Azure DevOps | `AZURE_DEVOPS_TOKEN` (planned) |

If no token is set, proposals degrade gracefully to a pushed branch.

## Deployment Integrations

### Docker Compose (Default)

The single-host deployment shape uses Docker Compose with optional managed services:

| Service | Notes |
|---|---|
| API container (`:4000`) | HTTP server. |
| Web container (`:3000`) | Next.js admin UI. |
| MCP container | stdio or HTTP MCP server (optional, via profile). |
| PostgreSQL with pgvector | Primary database. |
| Redis (optional) | Needed only if a Redis-backed queue adapter is selected. |
| Local object storage | For raw document snapshots (optional, not yet implemented). |

### Azure (Optional Managed Path)

Azure is the preferred managed cloud provider when a hosted environment is required. Likely managed services (from [`infra/azure/README.md`](infra/azure/README.md)):

- Azure Container Apps for API, web, MCP, and worker containers.
- Azure Database for PostgreSQL with vector support.
- Azure Cache for Redis (if a Redis queue adapter is used).
- Azure Blob Storage for raw snapshots and artifacts.
- Azure OpenAI for chat and embeddings.
- Azure DevOps Repos or GitHub for pull request workflows.
- Microsoft Entra ID for organisation authentication.

Portability is maintained; Azure adapters should not leak into core domain logic.

## MCP Integration (Model Context Protocol)

The MCP server (`apps/mcp`) is a thin proxy over the HTTP API, allowing AI agents (Claude Code, Codex, etc.) to interact with the knowledge base via standard MCP tools.

### Supported MCP Transports

| Transport | How It Works | Recommended For |
|---|---|---|
| **stdio** | MCP messages over stdin/stdout. The client launches the server as a subprocess. | Local development with Claude Code or similar. |
| **Streamable HTTP** | Long-lived HTTP server using SSE. Clients make JSON-RPC POST requests. | Network-accessible deployments, multi-client scenarios. |

### MCP Tools

- `kb.ask`: Ask a question with citations and gap detection. Accepts an optional `flow` parameter. When `flow` is `"auto"` (default) the question is routed normally; otherwise it must be a flow id from `kb.flows`. If the router cannot determine a flow, the response includes `flowSelectionRequired` with the available flows – call `kb.ask` again with `flow` set to one of those ids.
- `kb.search`: Search indexed sections by keyword.
- `kb.flows`: List the knowledge flows a question can be routed to. Returns the ids and names of configured flows. Use the returned ids as the `flow` argument to `kb.ask`.
- `kb.feedback`: Record feedback (`helpful`/`unhelpful`/`knowledge_gap`) on a past answer.
- `kb.outline`: Propose a seed plan for a flow by exploring its source repositories — **no topic needed**. It enqueues the source-grounded `outline_flow_seed` job, waits for it, then returns the **persisted plan** its completion created. It **only proposes** — nothing is drafted; the plan waits behind the review gate. Approve it with `kb_seed`, or review/edit it in the console. Input: `{ "flow": string, "notes"?: string }` (`notes` is an optional steer for this run). Returns `{ "planId": string, "charter"?: string, "charterProposed": boolean, "persona"?: string, "personaProposed": boolean, "items": SeedItem[], "rationale"?: string }`, where each `SeedItem` is `{ title?, targetPath?, coverage: string[], questions?: string[] }`. The `*Proposed` flags record that the charter/persona came from the model because the flow config lacked one — copy the value into `KNOWLEDGE_FLOWS` to make it permanent.
- `kb.seed`: Approve a seed plan (from `kb_outline` or the console): drafts one document per approved item straight into the proposal → pull-request pipeline, carrying the plan's run-scoped charter/persona. Edit or partially dismiss items in the console first if needed. Input: `{ "plan": string }` — the plan id (from `kb_outline`'s `planId`, or the console). Returns `{ "planId": string, "jobIds": string[] }` — one enqueued `draft_seed_document` job per approved item. See [ai-jobs.md](ai-jobs.md#seeding-a-flow) for the full seeding flow.

### Configuration

Common variables: `API_BASE_URL` (default `http://localhost:4000`), `ANSWER_POLL_INTERVAL_MS`, `ANSWER_TIMEOUT_MS`, `OUTLINE_POLL_INTERVAL_MS`, `OUTLINE_TIMEOUT_MS`.

HTTP transport requires `MCP_HTTP_PORT` (default `4001`) and optionally `MCP_RESOURCE_URL`. Auth requires `MCP_API_AUTH_TOKEN` when `AUTH_REQUIRED=true`.

For full details, see [`docs/mcp.md`](docs/mcp.md).

## External Agent Providers (Watcher)

The watcher can use a local CLI tool as the AI provider, enabling integration with Codex, Claude Code, or any command-line agent.

| Provider | Environment Variables |
|---|---|
| `codex` | `AI_PROVIDER=codex`, `CODEX_CLI_PATH`, `CODEX_CLI_ARGS`, `CODEX_CLI_PROMPT_MODE` (arg or stdin). |
| `claude` | `AI_PROVIDER=claude`, `CLAUDE_CLI_PATH`, `CLAUDE_CLI_ARGS`, `CLAUDE_CLI_PROMPT_MODE`. |
| `openai-compatible` | Same as chat provider but for watcher process; uses `AI_PROVIDER=openai-compatible`. |

> **Note:** The `mock` provider has been removed. Use one of the providers above or see the [Configuration Reference](configuration-reference.md) for the full list.
The agent must return JSON matching the job output schema. The watcher validates and posts results back to the API.

## Storage Backends

| Variable | Purpose | Options |
|---|---|---|
| `STORAGE_BACKEND` | Primary storage for questions, gaps, proposals, and AI jobs. | `postgres` (recommended), `memory` (local dev only). |
| `KNOWLEDGE_STORE` | Where indexed sections and embeddings are stored. | `postgres` (required for hybrid retrieval), `memory`. |
| `DATABASE_URL` | PostgreSQL connection string. | Postgres with pgvector extension. |

Redis is started by Docker Compose but is not required for default setup. It will be used when a Redis-backed queue adapter is selected.

## Authentication

Authentication **fails closed**: it is required by default unless explicitly disabled via `AUTH_REQUIRED=false`. When enabled, tokens from Auth0 are validated locally using JWKS.

- **API**: Validates `Authorization: Bearer <token>` on endpoint requests.
- **MCP HTTP**: Acts as an OAuth protected resource; requires per-tool scopes (`read:knowledge`, `ask:knowledge`, `feedback:questions`, `manage:jobs`).
- **MCP stdio**: Presents `MCP_AUTH_TOKEN` to the API.

All authentication is configurable via environment variables. See [`docs/mcp.md`](docs/mcp.md) and the `@magpie/auth` package.

## Quick-Start: Connecting Your Data Source

1. **Set environment variables** in `.env`:

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts
KNOWLEDGE_SOURCES=[{"id":"my-source","url":"https://github.com/org/docs.git","subpath":"articles"}]
KNOWLEDGE_DESTINATIONS=[{"id":"my-dest","url":"https://github.com/org/mykb.git","subpath":"docs"}]
KNOWLEDGE_FLOWS=[{"id":"myflow","sourceIds":["my-source"],"destinationId":"my-dest"}]
STORAGE_BACKEND=postgres
DATABASE_URL=postgres://...
AI_PROVIDER=openai-compatible
```

> **Note:** The `mock` provider has been removed. Use `openai-compatible` or another supported provider. For a complete list, see the [Configuration Reference](configuration-reference.md).

2. **Start Postgres** (if not already running):

```bash
docker compose up -d postgres
```

3. **Run migrations**:

```bash
npm run db:migrate
```

4. **Start the API**:

```bash
MAGPIE_CHECKOUT_ROOT="$PWD/.magpie/checkouts" npm run dev:api
```

5. **Index the destination**:

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index -H 'content-type: application/json' -d '{"flowId":"myflow"}'
```

6. **Ask a question**:

```bash
curl -s http://localhost:4000/api/ask -H 'content-type: application/json' -d '{"question":"What does my knowledge base cover?"}'
```

Your data source is now integrated and searchable.

## Step-by-Step: Connect a New Data Source

1. **Add the source** to `KNOWLEDGE_SOURCES` in `.env`:
   ```json
   [{"id":"my-api-docs","url":"https://github.com/myorg/api-docs.git","subpath":"content"}]
   ```
2. **Add a destination** repository for the curated KB:
   ```json
   [{"id":"my-kb","url":"https://github.com/myorg/knowledge-base.git","subpath":"docs"}]
   ```
3. **Create a flow** linking source(s) to the destination:
   ```json
   [{"id":"api-kb","sourceIds":["my-api-docs"],"destinationId":"my-kb"}]
   ```
4. **Set `MAGPIE_CHECKOUT_ROOT`** to a writable directory and ensure the API can clone repositories.
5. **Start the API** and index the flow:
   ```bash
   curl -X POST http://localhost:4000/api/knowledge/repositories/index \
     -H 'content-type: application/json' \
     -d '{"flowId":"api-kb"}'
   ```
6. **Ask a question** to verify the data is indexed:
   ```bash
   curl -s http://localhost:4000/api/ask \
     -H 'content-type: application/json' \
     -d '{"question":"How do I authenticate?"}'
   ```

## Summary

Markdown Magpie integrates with:

- **Data sources**: local, git, internet, agent
- **AI providers**: OpenAI‑compatible, Azure OpenAI, external CLI agents (Codex, Claude)
- **Embedding providers**: OpenAI‑compatible, Azure OpenAI
- **Storage**: Postgres (recommended), Redis, in‑memory
- **Version control**: Git, GitHub (PR support), GitLab/Azure DevOps (planned)
- **Deployment**: Docker Compose, Azure (optional)
- **Identity**: Auth0 (optional)
- **MCP clients**: Claude Code, Codex, any MCP‑aware tool

Each integration is provider-neutral behind defined interfaces, so you can mix and match components without modifying core logic.

---

*Based on source material from: `README.md`, `docs/ingestion.md`, `docs/chat-providers.md`, `docs/architecture.md`, `docs/ai-jobs.md`, `docs/mcp.md`, `infra/azure/README.md`.*

## Related Documentation

- [Architecture Overview](docs/architecture.md) – provider strategy and primary flow.
- [Ingestion Guide](docs/ingestion.md) – detailed source/destination configuration and indexing.
- [Chat Providers](docs/chat-providers.md) – AI provider setup.
- [MCP Server](docs/mcp.md) – connecting AI agents.
- [AI Jobs & Watcher](docs/ai-jobs.md) – background job contracts and external agents.
- [API Reference](docs/api.md) – all HTTP endpoints.
- [Configuration Reference](configuration-reference.md) – comprehensive environment variable reference.
- [Azure Deployment Notes](infra/azure/README.md) – managed Azure deployment.
- [README.md](README.md) – local development setup.
