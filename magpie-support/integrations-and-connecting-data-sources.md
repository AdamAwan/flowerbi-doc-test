---
title: Integrations and Connecting Data Sources
status: draft
---

# Integrations and Connecting Data Sources

Markdown Magpie connects to a variety of external systems for knowledge ingestion, AI providers, deployment, and agent collaboration. This guide explains each integration point and how to configure them.

## Data Source Integrations (Knowledge Ingestion)

Markdown Magpie ingests Markdown content from four kinds of sources, configured via `KNOWLEDGE_SOURCES` in your `.env` file.

### Source & Destination Configuration

Define sources and destinations in environment variables as JSON arrays:

| Variable | Description |
|---|---|
| `KNOWLEDGE_SOURCES` | Array of source objects where Markdown content is read from. |
| `KNOWLEDGE_DESTINATIONS` | Array of destination objects where curated knowledge and proposals are written. |
| `KNOWLEDGE_FLOWS` | Links sources to a destination for a given knowledge base. |
| `MAGPIE_CHECKOUT_ROOT` | Local path where git repositories are cloned (default `.magpie/checkouts`). |

Example configuration:

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts
KNOWLEDGE_SOURCES=[{"id":"flowerbi","name":"FlowerBI Source","url":"https://github.com/danielearwicker/flowerbi.git","subpath":"src"}]
KNOWLEDGE_DESTINATIONS=[{"id":"flowerbi-docs","name":"FlowerBI Docs","url":"https://github.com/AdamAwan/flowerbi-doc-test.git","subpath":"docs"}]
KNOWLEDGE_FLOWS=[{"id":"flowerbi","name":"FlowerBI KB","sourceIds":["flowerbi"],"destinationId":"flowerbi-docs"}]
```

### Supported Source Kinds

| Kind | Configuration | Example |
|------|---------------|---------|
| `local` | A local filesystem path | `{"id":"cats","kind":"local","path":"knowledge-bases/cats"}` |
| `git` | A remote Git repository URL, with optional `subpath` | `{"id":"flowerbi","url":"https://github.com/danielearwicker/flowerbi.git","subpath":"src"}` |
| `internet` | A publicly accessible URL (fetched at index time) | `{"id":"docs","kind":"internet","url":"https://example.com/docs"}` |
| `agent` | Knowledge captured from agent interactions (e.g., Codex, Claude) | `{"id":"agent","kind":"agent"}` |

Sources with kind `agent` or `internet` are used for drafting proposals but are **not indexed** into the answer corpus. Only destinations (curated knowledge bases) are indexed for answering questions.

### Connecting a Git Source

1. Set `MAGPIE_CHECKOUT_ROOT` to a writable directory (e.g., `.magpie/checkouts`).
2. Add the source to `KNOWLEDGE_SOURCES`:
   ```json
   [{"id":"my-repo","url":"https://github.com/myorg/mydocs.git","subpath":"docs"}]
   ```
3. The API clones or fast‑forward pulls the repository on startup.
4. Index the associated flow via `POST /api/knowledge/repositories/index`.

### Connecting a Local Source

For local development, a source path inside the repository or an absolute path can be used. The example `cats` knowledge base is a local folder `knowledge-bases/cats`.

### Connecting an Internet Source

Internet sources are fetched on demand during indexing. The source URL must return valid Markdown. This is useful for public documentation that changes infrequently.

## AI Provider Integrations

Markdown Magpie can generate answers and draft proposals using any of the following providers, configured via `AI_PROVIDER` and related environment variables.

| Provider | Environment Variables | Notes |
|----------|----------------------|-------|
| `mock` | None (default) | Deterministic, no API key needed. |
| `openai-compatible` | `OPENAI_COMPATIBLE_BASE_URL`, `OPENAI_COMPATIBLE_API_KEY`, `OPENAI_COMPATIBLE_MODEL` | Works with OpenAI, DeepSeek, OpenRouter, etc. |
| `azure-openai` | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_CHAT_DEPLOYMENT`, `AZURE_OPENAI_API_VERSION` | Azure OpenAI chat completions. |
| `codex` | `CODEX_CLI_PATH`, `CODEX_CLI_ARGS`, `CODEX_CLI_PROMPT_MODE` | External agent via CLI. |
| `claude` | `CLAUDE_CLI_PATH`, `CLAUDE_CLI_ARGS`, `CLAUDE_CLI_PROMPT_MODE` | External agent via CLI. |

**Execution Modes**

- `direct`: The API calls the provider synchronously (recommended for development).
- `queue`: Jobs are enqueued for a watcher process (useful for long‑running external agents).

Switch modes at runtime via `POST /api/config` or by setting `AI_EXECUTION_MODE` in your `.env` file.

### Embedding Providers

Embeddings are configured independently of chat providers. Set one of the following groups to enable hybrid retrieval (keyword + vector search):

