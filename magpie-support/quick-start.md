---
title: Quick Start
owner: magpie-ops
status: draft
tags: [getting-started, quickstart]
review_cycle_days: 90
---

# Quick Start

This guide gets you up and running with Markdown Magpie in under 10 minutes using the mock AI provider and a local knowledge base. For detailed configuration of AI providers, embeddings, authentication, and more, see the [Configuration Reference](configuration-reference.md).

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

Copy `.env.example` to `.env` and edit to set at minimum:

```env
DATABASE_URL=postgres://postgres:postgres@localhost:5432/markdown_magpie
STORAGE_BACKEND=postgres
AI_EXECUTION_MODE=direct
AI_PROVIDER=mock
```

## 3. Start Dependencies (Postgres + Redis)

```bash
docker compose up -d
until [ "$(docker inspect -f '{{.State.Health.Status}}' "$(docker compose ps -q postgres)")" = healthy ]; do sleep 2; done
```

## 4. Run Migrations

```bash
npm run db:migrate
```

## 5. Configure a Knowledge Flow

Define a simple flow with a local source. In `.env` add:

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts
KNOWLEDGE_SOURCES=[{"id":"cats","name":"Cat Care Repo","kind":"local","path":"knowledge-bases/cats"}]
KNOWLEDGE_DESTINATIONS=[{"id":"cats-docs","name":"Cat Care Docs","url":"https://github.com/your-org/cats-docs.git","subpath":"docs"}]
KNOWLEDGE_FLOWS=[{"id":"cats","name":"Cat Care KB","sourceIds":["cats"],"destinationId":"cats-docs"}]
```

## 6. Start the API

```bash
mkdir -p .magpie/checkouts
MAGPIE_CHECKOUT_ROOT="$PWD/.magpie/checkouts" npm run dev:api &
```

The API is available at `http://localhost:4000`. Verify:
```bash
curl localhost:4000/api/health
```

## 7. Index Your Content

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"cats"}'
```

## 8. Ask a Question

```bash
curl -s http://localhost:4000/api/ask \
  -H 'content-type: application/json' \
  -d '{"question":"How should I introduce a new cat food?"}'
```

## 9. (Optional) Start the Web Console

```bash
MAGPIE_DEV_API_PROXY="http://localhost:4000" npm run dev:web
```

Open `http://localhost:3000` to browse the knowledge base.

## Troubleshooting

| Problem | Solution |
|---|---|
| `curl localhost:4000/api/health` fails | Check API terminal; kill other processes on port 4000. |
| Indexing returns `400 configured_repository_required` | Provide a valid `flowId` defined in `KNOWLEDGE_FLOWS`. |
| `/ask` returns low confidence | Verify indexing; wait for background embedding to finish (if configured). |

For more troubleshooting, see the [Managing Knowledge Flows](managing-knowledge-flows-in-markdown-magpie.md) guide.

## Next Steps

- Learn about [knowledge gap detection and proposals](managing-knowledge-flows-in-markdown-magpie.md#the-gap-pipeline-and-flows).
- Set up [real AI providers](configuration-reference.md#ai-provider-configuration).
- Configure [hybrid retrieval with embeddings](configuration-reference.md#embedding-provider-configuration).
- Review [permissions and access controls](permissions-and-access-controls-in-markdown-magpie.md).

---
*Based on the Getting Started guide. For detailed configuration, see the Configuration Reference.*
