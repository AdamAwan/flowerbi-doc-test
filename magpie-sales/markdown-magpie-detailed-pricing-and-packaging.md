---
title: Markdown Magpie — Detailed Pricing and Packaging
status: draft
---

# Markdown Magpie — Detailed Pricing and Packaging

Markdown Magpie is open-source software (MIT License) that organisations deploy on their own infrastructure. There is no managed cloud or SaaS offering. The costs an operator bears fall into two categories: infrastructure (compute, storage, networking) and AI provider usage (model API calls). The application exposes configuration knobs and built-in controls to monitor and constrain both.

## Licensing and Distribution

Magpie is released under the MIT License and distributed as a single Docker container image published to the GitHub Container Registry (ghcr.io). The image is built by a CI workflow (Publish container image) that runs on every merge to main and on version tags. A local development environment requires Node.js 22+, npm, and a Postgres database with the pgvector extension.

## Deployment Architecture

All components ship in the same container image and are selected by startup command. A full deployment consists of:

- **API** — HTTP server (default port 4000). Owns the database and the pg-boss job queue. Enqueues AI work but never calls a chat model inline.
- **Web** — Next.js review and administration console (default port 3000).
- **Watcher** — Claims AI jobs from the queue, calls the configured AI provider, posts results back over HTTP. Has no direct database access. Exposes a health endpoint on port 4002 but accepts no inbound application traffic.
- **MCP** — Two transport modes: stdio and Streamable HTTP (default port 4001). Provides a programmatic surface over the API for agent clients.
- **Postgres** — Required datastore (pgvector/pg16 image). Stores application data, indexed knowledge, embeddings, and the job queue.

An optional logging stack (Loki + Alloy + Grafana) can be added via a Docker Compose profile; it is not part of the core application and operators may substitute their own log backend.

### Operational Requirements

- **Compute**: Each component runs as a separate process. In production the API, web, watcher, and MCP are typically deployed together on one host or across a small cluster.
- **Storage**: Postgres for all persistent state. The API and watcher each maintain their own Git checkout volumes (separate volumes prevent lock contention). Snapshot storage is API-only.
- **Networking**: The API serves on port 4000, the web console on port 3000, and the MCP HTTP server on port 4001. All services are designed to sit behind a reverse proxy that terminates TLS.
- **Git hosting**: Proposals are pushed as branches and opened as pull requests on GitHub or pushed to local-git (`file://`) destinations. A GitHub token or Azure DevOps PAT is needed for hosted PR operations.

### Security Hardening

The container runs as a non-root user (uid 1001) and installs only production dependencies. The Dockerfile uses a multi-stage build. The API and web emit standard security headers. Authentication fails closed — `AUTH_REQUIRED` defaults to on, and an invalid value keeps auth enabled. When auth is required, the API refuses to start unless a real Auth0 audience is configured.

## Cost Model

Magpie itself costs nothing to license, but operators pay for:

### Infrastructure Costs

- **Postgres database**: A managed or self-hosted Postgres instance with pgvector. Storage grows with the size of the indexed knowledge base and the number of question logs, proposals, and job records. Job records self-purge (retention window).
- **Compute**: The API, web, watcher, and optionally MCP processes. The watcher runs one job at a time and can be deployed as a separate instance to match workload.
- **Optional logging**: The example Loki + Alloy + Grafana stack adds storage and compute costs if adopted.

### AI Provider Costs

The operator chooses and pays for their own AI provider. Magpie is provider-neutral and supports four types:

