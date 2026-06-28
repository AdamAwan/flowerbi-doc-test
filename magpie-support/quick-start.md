---
title: Quick Start
owner: magpie-ops
status: draft
tags: [getting-started, quickstart]
review_cycle_days: 90
---

> **Note:** This guide has been consolidated into the [Getting Started: Onboarding and Indexing Content into Markdown Magpie](getting-started-onboarding-and-indexing-content-into-markdow.md) document. The following content is retained for reference; please refer to the Getting Started guide for the most current instructions.

# Quick Start

This guide gets you up and running with Markdown Magpie in under 10 minutes using the mock AI provider and a local knowledge base. For detailed configuration of AI providers, embeddings, authentication, and more, see the [Configuration Reference](configuration-reference.md).

> **Important:** Markdown Magpie uses a queue-only architecture. The API never calls an AI model directly — it enqueues jobs that a separate **watcher** process claims and completes. This guide includes starting the watcher in a dedicated step. Without it, questions will stay queued and never be answered.

## Prerequisites

- Node.js 22+ and npm 10 (if npm 11 fails, use `npx --yes npm@10 ci`).
- Docker and Docker Compose (for Postgres).
- A Git repository with Markdown files you want to manage.

> **Note:** Redis is **not required** for local development. The queue uses Postgres via pg-boss. The `QUEUE_URL` variable in `.env.example` is legacy and can be left blank.

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
AI_PROVIDER=mock
AUTH_REQUIRED=false
```

> `AI_PROVIDER=mock` enables the built-in mock provider — no API keys needed. `AUTH_REQUIRED=false` turns off authentication so the API and watcher can communicate without Auth0 credentials.

## 3. Start Dependencies (Postgres)

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

> The `knowledge-bases` directory is intentionally empty by default. If using a local source, create the folder and add Markdown files before indexing. A `MAGPIE_CHECKOUT_ROOT` override prevents bootstrap failures — see the [Configuration Reference](configuration-reference.md) for details.

## 6. Start the API

```bash
mkdir -p .magpie/checkouts
MAGPIE_CHECKOUT_ROOT="$PWD/.magpie/checkouts" npm run dev:api &
```

The API is available at `http://localhost:4000`. Verify:
```bash
curl localhost:4000/api/health
```

## 7. Start the Watcher (Required)

Start the watcher in a separate terminal so it can claim and complete AI jobs:

```bash
AUTH_REQUIRED=false WATCHER_API_CLIENT_ID= WATCHER_API_CLIENT_SECRET= \
  MAGPIE_CHECKOUT_ROOT="$PWD/.magpie/checkouts" npm run dev:watcher
```

> The watcher is **required** for all generative work: answering questions, drafting proposals, publishing, and maintenance jobs. Without it, `POST /api/ask` will return `202` and the question will never be answered. The mock provider works out of the box — no additional credentials needed.

## 8. Index Your Content

```bash
curl -s -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"cats"}'
```

After indexing, background embedding runs automatically. For the best search and answer quality, wait until the API logs `Embedded N section(s); 0 remaining` before asking questions. With the mock provider, answers are still returned even without embeddings, but confidence scores may be lower.

## 9. Ask a Question

```bash
curl -s http://localhost:4000/api/ask \
  -H 'content-type: application/json' \
  -d '{"question":"How should I introduce a new cat food?"}'
```

> The `/ask` endpoint is **enqueue-only**: it records the question, returns `202` with a job ID, and enqueues an `answer_question` job for the watcher. To wait for the answer, use the `wait` link:
> ```bash
> # After getting the 202 response
> JOB_ID="..."  # from the response's job.id
> curl -s "http://localhost:4000/api/jobs/$JOB_ID/wait"
> curl -s "http://localhost:4000/api/questions/<question-id>"
> ```
> The question ID is returned as `questionId` in the 202 response.

## 10. (Optional) Start the Web Console

```bash
MAGPIE_DEV_API_PROXY="http://localhost:4000" npm run dev:web
```

Open `http://localhost:3000` to browse the knowledge base.

## Troubleshooting

| Problem | Solution |
|---|---|
| `curl localhost:4000/api/health` fails | Check API terminal; kill other processes on port 4000. |
| Indexing returns `400 configured_repository_required` | Provide a valid `flowId` defined in `KNOWLEDGE_FLOWS`. |
| `/ask` returns `202` but never completes | The watcher is not running. Start it (step 7) and retry the question. |
| `/ask` returns low confidence | Verify indexing completed. Wait for background embedding to finish (step 8 note). |
| Watcher logs `Capability … not ready` | Check that the environment variables for the chosen provider are set correctly. For mock, no extra variables are needed. |
| `401` on API calls | Locally, set `AUTH_REQUIRED=false` in `.env` or as an environment variable. |
| Bootstrap fails with permission error | Override `MAGPIE_CHECKOUT_ROOT` to a writable local path (step 6). |

For more troubleshooting, see the [Managing Knowledge Flows](managing-knowledge-flows-in-markdown-magpie.md) guide.

## Next Steps

- Learn about [knowledge gap detection and proposals](managing-knowledge-flows-in-markdown-magpie.md#the-gap-pipeline-and-flows).
- Set up [real AI providers](configuration-reference.md#ai-provider-configuration).
- Configure [hybrid retrieval with embeddings](configuration-reference.md#embedding-provider-configuration).
- Review [permissions and access controls](permissions-and-access-controls-in-markdown-magpie.md).
- Understand the [architecture and job model](architecture.md).
- Try the [full Docker Compose stack](README.md#docker-compose) for a self-contained demo.

---
*Based on the Getting Started guide. For detailed configuration, see the Configuration Reference.*
