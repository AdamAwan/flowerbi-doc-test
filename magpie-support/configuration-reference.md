---
title: Configuration Reference
owner: magpie-ops
status: draft
tags: [configuration, reference, env]
review_cycle_days: 90
---

# Configuration Reference

This document describes all environment variables and configuration options for Markdown Magpie. It is the single source of truth for settings related to AI providers, embeddings, storage, knowledge sources/destinations, authentication, and deployment.

## Knowledge Sources, Destinations, and Flows

Configure the flow pipeline via these environment variables:

| Variable | Description | Example |
|---|---|---|
| `KNOWLEDGE_SOURCES` | JSON array of source objects. Each source must have at least `id`, `name`, and `kind`. | `[{"id":"my-repo","kind":"git","url":"...","subpath":"docs"}]` |
| `KNOWLEDGE_DESTINATIONS` | JSON array of destination objects (writable Git repos). | `[{"id":"my-dest","name":"My KB","url":"...","subpath":"docs"}]` |
| `KNOWLEDGE_FLOWS` | JSON array of flow objects linking source IDs to a destination ID. | `[{"id":"myflow","sourceIds":["my-repo"],"destinationId":"my-dest"}]` |
| `MAGPIE_CHECKOUT_ROOT` | Local directory for cloned repositories. | `.magpie/checkouts` |
| `KNOWLEDGE_STORE` | Backend for indexed knowledge. Either `postgres` (recommended) or `memory`. | `postgres` |
| `DATABASE_URL` | Postgres connection string (required when `KNOWLEDGE_STORE=postgres`). | `postgres://user:pass@localhost:5432/magpie` |

For backward compatibility, `KNOWLEDGE_REPOSITORIES` and `KNOWLEDGE_REPO_PATH` are still supported when the new variables are not set.

### Supported Source Kinds

| Kind | Configuration | Example | Notes |
|------|---------------|---------|-------|
| `local` | A folder on the server (e.g., `knowledge-bases/cats`). | `{"id":"docs","kind":"local","path":"knowledge-bases/product"}` | – |
| `git` | A remote Git repository, optionally with `subpath`. | `{"id":"flowerbi","url":"https://github.com/example/flowerbi.git","subpath":"src"}` | – |
| `internet` | A public URL returning Markdown (fetched at index time). | `{"id":"docs","kind":"internet","url":"https://example.com/docs"}` | Used for drafting proposals only; **not indexed** into the answer corpus. |
| `agent` | Knowledge captured from agent interactions (e.g., Codex, Claude). | `{"id":"agent","kind":"agent"}` | Used for drafting proposals only; **not indexed** for answering. |

Only destinations (curated knowledge bases) are indexed for answering questions.

### Further Examples

#### Two Git Sources and One Destination

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts
KNOWLEDGE_SOURCES=[{"id":"flowerbi","name":"FlowerBI Source","url":"https://github.com/danielearwicker/flowerbi.git","subpath":"src"},{"id":"agent","name":"Agent Knowledge","kind":"agent"}]
KNOWLEDGE_DESTINATIONS=[{"id":"flowerbi-docs","name":"FlowerBI Docs","url":"https://github.com/AdamAwan/flowerbi-doc-test.git","subpath":"docs"}]
KNOWLEDGE_FLOWS=[{"id":"flowerbi","name":"FlowerBI KB","sourceIds":["flowerbi","agent"],"destinationId":"flowerbi-docs"}]
```

#### Local Folder Source

```env
KNOWLEDGE_SOURCES=[{"id":"docs","name":"Product Docs","path":"knowledge-bases/product"}]
KNOWLEDGE_DESTINATIONS=[{"id":"docs","name":"Product Docs","path":"knowledge-bases/product"}]
KNOWLEDGE_FLOWS=[{"id":"docs","name":"Product Docs","sourceIds":["docs"],"destinationId":"docs"}]
```

## AI Provider Configuration

Set `AI_PROVIDER` to one of the following and configure the corresponding variables:

| Provider | `AI_PROVIDER` value | Required Variables | Notes |
|---|---|---|---|
| OpenAI‑compatible | `openai-compatible` | `OPENAI_COMPATIBLE_BASE_URL`, `OPENAI_COMPATIBLE_API_KEY`, `OPENAI_COMPATIBLE_MODEL` | |
| Azure OpenAI | `azure-openai` | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_CHAT_DEPLOYMENT`, `AZURE_OPENAI_API_VERSION` | |
| Codex (CLI) | `codex` | `CODEX_CLI_PATH`, `CODEX_CLI_ARGS`, `CODEX_CLI_PROMPT_MODE` | `CODEX_CLI_PATH` defaults to `codex` on `PATH` if unset. |
| Claude (CLI) | `claude` | `CLAUDE_CLI_PATH`, `CLAUDE_CLI_ARGS`, `CLAUDE_CLI_PROMPT_MODE` | `CLAUDE_CLI_PATH` defaults to `claude` on `PATH` if unset. |

