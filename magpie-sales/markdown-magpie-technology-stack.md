---
title: Markdown Magpie Technology Stack
status: draft
---

# Markdown Magpie Technology Stack

Markdown Magpie is built on modern JavaScript/TypeScript runtimes and a modular monorepo architecture. The following table summarises the core technologies used:

| Layer | Technology | Notes |
|-------|------------|-------|
| **Language** | TypeScript (Node.js 22+) | All packages and apps are written in TypeScript, compiled with `tsc`. The project targets ES modules (`"type": "module"`). |
| **Monorepo tool** | npm workspaces | The repository is organised as a set of npm workspaces under `packages/` and `apps/`. |
| **API framework** | [Hono](https://hono.dev/) | The HTTP API (`@magpie/api`) uses Hono 4.x with Zod validation and a plain Node server (`@hono/node-server`). |
| **Web console** | [Next.js](https://nextjs.org/) 16 (React 19) | The review and administration console (`@magpie/web`) is a Next.js application. |
| **Database** | PostgreSQL (with pgvector) | Persistent storage uses PostgreSQL. The pgvector extension powers hybrid (keyword + vector) search. Migrations are in `packages/db/migrations`. |
| **Job queue** | pg-boss | AI work and background tasks are managed via pg-boss, a Postgres-backed job queue. |
| **Git integration** | Custom Node.js adapters | Repositories are cloned and manipulated using the `@magpie/git` package, which shells out to Git via Node.js. |
| **MCP server** | Model Context Protocol (Node SDK) | The `@magpie/mcp` app provides a stdio and HTTP MCP server using the official MCP Node SDK. |
| **Containerisation** | Docker Compose | Local development and single-host deployments use Docker Compose to run the full stack (API, web, watcher, Postgres). |
| **AI providers** | OpenAI-compatible, Azure OpenAI, Codex, Claude | The `@magpie/watcher` runs provider adapters; the system is provider-neutral by design. See `docs/chat-providers.md`. |

## Repository Layout

The codebase is organised as a monorepo with the following workspaces:

- **`apps/api/`** — HTTP API and job queue owner (Hono, pg-boss).
- **`apps/web/`** — Next.js console.
- **`apps/watcher/`** — Worker that claims AI jobs and delegates to the configured provider.
- **`apps/mcp/`** — MCP server for agent clients.
- **`packages/core/`** — Shared domain types.
- **`packages/auth/`** — Auth0 token validation.
- **`packages/db/`** — Database schema and migrations.
- **`packages/git/`** — Git and pull request adapters.
- **`packages/jobs/`** — Job contracts and queue metadata.
- **`packages/markdown/`** — Markdown parsing and sectioning.
- **`packages/prompts/`** — Shared AI prompt catalog.
- **`packages/retrieval/`** — Search, embeddings, ranking, and answer orchestration.

## Development Requirements

- **Node.js** >= 22.12
- **npm** >= 10 (npm 11 may cause issues; use `npx --yes npm@10 ci` if needed)
- **Docker** (for Postgres or full-stack demos)

## Key Dependencies (from root `package.json`)

```json
{
  "engines": { "node": ">=22.12" },
  "devDependencies": {
    "typescript": "^6.0.3",
    "eslint": "^9.39.4",
    "prettier": "^3.8.4",
    "tsx": "^4.22.4"
  }
}
```

All application code is TypeScript, compiled via `tsx` for development and `tsc` for production builds.

## Sources

The information in this document is derived from the project’s source code:

- `README.md` — monorepo structure, requirements, and features.
- `package.json` (root) — workspaces, scripts, and engine constraints.
- `apps/api/package.json`, `apps/web/package.json`, `apps/watcher/package.json`, `apps/mcp/package.json` — per-app dependencies.
- `docs/architecture.md` — system design and provider strategy.
- `docs/chat-providers.md` — AI provider configuration.
- `docs/ai-jobs.md` — job contract and watcher model.