| Provider | Credential variables | Self-hostable |
|---|---|---|
| OpenAI-compatible | `OPENAI_COMPATIBLE_BASE_URL`, `OPENAI_COMPATIBLE_API_KEY`, `OPENAI_COMPATIBLE_MODEL` | Yes — point at vLLM, Ollama, or any OpenAI-compatible endpoint |
| Azure OpenAI | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_CHAT_DEPLOYMENT` | Private cloud via Azure agreement |
| Claude (Anthropic) | `CLAUDE_CLI_PATH` | No — public API |
| Codex | `CODEX_CLI_PATH` | Depends on the CLI's own backend |

A key architectural property keeps AI costs bounded: **the API never calls a chat model inline**. All generative work runs as jobs on a pg-boss queue, claimed and completed by the watcher. This means the API cannot be used to drive unbounded AI spend directly — every generative call is gated by the job queue and the capacity controls below.

### AI Cost Tracking (AI_PRICING)

Operators can supply a token-price table via the `AI_PRICING` environment variable — a JSON array of entries, each specifying a `provider`, `model`, `inputPerMTok`, and `outputPerMTok` (per million tokens, in the operator's billing currency). This is used at read time by the Insights charts to estimate monetary cost from stored token usage. Costs are never persisted; a corrected price entry retroactively re-values history. When no entry matches a (provider, model) triple, the usage appears as unpriced rather than zero, keeping the three states distinct: priced, unpriced, and unmetered.

Usage is reported by the watcher on job completion as part of the completion envelope. Each job carries the provider and model that executed it, plus the summed token counts (input, output, total) reported by the provider. CLI providers emit raw text and report no usage, so their completions are unmetered.

## Built-in Cost Controls

### L1 — Per-Principal Request Rate Limiting

A fixed-window rate limiter keys on the authenticated principal and applies two tiers:

| Tier | Endpoints | Default limit per window |
|---|---|---|
| Ask | `POST /api/ask`, `POST /api/retrieve` | 30 requests per 60-second window |
| Trigger | Source-sync, patrol runs, scheduled-task runs, repository index | 5 requests per 60-second window |

The rate limiter no-ops when authentication is disabled (local development), as there is no principal to key on.

### L2 — Global In-Flight AI Job Cap

Before enqueuing an `answer_question` job, the API checks a global ceiling on concurrent metered work. The cap is class-aware:

- **Interactive class** — jobs with a live caller waiting (`answer_question`, `outline_flow_seed`). A reserve (`AI_INTERACTIVE_RESERVED_JOBS`, default 5) guarantees that maintenance bursts cannot starve interactive requests.
- **Background class** — maintenance fan-out (patrol scans, drafting, gap summaries, etc.).

An interactive enqueue is rejected with `429 ai_capacity` only when BOTH the interactive reserve is fully occupied AND the global ceiling (`AI_MAX_INFLIGHT_JOBS`, default 20) is reached. This guarantees that maintenance work never pushes `POST /api/ask` into a 429.

| Env var | Default | Meaning |
|---|---|---|
| `RATE_LIMIT_ENABLED` | `true` | Master switch for L1 and L2 |
| `RATE_LIMIT_WINDOW_MS` | `60000` | Fixed-window width |
| `RATE_LIMIT_ASK_PER_WINDOW` | `30` | Ask-tier requests per principal per window |
| `RATE_LIMIT_TRIGGER_PER_WINDOW` | `5` | Trigger-tier requests per principal per window |
| `AI_MAX_INFLIGHT_JOBS` | `20` | Global ceiling on concurrent in-flight AI jobs |
| `AI_INTERACTIVE_RESERVED_JOBS` | `5` | Slots reserved for interactive AI jobs |

### Storage Backend Cost Control

The default storage backend is `memory`, which keeps the indexed knowledge in process memory. For production use, operators switch to `postgres` (via `STORAGE_BACKEND=postgres` or per-store overrides like `KNOWLEDGE_STORE=postgres`). Individual stores can be overridden independently, allowing operators to tune which state is persisted at Postgres cost.

## Observability and Spend Visibility

The Insights page provides several charts relevant to cost management:

- **C11 — AI Token Usage and Cost**: Stacked bar chart of tokens used per (job type, provider, model) triple over the window, priced at read time against the AI_PRICING table. Displays estimated monetary cost as a header total and per-bar tooltip.
- **AI Cost by Flow**: Per-flow breakdown of AI spend, so operators can see which knowledge area consumes the most model tokens.
- **Per-Schedule Cost**: An approximate cost column on the Schedules page, attributed to each scheduled maintenance task by its fan-out job types.

All cost figures are computed at read time from stored token counts and the current AI_PRICING table — correcting a mispriced entry retroactively re-values historical data.

## Deployment Sizing Considerations

- The watcher runs one job at a time. A deployment with heavy concurrent AI work needs at least two watchers: one to run the maintenance orchestration and a second to claim the AI jobs the orchestrator enqueues. The console warns when only one watcher is connected.
- AI jobs have configurable timeouts. The default agentic timeout (`MAGPIE_AGENTIC_TIMEOUT_MS`) is 600,000 ms (10 minutes), and the queue expiration is 900 seconds.
- Job records self-purge via pg-boss retention; the default retention is 30 days for completed jobs.
- The API's health endpoint (`GET /api/health`) and readiness endpoint (`GET /api/ready`) are public (auth-exempt) to support orchestrator probes.

## Comparison with Self-Hosted Alternatives

Positioning Magpie as self-hosted, open-source knowledge maintenance means the operator's total cost is the sum of their chosen infrastructure provider, their chosen AI provider, and the operational effort to configure and maintain the deployment. There are no per-seat, per-user, or subscription fees. The primary differentiator against building an equivalent system in-house is the pre-built pipeline: ingestion, retrieval, gap detection, clustering, proposal drafting, publication, and gap-closure verification — all configurable through environment variables and exposed through a web console, API, and MCP interface.