**Note:** The `mock` provider has been removed. `AI_PROVIDER` must now name a supported provider from the table above.

## Embedding Provider Configuration

Embeddings are configured independently of the chat provider. Set one of the following groups to enable hybrid retrieval (keyword + vector):

**OpenAI‑compatible:**
```env
OPENAI_COMPATIBLE_EMBEDDING_BASE_URL=https://api.openai.com/v1
OPENAI_COMPATIBLE_EMBEDDING_API_KEY=sk-...
OPENAI_COMPATIBLE_EMBEDDING_MODEL=text-embedding-3-small
```

**Azure OpenAI:**
```env
AZURE_OPENAI_ENDPOINT=https://myinstance.openai.azure.com
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=text-embedding-3-small
```

Embeddings require `KNOWLEDGE_STORE=postgres` and a live `DATABASE_URL`. The model must output 1536‑dimensional vectors.

Note: `EMBEDDING_PROVIDER` is an informational variable only — it is surfaced in `/api/config` for display and does not enable embeddings. Setting embedding credentials is what activates them.

## Job Queue Configuration

All AI work is enqueued to a pg‑boss queue in Postgres. The API never calls a model inline; a separate watcher claims and completes jobs. The following variables control job queue behaviour:

| Variable | Description | Default |
|---|---|---|
| `JOB_WAIT_TIMEOUT_MS` | Maximum time (ms) for a single long‑poll wait call on a job. | 25000 |
| `JOB_WAIT_POLL_MS` | Server‑side poll interval (ms) when waiting for a job to complete. | 250 |
| `JOB_SCHEDULE_TIMEZONE` | Timezone for scheduled jobs (e.g., crunch). | `UTC` |

Set `QUEUE_URL` to configure Redis (optional) for the job queue; otherwise pg‑boss uses Postgres as its queue backend.

## Storage Backend Configuration

| Backend | Variable | Value |
|---|---|---|
| Postgres (default) | `STORAGE_BACKEND` | `postgres` |
| In‑memory (fallback) | `STORAGE_BACKEND` | `memory` |

For Postgres, also set `DATABASE_URL` as above. Redis is optional and used only for job queuing (set `QUEUE_URL` if needed).

## Version Control Integration (Pull Requests)

| Provider | Required Variable |
|---|---|
| GitHub | `GITHUB_TOKEN`, `MAGPIE_GIT_AUTHOR_NAME`, `MAGPIE_GIT_AUTHOR_EMAIL` |
| GitLab (planned) | `GITLAB_TOKEN` |
| Azure DevOps (planned) | `AZURE_DEVOPS_TOKEN` |

`MAGPIE_GIT_AUTHOR_NAME` and `MAGPIE_GIT_AUTHOR_EMAIL` are used as the Git author identity when creating commits and pull requests. If no token is set, proposals are pushed to a branch but no PR is created.

