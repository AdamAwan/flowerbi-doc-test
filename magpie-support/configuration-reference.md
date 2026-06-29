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

## AI Provider Configuration

Set `AI_PROVIDER` to one of the following and configure the corresponding variables:

| Provider | `AI_PROVIDER` value | Required Variables | Notes |
|---|---|---|---|
| Mock (deterministic) | `mock` | None | |
| OpenAI‑compatible | `openai-compatible` | `OPENAI_COMPATIBLE_BASE_URL`, `OPENAI_COMPATIBLE_API_KEY`, `OPENAI_COMPATIBLE_MODEL` | |
| Azure OpenAI | `azure-openai` | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_CHAT_DEPLOYMENT`, `AZURE_OPENAI_API_VERSION` | |
| Codex (CLI) | `codex` | `CODEX_CLI_PATH`, `CODEX_CLI_ARGS`, `CODEX_CLI_PROMPT_MODE` | `CODEX_CLI_PATH` defaults to `codex` on `PATH` if unset. |
| Claude (CLI) | `claude` | `CLAUDE_CLI_PATH`, `CLAUDE_CLI_ARGS`, `CLAUDE_CLI_PROMPT_MODE` | `CLAUDE_CLI_PATH` defaults to `claude` on `PATH` if unset. |

**Execution Modes**

Set `AI_EXECUTION_MODE` to `direct` (synchronous) or `queue` (async with watcher). Default: `direct`.

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
| `JOB_WAIT_TIMEOUT_MS` | Maximum time (ms) for a single long‑poll wait call on a job. | (not set) |
| `JOB_WAIT_POLL_MS` | Server‑side poll interval (ms) when waiting for a job to complete. | (not set) |
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
|---|---|---|
| GitHub | `GITHUB_TOKEN`, `MAGPIE_GIT_AUTHOR_NAME`, `MAGPIE_GIT_AUTHOR_EMAIL` |
| GitLab (planned) | `GITLAB_TOKEN` |
| Azure DevOps (planned) | `AZURE_DEVOPS_TOKEN` |

`MAGPIE_GIT_AUTHOR_NAME` and `MAGPIE_GIT_AUTHOR_EMAIL` are used as the Git author identity when creating commits and pull requests. If no token is set, proposals are pushed to a branch but no PR is created.

## Authentication (Auth0 / Entra ID)

For a comprehensive guide covering authentication setup, per-tool MCP scopes, web console auth, and future directions, see [Permissions and Access Controls in Markdown Magpie](permissions-and-access-controls-in-markdown-magpie.md).

Set `AUTH_REQUIRED=true` to enable authentication.

Key variables: `AUTH_REQUIRED`, `AUTH0_ISSUER_BASE_URL`, `AUTH0_DOMAIN`, `AUTH0_AUDIENCE`, `MCP_AUTH_TOKEN`, `MCP_API_AUTH_TOKEN`. See the Permissions document for their complete descriptions.

Set `AUTH_REQUIRED=true` to enable authentication. Then configure one of the following:

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

Both are required when `AUTH_REQUIRED=true` and the MCP server is active.

### Web Console Authentication

The web administration console (port 3000) validates Auth0-issued tokens via the `@auth0/auth0-react` SDK when authentication is enabled. It requires the following environment variables:

| Variable | Purpose |
|----------|---------|
| `NEXT_PUBLIC_AUTH0_ISSUER` | Auth0 issuer URL for the browser-side SDK. |
| `NEXT_PUBLIC_AUTH0_CLIENT_ID` | Auth0 client ID for the web application. |
| `NEXT_PUBLIC_AUTH0_AUDIENCE` | Expected API audience (must match `AUTH0_AUDIENCE`). |

When `AUTH_REQUIRED=false`, the web console skips authentication and operates without login. In production deployments, set these variables to match the Auth0 application registered for the web console.

### Auth Variables Summary

All authentication variables are shared between the API and MCP server via the `@magpie/auth` package:

| Variable | Default | Purpose |
|----------|---------|---------|
| `AUTH_REQUIRED` | `false` | Enable authentication. |
| `AUTH0_ISSUER_BASE_URL` | – | Full Auth0 issuer (e.g., `https://your-tenant.eu.auth0.com`). |
| `AUTH0_DOMAIN` | – | Alternative to issuer base; issuer becomes `https://<domain>/`. |
| `AUTH0_AUDIENCE` | `https://markdown-magpie.local/api` | API identifier the token must carry. |
| `AUTH0_JWKS_URI` | Derived from issuer | JWKS endpoint for token validation. |
| `MCP_AUTH_TOKEN` | – | stdio only: bearer token presented to the API. Required when `AUTH_REQUIRED=true`. |
| `MCP_API_AUTH_TOKEN` | – | HTTP only: service token for API calls. Required when `AUTH_REQUIRED=true`. |

### Configuration Examples

#### Enable Simple Auth (API + MCP stdio)

```env
AUTH_REQUIRED=true
AUTH0_ISSUER_BASE_URL=https://markdown-magpie.auth0.com
AUTH0_AUDIENCE=https://markdown-magpie.local/api
MCP_AUTH_TOKEN=eyJhbGci...
```

#### Enable Per‑Tool Scopes (MCP HTTP)

Add the MCP HTTP server service token and ensure the client tokens contain the appropriate scopes. The HTTP server’s own token is used for API calls and does not need scopes.

```env
AUTH_REQUIRED=true
MCP_API_AUTH_TOKEN=eyJhbGci...
# Also set Auth0 issuer and audience as above.
```

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

For a detailed explanation of the permission model and future scope additions, see [Permissions and Access Controls](permissions-and-access-controls-in-markdown-magpie.md).

## Future Directions

- **Role‑based API access** – The API will enforce role‑based checks (e.g., admin, editor, viewer) on endpoints such as proposal creation, gap management, and system configuration.
- **Team and organization support** – Integration with Microsoft Entra ID or Auth0 organizations will allow scoping permissions to specific teams.
- **Audit logging** – All authentication decisions and permission checks will be logged for compliance and troubleshooting.

## Deployment Configuration

| Target | Key Variables |
|---|---|
| Docker Compose | Use `docker compose up`; application containers require `profile=app`. |
| Azure Container Apps | See `infra/azure/README.md`. |
| Local (npm) | `npm run dev:api` and `npm run dev:web` with Docker for Postgres/Redis. |

The default production shape is Docker Compose.

---
*This reference consolidates content from the Getting Started, Integrations, Flows, and Permissions documents.*