| Group | Variables | Vector Dimension |
|-------|-----------|------------------|
| OpenAI‑compatible | `OPENAI_COMPATIBLE_EMBEDDING_BASE_URL` (falls back to chat endpoint), `OPENAI_COMPATIBLE_EMBEDDING_API_KEY`, `OPENAI_COMPATIBLE_EMBEDDING_MODEL` | 1536 |
| Azure OpenAI | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_EMBEDDING_DEPLOYMENT` | 1536 |

Embeddings require `KNOWLEDGE_STORE=postgres` and a live `DATABASE_URL`.

## Storage Backend Integrations

| Backend | Environment Variables | Use Case |
|---------|----------------------|----------|
| Postgres (pgvector) | `DATABASE_URL`, `STORAGE_BACKEND=postgres` | Primary production storage; enables vector search. |
| Redis | Optional – set `QUEUE_URL` if needed | Job queue backend (currently postgres is used by default). |
| In‑memory | `STORAGE_BACKEND=memory` (fallback) | Local development without Postgres. |

Postgres is the recommended and default backend for all deployments.

## Version Control Integrations (Pull Requests)

Once a proposal is drafted, it can be published to a Git branch and raised as a pull request on a hosted provider. The system uses `LocalGitProposalPublisher` to commit to a local branch, then optionally pushes the branch and creates a PR when a host token is configured.

| Provider | Environment Variables |
|----------|----------------------|
| GitHub | `GITHUB_TOKEN` |
| GitLab | `GITLAB_TOKEN` (planned) |
| Azure DevOps | `AZURE_DEVOPS_TOKEN` (planned) |

If no token is set, proposals degrade gracefully to a pushed branch.

## Deployment Integrations

| Target | Configuration | Notes |
|--------|---------------|-------|
| Docker Compose | Provided `docker-compose.yml` with `app` profile | Single‑host deployment for demos and small production setups. |
| Azure Container Apps | Azure deployment notes in `infra/azure/` | Optional managed deployment; requires PostgreSQL, Redis, Blob Storage, etc. |
| Local (npm) | `npm run dev:api`, `npm run dev:web` | Development loop with host‑based processes + Docker for Postgres/Redis. |

The default production shape is Docker Compose. Azure is the preferred managed cloud provider but the product remains portable to other clouds.

## Identity Integration (Authentication)

| Provider | Environment Variables |
|----------|----------------------|
| Auth0 | `AUTH0_ISSUER_BASE_URL`, `AUTH0_DOMAIN`, `AUTH0_AUDIENCE`, `AUTH0_JWKS_URI` | MCP HTTP transport uses OAuth 2.0; web and API use bearer tokens. |
| None (unauthenticated) | `AUTH_REQUIRED=false` (default) | Local development only. |

Authentication is optional and disabled by default. The MCP server and web UI can be gated behind Auth0 tokens.

## MCP Client Integration

The MCP server (`apps/mcp`) allows AI agents (Claude Code, Codex, etc.) to query the knowledge base and submit feedback. It supports two transports:

- **stdio**: The client launches the server as a subprocess (e.g., `.mcp.json` in a Claude Code project).
- **Streamable HTTP**: A persistent HTTP server on port 4001, supporting OAuth authentication.

Both transports proxy all requests to the Markdown Magpie API. See [MCP Server](managing-knowledge-flows-in-markdown-magpie.md#mcp-integration) for full configuration.

## External Agent Providers (Watcher)

The watcher can use a local CLI tool as the AI provider, enabling integration with Codex, Claude Code, or any command-line agent.

| Provider | Environment Variables |
|---|---|
| `codex` | `AI_PROVIDER=codex`, `CODEX_CLI_PATH`, `CODEX_CLI_ARGS`, `CODEX_CLI_PROMPT_MODE` (arg or stdin). |
| `claude` | `AI_PROVIDER=claude`, `CLAUDE_CLI_PATH`, `CLAUDE_CLI_ARGS`, `CLAUDE_CLI_PROMPT_MODE`. |
| `openai-compatible` | Same as chat provider but for watcher process; uses `AI_PROVIDER=openai-compatible`. |
| `mock` | Deterministic mock provider for testing. |

The agent must return JSON matching the job output schema. The watcher validates and posts results back to the API.

## Step‑by‑Step: Connect a New Data Source

1. **Set environment variables** in `.env`:

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts
KNOWLEDGE_SOURCES=[{"id":"my-source","url":"https://github.com/org/docs.git","subpath":"articles"}]
KNOWLEDGE_DESTINATIONS=[{"id":"my-dest","url":"https://github.com/org/mykb.git","subpath":"docs"}]
KNOWLEDGE_FLOWS=[{"id":"myflow","sourceIds":["my-source"],"destinationId":"my-dest"}]
STORAGE_BACKEND=postgres
DATABASE_URL=postgres://...
AI_PROVIDER=mock
```

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

## Summary

Markdown Magpie integrates with:

- **Data sources**: local, git, internet, agent
- **AI providers**: mock, OpenAI‑compatible, Azure OpenAI, external CLI agents (Codex, Claude)
- **Embedding providers**: OpenAI‑compatible, Azure OpenAI
- **Storage**: Postgres (recommended), Redis, in‑memory
- **Version control**: Git, GitHub (PR support), GitLab/Azure DevOps (planned)
- **Deployment**: Docker Compose, Azure (optional)
- **Identity**: Auth0 (optional)
- **MCP clients**: Claude Code, Codex, any MCP‑aware tool

Each integration is provider‑neutral behind defined interfaces, so you can mix and match components without modifying core logic.

---

*Based on source material from: `README.md`, `docs/ingestion.md`, `docs/chat-providers.md`, `docs/architecture.md`, `docs/ai-jobs.md`, `docs/mcp.md`, `infra/azure/README.md`.*