### Source Change Sync & Git Clone

| Variable | Description | Default |
|---|---|---|
| `SOURCE_SYNC_MAX_CHANGED_FILES` | Maximum number of changed files to materialize downstream (into retrieval and model) when a commit touches many files. The true total is still recorded on the run. | 1000 |
| `GIT_PARTIAL_CLONE` | Set to `0`, `false`, or `off` to disable blobless partial cloning. Partial clones defer historical file blobs, reducing initial clone time. | `true` (enabled) |

## Authentication (Auth0 / Entra ID)

Authentication **fails closed**: it is required by default unless explicitly disabled with `AUTH_REQUIRED=false`. When enabled, configure one of the following:

**Auth0:**
```env
AUTH0_ISSUER_BASE_URL=https://your-tenant.eu.auth0.com
AUTH0_AUDIENCE=https://markdown-magpie.local/api
```
Alternatively, `AUTH0_DOMAIN` can be used instead of `AUTH0_ISSUER_BASE_URL`.

**Microsoft Entra ID:**
- Use the standard Azure environment variables (see `infra/azure/README.md`).

**MCP‑specific tokens:**
- `MCP_AUTH_TOKEN`: Token used by the MCP stdio server to authenticate to the API.
- `MCP_API_AUTH_TOKEN`: Token used by the MCP HTTP server for downstream API calls.

Both are required when auth is enabled (the default, unless `AUTH_REQUIRED=false`) and the MCP server is active.

**Watcher M2M tokens:**
- `WATCHER_API_CLIENT_ID`: Client ID used by the watcher to authenticate to the API when auth is enabled. Must be set together with `WATCHER_API_CLIENT_SECRET`.
- `WATCHER_API_CLIENT_SECRET`: Client secret for the watcher M2M token. Must be set together with `WATCHER_API_CLIENT_ID`.
- `API_TOKEN`: (Legacy) Static token used by the watcher as an alternative to the client-credentials pair.

At least one of the following must be set when auth is enabled (the default) and the watcher is active:
- Both `WATCHER_API_CLIENT_ID` and `WATCHER_API_CLIENT_SECRET` (preferred), or
- `API_TOKEN` (legacy fallback).

## CORS Configuration

| Variable | Description | Default |
|---|---|---|
| `CORS_ALLOWED_ORIGINS` | Comma-separated list of allowed origins for the Access-Control-Allow-Origin header. Unset or `"*"` allows any origin. Set to a comma-separated list to restrict. | `*` |

## MCP Server Configuration

The MCP server (`apps/mcp`) supports two transports: `stdio` (launched as subprocess) and `streamable-http` (persistent HTTP on port 4001). Configure via:

```env
MCP_TRANSPORT=stdio  # or streamable-http
MCP_PORT=4001        # for HTTP transport
```

When HTTP transport is used, per‑tool OAuth scopes are enforced:

| Tool | Scope |
|---|---|
| `kb.search` | `read:knowledge` |
| `kb.ask` | `ask:knowledge` |
| `kb.feedback` | `feedback:questions` |

## Deployment Configuration

| Target | Key Variables |
|---|---|
| Docker Compose | Use `docker compose up`; application containers require `profile=app`. |
| Azure Container Apps | See `infra/azure/README.md`. |
| Local (npm) | `npm run dev:api` and `npm run dev:web` with Docker for Postgres/Redis. For web development, set `MAGPIE_DEV_API_PROXY=http://localhost:4000` so the web dev server proxies API requests. |

The default production shape is Docker Compose.

---
*This reference consolidates content from the Getting Started, Integrations, Flows, and Permissions documents. For detailed guides and step-by-step instructions, see [Managing Knowledge Flows](managing-knowledge-flows-in-markdown-magpie.md), [Getting Started](getting-started-onboarding-and-indexing-content-into-markdow.md), and [Integrations and Connecting Data Sources](integrations-and-connecting-data-sources.md).*