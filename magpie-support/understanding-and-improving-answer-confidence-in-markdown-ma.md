---
title: Understanding and Improving Answer Confidence in Markdown Magpie
status: draft
---

# Understanding and Improving Answer Confidence in Markdown Magpie

When Markdown Magpie returns a low-confidence answer, it means the system could not find sufficiently relevant source material in the indexed knowledge base to answer your question with high certainty. This guide explains why low-confidence answers occur and how to improve answer quality.

## What Does Confidence Mean in Magpie?

Magpie assigns a confidence score to each generated answer. This score reflects how certain the system is that the answer is correct and well‑supported. Confidence is typically derived from:

- **Model probability**: The likelihood assigned by the underlying language model to the generated tokens.
- **Retrieval relevance**: When using Retrieval Augmented Generation (RAG), how closely the retrieved documents match the query.
- **Answer completeness**: Whether the answer directly addresses the question without ambiguity.

Low confidence does not necessarily mean the answer is wrong – it indicates that the system has less assurance due to missing or conflicting evidence.

In some cases, the system may return a confidence of **`unknown`**. This happens when the question router cannot determine which knowledge flow best matches the question and the answer is withheld. The caller is then asked to pick a flow and re‑ask the question. When `unknown` is returned, the answer is a short human‑readable note, no citations are provided, and a `flowSelectionRequired` field lists the available flows for the caller to choose from.

### How Retrieval Mode Affects Confidence

Markdown Magpie supports two retrieval modes: `keyword` (in-memory term matching) and `hybrid` (keyword + pgvector embeddings). Hybrid mode generally yields higher relevance and confidence because it captures semantic meaning beyond exact keyword matches.

When both `KNOWLEDGE_STORE=postgres` and an embeddings provider are configured, retrieval is **hybrid**: a pgvector nearest-neighbour search is fused with keyword relevance scores (using an in-memory keyword scorer with heading match +3, content match +1, or when the Postgres full-text search backend is available, that is used instead) using Reciprocal Rank Fusion (RRF). Results carry a `[0,1]` relevance score. When either condition is absent, the system falls back to keyword-only scoring. The active retrieval mode is reported by `GET /api/config` under `retrieval.mode` (`hybrid` or `keyword`) along with a plain-language `reason`.

