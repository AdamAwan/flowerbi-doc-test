---
title: Supported AI Models and Providers
status: draft
---

# Supported AI Models and Providers

Markdown Magpie is designed around a **provider-neutral** architecture. The system does not hard-code a single model or vendor; instead, it routes AI work through configurable adapters. This document describes the supported inference providers, language models, embedding models, and how to configure them.

## Provider Strategy

The core packages define interfaces for chat completion, embeddings, and AI job execution. Concrete adapters can target:

- Any OpenAI-compatible `/chat/completions` endpoint
- Azure OpenAI
- External agent CLIs (Codex, Claude Code)

The API never calls a model inline. All generative work runs on a separate **watcher** process that claims jobs from a queue and invokes the configured provider.

## Inference Providers

`AI_PROVIDER` is mandatory and selects the provider for answer synthesis and other generative tasks. Valid values:

| Provider | Required Environment Variables |
|----------|--------------------------------|
| `openai-compatible` | `OPENAI_COMPATIBLE_BASE_URL`, `OPENAI_COMPATIBLE_API_KEY`, `OPENAI_COMPATIBLE_MODEL` (e.g., `gpt-4o`, `deepseek-chat`, `claude-sonnet-4-20250514`) |
| `azure-openai` | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_CHAT_DEPLOYMENT` |
| `codex` | `CODEX_CLI_PATH` (defaults to `codex` on `PATH`) |
| `claude` | `CLAUDE_CLI_PATH` (defaults to `claude` on `PATH`) |

Configuration is set via environment variables (`.env`). The watcher advertises a provider **capability** only when its credentials are present; jobs are routed only to capabilities a running watcher offers.

### OpenAI-Compatible

Any model served by an OpenAI-compatible API can be used — for example, OpenAI's GPT-4o, DeepSeek, OpenRouter models, or local Ollama endpoints. Set:

```env
OPENAI_COMPATIBLE_BASE_URL=https://api.openai.com/v1
OPENAI_COMPATIBLE_API_KEY=sk-...
OPENAI_COMPATIBLE_MODEL=gpt-4o
```

### Azure OpenAI

Requires an Azure OpenAI resource with a chat deployment:

```env
AZURE_OPENAI_ENDPOINT=https://example.openai.azure.com
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_CHAT_DEPLOYMENT=gpt-4o
AZURE_OPENAI_API_VERSION=2024-10-21
```

### Codex / Claude

External agent CLIs (Codex or Claude Code) are invoked as subprocesses by the watcher. They handle the full non-embedding LLM job contract.

```env
CODEX_CLI_PATH=codex        # or leave unset to resolve from PATH
CLAUDE_CLI_PATH=claude
```

### Runtime Provider Switching

The active provider can be changed at runtime via `POST /api/config` (see [api.md](api.md)).

## Embedding Models

Embeddings are configured **independently** of chat providers. The system uses embeddings for hybrid (vector + keyword) retrieval. An embedding endpoint is detected by the presence of its credential variables:

| Variable | Purpose |
|----------|---------|
| `KNOWLEDGE_STORE=postgres` + `DATABASE_URL` | Required for vector storage |
| `OPENAI_COMPATIBLE_EMBEDDING_MODEL` | OpenAI-compatible embedding model (e.g., `text-embedding-3-small`, `text-embedding-ada-002`) |
| `OPENAI_COMPATIBLE_EMBEDDING_BASE_URL` | (optional) Override base URL for embeddings |
| `OPENAI_COMPATIBLE_EMBEDDING_API_KEY` | (optional) Override API key for embeddings |
| `AZURE_OPENAI_ENDPOINT` + `AZURE_OPENAI_API_KEY` + `AZURE_OPENAI_EMBEDDING_DEPLOYMENT` | Azure OpenAI embeddings |

Both `text-embedding-3-small` and `ada-002` output 1536-dimensional vectors and are compatible. When embedding credentials are present, hybrid retrieval activates; otherwise the system falls back to keyword-only search.

You can use different providers for chat and embeddings — for example, DeepSeek for answering questions and OpenAI for embeddings.

## Model Support Summary

- **Language models**: Any model accessible via the configured provider. The system does not restrict to a specific list; it relies on the provider's capabilities.
- **Embedding models**: Must output 1536-dimensional vectors (OpenAI-compatible or Azure).
- **CLI agents**: Codex and Claude Code, with their own built-in models.

The system is intentionally model-agnostic. To add a new provider, implement a watcher adapter that claims jobs and calls the provider's API.

## See Also

- [Chat Providers](chat-providers.md) — detailed provider configuration
- [AI Job Contract](ai-jobs.md) — watcher capabilities and job routing
- [Ingestion](ingestion.md) — embedding configuration and hybrid retrieval
- [Architecture](architecture.md) — provider strategy overview