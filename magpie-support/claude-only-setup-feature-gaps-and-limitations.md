---
title: Claude-Only Setup: Feature Gaps and Limitations
status: draft
---

# Claude-Only Setup: Feature Gaps and Limitations

When configuring Markdown Magpie to use **Claude** as the sole AI provider (via the `claude` CLI), some capabilities available with API-based providers (`openai-compatible`, `azure-openai`) or the `codex` CLI are reduced or missing. This article lists the gaps and explains workarounds where they exist.

## 1. Embeddings Must Come From a Separate Provider

Markdown Magpie uses an **independent embedding provider** for indexing and query-time vector search. The embedding provider is configured via `EMBEDDING_PROVIDER` and its own set of environment variables (e.g., `EMBEDDING_MODEL`, `EMBEDDING_API_KEY`).

- **Gap:** The Claude CLI does **not** support embedding generation. If `AI_PROVIDER=claude` but no embedding provider is configured, the system falls back to **keyword-only retrieval** (BM25), which is less accurate for semantic search.
- **Workaround:** Set a separate embedding provider, such as `EMBEDDING_PROVIDER=openai-compatible` with a compatible model (e.g., `text-embedding-ada-002`). This is often already required when running locally because embeddings are computed inline by the API (not by the watcher). See `configuration-reference.md` for the full list of embedding configuration variables.

## 2. No Native Streaming or Tool Use

API-based providers (OpenAI-compatible, Azure OpenAI) support streaming responses and tool/function calling. The Claude CLI mode does not:

- **Streaming:** The watcher expects the full output from the CLI; answers cannot be streamed to the UI.
- **Tool Use:** The CLI provider does not support structured tool calls. The watcher must parse free-form text output to extract citations or structured data. In practice, the current implementation uses prompt-based instructions for citation formatting, which works but is less robust than API-based function calling.

## 3. Prompt-Length Limitations (Windows Only)

On **Windows**, passing the prompt as a command-line argument (`CLAUDE_CLI_PROMPT_MODE=arg`) can fail with `spawn ENAMETOOLONG` when the prompt exceeds approximately 32 KB. This is common with retrieval-augmented prompts that include many context sections.

- **Workaround:** Set `CLAUDE_CLI_PROMPT_MODE=stdin` to pipe the prompt through stdin, which avoids the command-line length limit. This is effectively required on Windows for most real-world use. See `magpie-local-troubleshooting` skill for details.

## 4. No Built-in Rate-Limiting or Retry Logic

The watcher delegates all retry and rate-limit handling to the CLI binary. API-based providers have built-in retry logic in the `openai-compatible` watcher runner. With Claude CLI:

- **Gap:** If the `claude` binary returns a non-zero exit code (e.g., due to API throttling), the watcher marks the job as failed after a single attempt unless a custom wrapper is used. There is no automatic exponential backoff within the watcher.

## 5. Model Selection Is Limited to What the CLI Supports

The `CLAUDE_CLI_MODEL` variable is passed as `--model` to the binary. Not all Claude models available via API are available via the CLI; newer models may lag or require specific binary versions.

- **Gap:** You are constrained to the models the `claude` executable supports (check its documentation). Switching to a different model family (e.g., Opus vs Sonnet) requires a binary update, not just an environment variable change.

## 6. Maintenance Orchestrators May Be Slower

Maintenance jobs (e.g., `verify_gap_closure`, patrols) enqueue multiple AI sub-jobs. With Claude CLI, each sub-job is a separate process spawn, which adds startup overhead. With API-based providers, the watcher can reuse connections and benefit from batch endpoints (where supported).

- **Gap:** Running maintenance with Claude CLI can be noticeably slower than with API providers, especially when many verifications are triggered (e.g., after a merged PR).

## Summary Table

| Feature                          | API Providers        | Claude CLI           | Workaround                                  |
|----------------------------------|----------------------|----------------------|---------------------------------------------|
| Embeddings                       | Built-in (separate)  | Not available        | Configure a separate embedding provider     |
| Streaming                        | Supported            | Not supported        | N/A                                         |
| Tool/function calling            | Supported            | Not supported        | Prompt-based parsing (less robust)          |
| Windows prompt length            | No issue             | Limited (arg mode)   | Use stdin mode (`CLAUDE_CLI_PROMPT_MODE=stdin`) |
| Retry/rate-limit handling        | Built-in backoff     | No retry (binary exit code) | Wrap `claude` in a retrying script |
| Model flexibility                | Wide model selection | Tied to CLI binary   | Update binary or switch provider            |
| Maintenance job performance      | Efficient            | Slower (per-job spawn) | Accept slower performance or add multiple watchers |

## Conclusion

Using Claude CLI as the sole provider is feasible for small to medium knowledge bases, but requires a separate embedding provider and awareness of the gaps above. For production deployments expecting high query volume or advanced retrieval accuracy, consider using an API-based provider for chat and keeping Claude only for specific tasks, or running both providers simultaneously (the watcher can handle multiple provider queues).

See:
- [model-configuration-and-integration.md](model-configuration-and-integration.md) for Claude CLI configuration details.
- [configuration-reference.md](configuration-reference.md) for all provider environment variables.
- [integrations-and-data-sources-in-markdown-magpie.md](integrations-and-data-sources-in-markdown-magpie.md) for an overview of chat and embedding providers.
