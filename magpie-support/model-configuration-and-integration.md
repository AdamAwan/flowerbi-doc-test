---
title: Model Configuration and Integration
status: draft
---

# Model Configuration and Integration

Markdown Magpie is provider-neutral: the API never calls a model inline. All generative work—answer synthesis, gap reconciliation, proposal drafting—runs in a background **watcher** process that claims jobs and invokes the configured provider. Embeddings are handled separately, also via a configurable endpoint.

This article covers how to configure the AI provider, embedding endpoint, and the integration endpoints (API, MCP, watcher) that clients and agents use to interact with the system.

## Overview

The system distinguishes two kinds of AI work:

- **Chat/answer work**: routed by `AI_PROVIDER` to one of `openai-compatible`, `azure-openai`, `codex`, or `claude`.
- **Embedding work**: configured independently via `OPENAI_COMPATIBLE_EMBEDDING_MODEL` (or the Azure embedding endpoint). Embeddings are produced by the API process synchronously at query time, not by the watcher.

All generative work is enqueued as jobs to a pg-boss queue in Postgres. The watcher claims jobs, runs the provider adapter, and posts results back to the API. The watcher has no direct database access—it talks only to the HTTP API.

## Provider Configuration

### `AI_PROVIDER` – Mandatory

Set `AI_PROVIDER` in your `.env` to one of: `openai-compatible`, `azure-openai`, `codex`, `claude`. This selects which provider the watcher uses for chat completions.

Each provider requires a specific set of environment variables. The watcher advertises a **capability** only when those variables are present, and the API routes jobs to capabilities that running watchers actually offer.

### OpenAI-Compatible (`openai-compatible`)

```env
AI_PROVIDER=openai-compatible
OPENAI_COMPATIBLE_BASE_URL=https://api.openai.com/v1   # or any compatible endpoint
OPENAI_COMPATIBLE_API_KEY=sk-...
OPENAI_COMPATIBLE_MODEL=gpt-4o
```

The watcher advertises the `openai-compatible` capability when all three variables are set.

### Azure OpenAI (`azure-openai`)

```env
AI_PROVIDER=azure-openai
AZURE_OPENAI_ENDPOINT=https://example.openai.azure.com
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_CHAT_DEPLOYMENT=gpt-4o-deployment
AZURE_OPENAI_API_VERSION=2024-10-21
```

The watcher advertises `azure-openai` when `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, and `AZURE_OPENAI_CHAT_DEPLOYMENT` are set.

### Codex or Claude (`codex` / `claude`)

These run an external agent CLI (Codex or Claude Code). The watcher shells out to the binary.

```env
AI_PROVIDER=codex
CODEX_CLI_PATH=codex   # defaults to 'codex' on PATH

# or

AI_PROVIDER=claude
CLAUDE_CLI_PATH=claude # defaults to 'claude' on PATH
```

The watcher advertises `codex` when `CODEX_CLI_PATH` is set, and `claude` when `CLAUDE_CLI_PATH` is set.

### Switching Provider at Runtime

The active AI provider can be changed without restarting via the API:

```bash
POST /api/config
{
  "aiProvider": "claude"
}
```

This returns the updated config. The provider must be pre-configured in the environment; otherwise a `400` error is returned.

## Embedding Configuration

Embeddings are used for hybrid retrieval (keyword + vector). They are configured independently of the chat provider. You can use different providers for chat and embeddings (e.g., DeepSeek for chat, OpenAI for embeddings).

### OpenAI-Compatible Embeddings

```env
OPENAI_COMPATIBLE_EMBEDDING_BASE_URL=https://api.openai.com/v1   # optional; falls back to OPENAI_COMPATIBLE_BASE_URL
OPENAI_COMPATIBLE_EMBEDDING_API_KEY=...                         # optional; falls back to OPENAI_COMPATIBLE_API_KEY
OPENAI_COMPATIBLE_EMBEDDING_MODEL=text-embedding-3-small        # required; must output 1536-dimensional vectors
```

### Azure OpenAI Embeddings

```env
AZURE_OPENAI_ENDPOINT=https://example.openai.azure.com
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=text-embedding-3-small   # must output 1536-dimensional vectors
```

### Enabling Hybrid Retrieval

Hybrid retrieval (keyword + pgvector) activates automatically when:
- `KNOWLEDGE_STORE=postgres` is set with a valid `DATABASE_URL`.
- Embedding credentials are fully configured (either the OpenAI-compatible trio or the Azure trio above).

Check `GET /api/config` for the current `retrieval.mode` (`hybrid` or `keyword`) and a plain-language `reason`.

## Integration Endpoints

### API Base URL

The API listens on `PORT` (default `4000`) and serves all endpoints under `/api`.

Example for local development:
```
http://localhost:4000/api
```

Key endpoints for integration:

| Endpoint | Purpose |
|----------|---------|
| `POST /api/ask` | Submit a question; returns `202` with a job ID. |
| `GET /api/jobs/:id/wait` | Long-poll until a job completes. |
| `GET /api/questions/:id` | Retrieve the answer and citations. |
| `GET /api/knowledge/search?q=...` | Search indexed sections (supports hybrid). |
| `POST /api/knowledge/repositories/index` | Index a configured knowledge flow. |
| `GET /api/config` | Current runtime configuration (providers, retrieval mode, etc.). |
| `POST /api/config` | Switch the active AI provider at runtime. |
| `POST /api/proposals/from-gap` | Create a proposal draft from a gap cluster. |
| `POST /api/proposals/:id/publish` | Publish a proposal as a git branch and pull request. |

All endpoints accept and return JSON. CORS defaults to open (`access-control-allow-origin: *`) but can be restricted by setting `CORS_ALLOWED_ORIGINS` to a comma-separated list of allowed origins. Every response also includes standard security headers (`X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, `Referrer-Policy`, `Strict-Transport-Security`).

