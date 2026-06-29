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

### How Retrieval Mode Affects Confidence

Markdown Magpie supports two retrieval modes: `keyword` (in-memory term matching) and `hybrid` (keyword + pgvector embeddings). Hybrid mode generally yields higher relevance and confidence because it captures semantic meaning beyond exact keyword matches.

When both `KNOWLEDGE_STORE=postgres` and an embeddings provider are configured, retrieval is **hybrid**: a pgvector nearest-neighbour search is fused with an in-memory keyword scorer (heading match +3, content match +1) using Reciprocal Rank Fusion (RRF). Results carry a `[0,1]` relevance score. When either condition is absent, the system falls back to keyword-only scoring. The active retrieval mode is reported by `GET /api/config` under `retrieval.mode` (`hybrid` or `keyword`) along with a plain-language `reason`.

Query-time embedding (embedding the user's question) is synchronous in the API request. Section embeddings are generated asynchronously after an index operation.

## Why Low-Confidence Answers Happen

Confidence is derived from the relevance scores of the indexed sections retrieved for your question. A low score indicates one of the following:

- **The knowledge index is empty or incomplete.** If no documents have been indexed, there is no material to retrieve, and every answer will be low-confidence. This is the most common cause in fresh deployments or after a reset.
- **The question falls outside the scope of the indexed documents.** The knowledge base may not cover the topic, or the relevant information exists but is not effectively retrievable due to poor document structure or lack of coverage.
- **The retrieval mode is not optimal.** Keyword-only retrieval limits the ability to match semantically similar content. Hybrid mode (keyword + vector) is recommended for best results.
- **The embeddings are not generated.** Even with hybrid retrieval enabled, if embeddings have not been computed for indexed sections (e.g., they are still `NULL` in the database), the hybrid fallback may be keyword-only, lowering retrieval quality.
- **No watcher is running with the required capability.** Markdown Magpie uses a queue‑only architecture: the API never calls a chat model inline. It enqueues an `answer_question` job, and a separate **watcher** process claims it, calls the configured chat provider, and posts the result back. If no watcher is running or the watcher does not advertise the required capability (e.g., `openai-compatible`, `azure-openai`), the job stays queued and never completes, resulting in no answer or an answer with unknown confidence. Check the watcher's startup logs for "Capability provider — ready" or similar.
- **The AI provider is not configured correctly.** The watcher advertises capabilities only when all required environment variables are set for that provider. Missing API keys, base URLs, or model names prevent the watcher from claiming jobs, leading to stalled answers and low confidence.
- **The question is ambiguous or malformed.** The retrieval pipeline works best with clear, specific questions.
- **Insufficient or poorly formatted context.** If the knowledge base lacks relevant information or contains conflicting data, Magpie may produce an answer with low confidence.
- **Outdated or incomplete knowledge base.** When the source documents used for RAG are not up‑to‑date or missing key topics, the generated answer may rely on weak evidence.
- **Model limitations.** Smaller or less capable models may struggle with complex reasoning, leading to lower confidence even when context is sufficient.

## Checking Current Answer Quality

1. **Inspect the answer response:** When you call `POST /api/ask`, the result includes a `confidence` field (`high`, `medium`, or `low`) and a list of `citations` with relevance scores. Low confidence often means zero or very few citations.
2. **Review the knowledge base stats:**
   ```bash
   curl http://localhost:4000/api/knowledge/stats
   ```
   If `sectionCount` is 0, the index is empty.
3. **Check retrieval mode and watcher capability:**
   ```bash
   curl http://localhost:4000/api/config
   ```
   Look for `retrieval.mode` — it should be `hybrid` for best results. Also check `ai.runtime.provider` and `ai.runtime.availableProviders` to see which watcher capabilities are active.
4. **List gap candidates:**
   ```bash
   curl http://localhost:4000/api/gaps/candidates
   ```
   This shows questions the system has flagged as low-confidence. Repeated similar gaps indicate a knowledge base gap.
5. **Monitor watcher logs and confirm the watcher is running:**
   The watcher logs on startup which capabilities are ready. If `Capability provider — ready` is missing, verify the provider credentials. Also confirm a watcher process is running (e.g., `npm run dev:watcher` or equivalent).

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

Embeddings are configured **independently of chat** — you can answer questions with one provider and embed with another (e.g., DeepSeek for `/api/ask`, OpenAI for embeddings). The system automatically activates hybrid retrieval when both `KNOWLEDGE_STORE=postgres` and a complete set of embedding credentials are present. `GET /api/config` reports the resolved `retrieval.mode` and a plain‑language `reason`.

After configuration, re-index the flow (step 1) to generate embeddings.

### 3. Expand and Improve the Knowledge Base Content

Low confidence often means the knowledge base lacks relevant documents. Use the gap-clustering feature to identify missing topics:

```bash
curl http://localhost:4000/api/gaps/clusters
```

Each cluster represents a set of related questions that a single document could resolve. The `gaps-to-pull-requests` reconciler (a scheduled maintenance job) automatically clusters gaps, drafts proposals for uncovered clusters, publishes them as pull requests, and advances proposals as their PRs merge or close. You can also manually draft or generate a proposal:

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

- Confirm the watcher is running and advertises the required capability. The watcher logs will show `Capability provider — ready` when its credentials match the configured `AI_PROVIDER`. If this line is missing, review the startup logs for errors.
- You can also check the active capabilities by examining the `ai.runtime.availableProviders` field in the response from `GET /api/config`.
- Test with a simple question that should be well-covered. If the job stays queued, consult the watcher logs for errors.
- Switch to the `mock` provider to isolate issues: set `AI_PROVIDER=mock` and restart the watcher. `mock` produces deterministic answers from retrieved context without requiring API keys.

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

## Interpreting Confidence Scores

Confidence is a tool for developers and users, not an absolute measure of correctness. Use it as a guide to:
- Flag answers that require human review.
- Prioritise which knowledge base sections need enrichment.
- Compare model versions or prompt configurations.

## Summary Table

| Symptom | Likely Cause | Action |
|---|---|---|
| All answers are low-confidence | Knowledge index is empty | Index the flow and wait for embedding pass |
| Low confidence on specific topics | Knowledge gap in the base | Write or generate a proposal for the missing topic |
| Retrieval mode is `keyword` | Embeddings not configured | Set embedding provider and re-index |
| No answer or job never completes | No watcher running or missing capability | Start watcher, check credentials, verify startup logs |
| Answers are gibberish or off-topic | AI provider credentials wrong, misconfigured, or watcher not running | Check provider env vars, verify watcher is running, review watcher logs, test with `mock` |
| Questions return few citations | Poor document structure | Rewrite sections with clear headings and better keywords |
| Ambiguous questions lead to low confidence | Query too vague | Refine queries to be specific |

By following these steps, you can systematically raise answer confidence from low to high and close knowledge gaps over time.