Query-time embedding (embedding the user's question) is synchronous in the API request. Section embeddings are computed inline (within the API process) during indexing, not as a separate asynchronous job.

**Note:** CLI agents such as Codex and Claude cannot produce embeddings. Embeddings must come from an admin-configured OpenAI-compatible or Azure endpoint.

## Why Low-Confidence Answers Happen

Confidence is derived from the relevance scores of the indexed sections retrieved for your question. A low score indicates one of the following:

- **The knowledge index is empty or incomplete.** If no documents have been indexed, there is no material to retrieve, and every answer will be low-confidence. This is the most common cause in fresh deployments or after a reset.
- **The question falls outside the scope of the indexed documents.** The knowledge base may not cover the topic, or the relevant information exists but is not effectively retrievable due to poor document structure or lack of coverage.
- **The retrieval mode is not optimal.** Keyword-only retrieval limits the ability to match semantically similar content. Hybrid mode (keyword + vector) is recommended for best results.
- **The embeddings are not generated.** Even with hybrid retrieval enabled, if embeddings have not been computed for indexed sections (e.g., they are still `NULL` in the database), the hybrid fallback may be keyword-only, lowering retrieval quality.
- **No watcher is running with the required capability.** Markdown Magpie uses a queue‑only architecture: the API never calls a chat model inline. It enqueues an `answer_question` job, and a separate **watcher** process claims it, calls the configured chat provider, and posts the result back. Jobs move through states: `created` → `active` → `completed` (or `failed`, `retry`, `cancelled`, `blocked`). A watcher claims active jobs; if none is running, jobs remain `created` indefinitely, resulting in no answer or an answer with unknown confidence. Note that there is no longer a "direct mode" (the old `AI_EXECUTION_MODE` option was removed) — the watcher is always required for generative work. The watcher does not have direct database access; it communicates with the API over HTTP (using scoped endpoints like retrieval) and shares only the managed-checkout volume for file operations. Check the watcher's startup logs for "Capability provider — ready" or similar.
- **The AI provider is not configured correctly.** The watcher advertises capabilities only when all required environment variables are set for that provider. Missing API keys, base URLs, or model names prevent the watcher from claiming jobs, leading to stalled answers and low confidence.
- **The question is ambiguous or malformed.** The retrieval pipeline works best with clear, specific questions.
- **Insufficient or poorly formatted context.** If the knowledge base lacks relevant information or contains conflicting data, Magpie may produce an answer with low confidence.
- **Outdated or incomplete knowledge base.** When the source documents used for RAG are not up‑to‑date or missing key topics, the generated answer may rely on weak evidence.
- **Model limitations.** Smaller or less capable models may struggle with complex reasoning, leading to lower confidence even when context is sufficient.
- **The question could not be routed to a specific knowledge flow.** When using `auto` routing, if the model abstains from choosing a flow, the answer is withheld with confidence `unknown` and a `flowSelectionRequired` field. The caller must re-ask with an explicit `flow` parameter. This is not a failure – it is a deliberate signal that the system cannot determine the correct knowledge area.

## Checking Current Answer Quality

1. **Submit a question and retrieve the answer.** `POST /api/ask` returns `202` with a job object and a `questionId`; the answer is not in this response. To get the final answer:
   - Poll `GET /api/jobs/<job-id>/wait` — this long-polls until the job is terminal (**`200`** when complete, **`202`** if still running, in which case you re-issue the call).
   - Once the job is complete, fetch `GET /api/questions/<question-id>` to see the answer, confidence (`high`, `medium`, `low`, or `unknown`), and citations with relevance scores. If the answer has confidence `unknown`, look for the `flowSelectionRequired` field to see which flows are available.
   Low confidence often means zero or very few citations. When `unknown`, no citations are provided.
2. **Review the knowledge base stats:**
   ```bash
   curl http://localhost:4000/api/knowledge/stats
   ```
   If `sectionCount` is 0, the index is empty.
3. **Check retrieval mode and watcher capability:**
   ```bash
   curl http://localhost:4000/api/config
   ```
   Look for `retrieval.mode` — it should be `hybrid` for best results. Also check `ai.runtime.availableProviders` to see which watcher capabilities are active.
4. **Inspect job schedules (cron tasks):**
   ```bash
   curl http://localhost:4000/api/jobs/schedules
   ```
   This lists pg-boss cron schedules that drive maintenance tasks such as gap drafting and patrol jobs.
5. **List gap candidates:**
   ```bash
   curl http://localhost:4000/api/gaps/candidates
   ```
   This shows questions the system has flagged as low-confidence. Repeated similar gaps indicate a knowledge base gap.
6. **Monitor watcher logs and confirm the watcher is running:**
   The watcher logs on startup which capabilities are ready. If `Capability provider — ready` is missing, verify the provider credentials. Also confirm a watcher process is running (e.g., `npm run dev:watcher` or equivalent).
7. **Check deep readiness and build identity:**
   ```bash
   curl http://localhost:4000/api/ready
   curl http://localhost:4000/api/version
   ```
   `/api/ready` reports whether Postgres and the job broker are reachable (returns `200` when ready, `503` otherwise). `/api/version` shows the deployed commit's short SHA, subject line, and committer date, which is useful for confirming which build is running.

## How to Improve Answer Quality

### 1. Ensure the Knowledge Base Is Indexed

If the index is empty, index the configured flow:

```bash
curl -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"flowerbi"}'
```

The API spawns a background task to embed any sections whose embedding is `NULL`. This embedding pass runs inside the API process — there is no separate watcher or scheduled job for it. You can monitor progress by checking the API logs for "Embedded N section(s); 0 remaining".

### 2. Configure Hybrid Retrieval with Embeddings

Hybrid retrieval (keyword + vector) significantly improves relevance. To enable it:

- Set `KNOWLEDGE_STORE=postgres` and provide a valid `DATABASE_URL`.
- Set either:
  - **OpenAI-compatible embedding provider:** Set `OPENAI_COMPATIBLE_EMBEDDING_MODEL` (e.g., `text-embedding-3-small`), `OPENAI_COMPATIBLE_BASE_URL`, and `OPENAI_COMPATIBLE_API_KEY`. The embedding endpoint resolves its base URL and API key from `OPENAI_COMPATIBLE_EMBEDDING_BASE_URL` / `OPENAI_COMPATIBLE_EMBEDDING_API_KEY`, each falling back to the shared chat values when left blank.
  - **Azure OpenAI embedding provider:** Set `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, and `AZURE_OPENAI_EMBEDDING_DEPLOYMENT`.

Embeddings are configured **independently of chat** — you can answer questions with one provider and embed with another (e.g., DeepSeek for `/api/ask`, OpenAI for embeddings). The system automatically activates hybrid retrieval when both `KNOWLEDGE_STORE=postgres` and a complete set of embedding credentials are present. `GET /api/config` reports the resolved `retrieval.mode` and a plain‑language `reason`. Note that CLI agents (Codex, Claude) cannot produce embeddings; only administrator-configured embedding endpoints can.

After configuration, re-index the flow (step 1) to generate embeddings.

### 3. Expand and Improve the Knowledge Base Content

Low confidence often means the knowledge base lacks relevant documents. Use the gap-clustering feature to identify missing topics:

```bash
curl http://localhost:4000/api/gaps/clusters
```

Each cluster represents a set of related questions that a single document could resolve. The `gaps-to-pull-requests` reconciler (a scheduled maintenance job) automatically clusters gaps, drafts proposals for uncovered clusters, publishes them as pull requests, and advances proposals as their PRs merge or close. Before clustering, the reconciler prunes resolved gaps (those whose proposals have merged) and dismisses off-topic clusters that are unrelated to the knowledge base, preventing unnecessary proposals. You can also manually draft or generate a proposal:

```bash
curl -X POST http://localhost:4000/api/proposals/from-gap \
  -H 'content-type: application/json' \
  -d '{"summary":"No hotfix rollback procedure is documented","destinationId":"flowerbi-docs"}'
```

Proposals move through a status lifecycle: `draft`, `ready`, `branch-pushed`, `pr-opened`, `merged`, `rejected`. Once a proposal is `ready` and its target path maps to an indexed Git checkout, you can publish it:

```bash
POST /api/proposals/:id/publish
```

Publication enqueues a job that the watcher completes by committing the Markdown to a new branch, pushing it, and opening a pull request. Review the generated Markdown and merge it. After re-indexing, the same question will return high-confidence answers.

### 4. Optimize Document Structure

Markdown Magpie splits documents by headings. To improve retrieval:

- Use descriptive, unique headings that summarize section content.
- Keep sections focused on a single topic.
- Include relevant keywords naturally.
- Avoid extremely long sections; break them into smaller, well-named subsections.

### 5. Refine Your Queries

- Be specific and avoid open‑ended phrasing.
- Include relevant keywords that match the structure of your knowledge base.
- Use prompt engineering: provide clear instructions in system prompts, specifying the desired format and level of certainty.

### 6. Verify AI Provider and Watcher Configuration

If the answer content itself is poor (not just low confidence), check the chat provider and watcher.

- Ensure the provider environment variables are set correctly. A watcher advertises a **capability** for each provider whose credentials are present in its environment; the API only routes a job to a capability a running watcher actually offers. The table below lists the required environment variables per capability:

| Capability | Required environment variables |
|---|---|
| `openai-compatible` | `OPENAI_COMPATIBLE_BASE_URL`, `OPENAI_COMPATIBLE_API_KEY`, `OPENAI_COMPATIBLE_MODEL` |
| `azure-openai` | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_CHAT_DEPLOYMENT` |
| `codex` | `CODEX_CLI_PATH` (defaults to `codex` on `PATH`) |
| `claude` | `CLAUDE_CLI_PATH` (defaults to `claude` on `PATH`) |
| `local-git` | `MAGPIE_GIT_AUTHOR_NAME`, `MAGPIE_GIT_AUTHOR_EMAIL` (and git on `PATH`) |
| `github` | `GITHUB_TOKEN`, `MAGPIE_GIT_AUTHOR_NAME`, `MAGPIE_GIT_AUTHOR_EMAIL` |
| `maintenance` | (none; always available) |

  The `github` capability is required for publishing proposals as pull requests. The `local-git` capability publishes proposals to file:// destinations (branch push only, no PR). The `maintenance` capability covers scheduled tasks such as gap reconciliation, source sync, and patrol jobs.

- Confirm the watcher is running and advertises the required capability. The watcher logs will show `Capability provider — ready` when its credentials match the configured `AI_PROVIDER`. If this line is missing, review the startup logs for errors.
- You can also check the active capabilities by examining the `ai.runtime.availableProviders` field in the response from `GET /api/config`.
- Test with a simple question that should be well-covered. If the job stays queued, consult the watcher logs for errors.
- To isolate provider issues, temporarily switch to a known-working provider (e.g., `openai-compatible` with a test endpoint) or choose a different provider from the table above.

### 7. Adjust System Parameters

- **Temperature**: Lower values (e.g., 0.1–0.3) produce more deterministic answers; higher values increase creativity but may reduce confidence.
- **Top‑k / Top‑p**: Tighten these parameters to limit the model’s sampling space for more focused outputs.
- **Max tokens**: Ensure the answer length is sufficient to fully address the question.

### 8. Monitor Feedback and Gap Signals

Use the feedback mechanism to improve the system:

```bash
curl -X POST http://localhost:4000/api/questions/:id/feedback \
  -H 'content-type: application/json' \
  -d '{"feedback":"unhelpful"}'
curl -X POST http://localhost:4000/api/questions/:id/gap \
  -H 'content-type: application/json' \
  -d '{"summary":"Missing information on X"}'
```

This feeds the gap-clustering algorithm and helps prioritize which documents to write.

### 9. Evaluate and Iterate

- Review low‑confidence answers to identify patterns (e.g., specific topics or question types).
- Test changes to queries or knowledge base updates to measure improvement.

## Scheduled Maintenance Tasks

Markdown Magpie runs several scheduled maintenance jobs that automatically address low-confidence gaps and keep the knowledge base healthy. These tasks are configured per flow and managed from the **Schedules** page in the web console. Each scheduled task fires on its cron cadence and may fan out into one or more AI jobs:

| Task label | Job type | Cadence | What it does |
|---|---|---|---|
| Gap drafting | `process_gaps_to_pull_requests` | ~10 min | Clusters new low-confidence answers into gaps, prunes resolved gaps, dismisses off-topic clusters, drafts proposals for uncovered clusters, and publishes them as pull requests. |
| Source sync | `source_change_sync` | ~10 min | Detects upstream source changes and generates update proposals. |
| Snapshot refresh | `refresh_flow_snapshot` | ~5 min | Writes a flow snapshot of gaps, proposals, and PR state for reconciler consumption. |
| Correctness patrol | `correctness_patrol` | hourly | Verifies document claims, corrects errors, deduplicates, and splits overly large files. |
| Editorial patrol | `editorial_patrol` | hourly | Improves completeness of fine-but-thin documents. |

These tasks operate through the same job queue as answer synthesis. The `maintenance` capability (always available) is required for the orchestrator jobs. For details, see the architecture documentation.

## Interpreting Confidence Scores

Confidence is a tool for developers and users, not an absolute measure of correctness. Use it as a guide to:
- Flag answers that require human review.
- Prioritise which knowledge base sections need enrichment.
- Compare model versions or prompt configurations.
- When confidence is `unknown`, it means the system could not route the question – the caller should pick a flow from the returned `flowSelectionRequired` list and re-ask.

## Summary Table

| Symptom | Likely Cause | Action |
|---|---|---|
| All answers are low-confidence | Knowledge index is empty | Index the flow and wait for embedding pass |
| Low confidence on specific topics | Knowledge gap in the base | Write or generate a proposal for the missing topic |
| Retrieval mode is `keyword` | Embeddings not configured | Set embedding provider and re-index |
| No answer or job never completes | No watcher running or missing capability | Start watcher, check credentials, verify startup logs |
| Answers are gibberish or off-topic | AI provider credentials wrong, misconfigured, or watcher not running | Check provider env vars, verify watcher is running, review watcher logs, test with a different provider |
| Questions return few citations | Poor document structure | Rewrite sections with clear headings and better keywords |
| Ambiguous questions lead to low confidence | Query too vague | Refine queries to be specific |
| Answer confidence is `unknown` with `flowSelectionRequired` | Router could not determine a flow | Re-ask with an explicit `flow` parameter set to one of the available flow ids |

By following these steps, you can systematically raise answer confidence from low to high and close knowledge gaps over time.
