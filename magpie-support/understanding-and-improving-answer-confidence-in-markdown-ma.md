---
title: Understanding and Improving Answer Confidence in Markdown Magpie
status: draft
---

# Understanding and Improving Answer Confidence in Markdown Magpie

When Markdown Magpie returns a low-confidence answer, it means the system could not find sufficiently relevant source material in the indexed knowledge base to answer your question with high certainty. This guide explains why low-confidence answers occur and how to improve answer quality.

## Why Low-Confidence Answers Happen

Confidence is derived from the relevance scores of the indexed sections retrieved for your question. A low score indicates one of the following:

- **The knowledge index is empty or incomplete.** If no documents have been indexed, there is no material to retrieve, and every answer will be low-confidence. This is the most common cause in fresh deployments or after a reset.
- **The question falls outside the scope of the indexed documents.** The knowledge base may not cover the topic, or the relevant information exists but is not effectively retrievable due to poor document structure or lack of coverage.
- **The retrieval mode is not optimal.** Markdown Magpie supports two retrieval modes: `keyword` (in-memory term matching) and `hybrid` (keyword + pgvector embeddings). Hybrid mode generally yields higher relevance and confidence because it captures semantic meaning beyond exact keyword matches.
- **The embeddings are not generated.** Even with hybrid retrieval enabled, if embeddings have not been computed for indexed sections (e.g., they are still `NULL` in the database), the hybrid fallback may be keyword-only, lowering retrieval quality.
- **The AI provider is not configured correctly.** In `direct` mode, the answer is synthesized by a chat provider (e.g., OpenAI-compatible or Azure OpenAI). If the provider is misconfured, the answer may be generic or nonsensical, leading to low confidence scores.
- **The question is ambiguous or malformed.** The retrieval pipeline works best with clear, specific questions.

## Checking Current Answer Quality

1. **Inspect the answer response:** When you call `POST /api/ask`, the result includes a `confidence` field (`high`, `medium`, or `low`) and a list of `citations` with relevance scores. Low confidence often means zero or very few citations.
2. **Review the knowledge base stats:**
   ```bash
   curl http://localhost:4000/api/knowledge/stats
   ```
   If `sectionCount` is 0, the index is empty.
3. **Check retrieval mode:**
   ```bash
   curl http://localhost:4000/api/config
   ```
   Look for `retrieval.mode` — it should be `hybrid` for best results. If it is `keyword`, consider configuring embeddings.
4. **List gap candidates:**
   ```bash
   curl http://localhost:4000/api/gaps/candidates
   ```
   This shows questions the system has flagged as low-confidence. Repeated similar gaps indicate a knowledge base gap.

## How to Improve Answer Quality

### 1. Ensure the Knowledge Base Is Indexed

If the index is empty, index the configured flow:

```bash
curl -X POST http://localhost:4000/api/knowledge/repositories/index \
  -H 'content-type: application/json' \
  -d '{"flowId":"flowerbi"}'
```

Wait for the background embedding pass to finish. You can monitor progress by checking the API logs for "Embedded N section(s); 0 remaining".

### 2. Configure Hybrid Retrieval with Embeddings

Hybrid retrieval (keyword + vector) significantly improves relevance. To enable it:

- Set `KNOWLEDGE_STORE=postgres` and provide a valid `DATABASE_URL`.
- Set either:
  - **OpenAI-compatible embedding provider:** Set `OPENAI_COMPATIBLE_EMBEDDING_MODEL` (e.g., `text-embedding-3-small`), `OPENAI_COMPATIBLE_EMBEDDING_BASE_URL`, and `OPENAI_COMPATIBLE_EMBEDDING_API_KEY`.
  - **Azure OpenAI embedding provider:** Set `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, and `AZURE_OPENAI_EMBEDDING_DEPLOYMENT`.

After configuration, re-index the flow (step 1) to generate embeddings. The system will automatically switch to hybrid mode.

### 3. Expand and Improve the Knowledge Base Content

Low confidence often means the knowledge base lacks relevant documents. Use the gap-clustering feature to identify missing topics:

```bash
curl http://localhost:4000/api/gaps/clusters
```

Each cluster represents a set of related questions that a single document could resolve. Manually draft or generate a proposal:

```bash
curl -X POST http://localhost:4000/api/proposals/from-gap \
  -H 'content-type: application/json' \
  -d '{"summary":"No hotfix rollback procedure is documented","destinationId":"flowerbi-docs"}'
```

Review the generated Markdown and publish it to a branch, then merge it. After re-indexing, the same question will return high-confidence answers.

### 4. Optimize Document Structure

Markdown Magpie splits documents by headings. To improve retrieval:

- Use descriptive, unique headings that summarize section content.
- Keep sections focused on a single topic.
- Include relevant keywords naturally.
- Avoid extremely long sections; break them into smaller, well-named subsections.

### 5. Verify AI Provider Configuration

If the answer content itself is poor (not just low confidence), check the chat provider:

- Ensure the provider environment variables are set correctly (e.g., `OPENAI_COMPATIBLE_API_KEY`, `AZURE_OPENAI_CHAT_DEPLOYMENT`).
- Test with a simple question that should be well-covered.
- Switch to the `mock` provider to isolate issues: set `AI_PROVIDER=mock` and restart. `mock` produces deterministic answers from retrieved context without requiring API keys.

### 6. Monitor Feedback and Gap Signals

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

## Summary Table

| Symptom | Likely Cause | Action |
|---|---|---|
| All answers are low-confidence | Knowledge index is empty | Index the flow and wait for embedding pass |
| Low confidence on specific topics | Knowledge gap in the base | Write or generate a proposal for the missing topic |
| Retrieval mode is `keyword` | Embeddings not configured | Set embedding provider and re-index |
| Answers are gibberish or off-topic | AI provider misconfigured or down | Check provider env vars, test with `mock` |
| Questions return few citations | Poor document structure | Rewrite sections with clear headings and better keywords |

By following these steps, you can systematically raise answer confidence from low to high and close knowledge gaps over time.