### MCP Server

The MCP server provides tools (`kb.ask`, `kb.search`, `kb.feedback`) for agent clients. It supports two transports:

- **stdio**: launched as a subprocess. Run:
  ```bash
  API_BASE_URL=http://localhost:4000 node apps/mcp/dist/main.js
  ```
- **streamable-http**: persistent HTTP on port `4001` (configurable via `MCP_PORT`). Run:
  ```bash
  MCP_TRANSPORT=streamable-http npm run dev:mcp
  ```

Configure via:
```env
MCP_TRANSPORT=stdio       # or streamable-http
MCP_PORT=4001
```

Clients can use the MCP tools to query the knowledge base and provide feedback, which feeds into the gap detection system.

### Watcher

The watcher is a separate process that claims and completes jobs. It must be running for any generative work (answers, proposals, crunch).

Run it locally:
```bash
npm run dev:watcher
```

The watcher reads the same `.env` file as the API. It advertises capabilities based on the environment variables present. Ensure `AUTH_REQUIRED=false` for local development, or configure Auth0 M2M credentials.

Available capabilities:

| Capability | Required env vars |
|------------|------------------|
| `openai-compatible` | `OPENAI_COMPATIBLE_BASE_URL` + `OPENAI_COMPATIBLE_API_KEY` + `OPENAI_COMPATIBLE_MODEL` |
| `azure-openai` | `AZURE_OPENAI_ENDPOINT` + `AZURE_OPENAI_API_KEY` + `AZURE_OPENAI_CHAT_DEPLOYMENT` |
| `codex` | `CODEX_CLI_PATH` (defaults to `codex`) |
| `claude` | `CLAUDE_CLI_PATH` (defaults to `claude`) |
| `maintenance` | (none; always available) |

The API only routes a job to a capability offered by a running watcher. A job stays queued until a capable watcher claims it.

## Capability Model

The watcher advertises its capabilities on startup. The API uses these to route jobs. This ensures provider-neutrality: you can run multiple watchers with different capabilities, and jobs will be picked up by whichever provider is configured.

To see which capabilities are ready, check the watcher startup logs:
```
[watcher] Capability openai-compatible — ready
[watcher] Capability maintenance — ready
```

## Runtime Configuration Endpoint

`GET /api/config` returns a full snapshot of the current configuration, including:
- `ai.provider` – the active chat provider.
- `ai.availableProviders` – list of configured providers.
- `retrieval.mode` – `hybrid` or `keyword` with a `reason`.
- `storage` – the active backend (postgres).
- `watcher` – watcher settings (if configured).

This endpoint is useful for troubleshooting and for clients that need to know the current system state.

## See Also

- [Chat Providers](chat-providers.md) — detailed provider configuration
- [AI Job Contract](ai-jobs.md) — watcher capabilities and job routing
- [Ingestion](ingestion.md) — embedding configuration and hybrid retrieval
- [Architecture](architecture.md) — provider strategy overview