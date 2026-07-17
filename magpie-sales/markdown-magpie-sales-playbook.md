---
title: Markdown Magpie Sales Playbook
status: draft
---

# Markdown Magpie Sales Playbook

This playbook describes a repeatable sales methodology for selling Markdown Magpie. It covers qualification, discovery, demonstration, and close, grounded in the product's architecture and design. It is intended for sales engineers and account executives who understand the product's technical foundations.

## The Core Positioning: Won't Lie, Won't Leak, Won't Rot

Markdown Magpie is a Git-backed Markdown knowledge maintenance system. The product's sharpened pitch — "Won't lie, won't leak, won't rot" — maps directly onto the failure modes of ordinary internal knowledge bases and is the organising framework for every sales conversation.

- **Won't lie**: every answer cites the file, heading, and commit it came from. The system logs confidence and abstains when unsure. Answers are grounded by construction, not by "an LLM that is usually right."
- **Won't leak**: raw material (code, internal docs, restricted folders) never reaches end users. They see a governed, curated knowledge layer. Every change is a reviewed Git PR with full audit history.
- **Won't rot**: the system finds its own gaps by clustering failed and low-confidence questions, drafts fixes, raises PRs, and runs scheduled patrols to verify correctness and improve thin documents. Usage itself is the maintenance signal.

## Qualification Framework

A prospect is a good fit for Markdown Magpie when they meet the following criteria:

### Core Requirements

- **Git-backed Markdown documentation** — the prospect's knowledge base is stored as Markdown in a Git repository, or they are willing to adopt Markdown as their documentation format.
- **Need for cited, trustworthy answers** — the prospect has a need for answers that can be traced back to source material, rather than opaque LLM output.
- **Existing documentation pain** — the prospect's knowledge base is stale, incomplete, or goes out of date faster than it can be maintained.

### Strong Indicators

- **Multiple source repositories** — the prospect draws on several codebases, internal wikis, or vendor docs to produce one curated knowledge base.
- **Regulated or proprietary content** — the prospect handles data that cannot be sent to a public AI provider without governance.
- **Existing investment in AI** — the prospect already uses or evaluates LLM tooling (Claude Code, Codex, OpenAI) and understands the limitations of ungrounded answers.
- **MCP-aware tooling** — the prospect uses AI agents (Claude Code, Cursor, VS Code extensions) that can connect via the Model Context Protocol.

### Poor Fit (Challenges but Not Deal-Breakers)

- **Non-Markdown documentation** — the prospect uses a binary format (Word, Confluence, Notion). These require export/conversion before Magpie can index them.
- **No Git workflow** — the prospect does not use Git for documentation. The Git backing is essential for PR-based review and provenance.
- **Single-person docs** — a one-person documentation effort may not justify the overhead of the full pipeline, though Magpie's automated maintenance still adds value.

## The Technology Decision-Maker Conversation

Markdown Magpie's primary buyer is a technically strong decision-maker who is enthusiastic about AI but sceptical of ungrounded, ungoverned LLM deployments. The conversation should be structured around three proofs:

### Proof 1: Grounded by Construction

The prospect should understand that Magpie's API never calls a chat model inline. All generative AI work is modelled as jobs on a pg-boss queue in Postgres. The API enqueues a job; a separate watcher process claims it, invokes the configured provider, and posts the result back over HTTP. This architecture means:

- The API has **no direct provider dependency** — it never blocks on a model call.
- The watcher has **no database access** — it receives tightly scoped job payloads and calls back into the API only over HTTP.
- The provider can be swapped at any time by changing an environment variable and restarting the watcher.

Embeddings are the single exception: the API computes them inline for indexing and retrieval, because they are fast, cheap, and not generative.

### Proof 2: Provider-Neutral Design

The prospect is not locked into any AI vendor. `AI_PROVIDER` selects the provider:

- `openai-compatible` — any OpenAI-compatible `/chat/completions` endpoint, including self-hosted models (vLLM, Ollama).
- `azure-openai` — Azure OpenAI chat completions.
- `codex` — an external Codex CLI the watcher shells out to.
- `claude` — an external Claude CLI the watcher shells out to.

A watcher advertises a capability only when that provider's credentials are present in its environment. The API routes a job to a capability only a running watcher actually offers. This means a prospect can run Magpie entirely against self-hosted models for sensitive corpora, or mix providers (e.g. DeepSeek for answers, OpenAI for embeddings).

### Proof 3: Mandatory Human Review

No AI-generated content reaches a destination repository's default branch without a human merging a pull request. Automated flows may draft, publish a branch, and open a PR, but the merge is always a human action on the hosting provider. This is the primary control against prompt injection and the non-negotiable invariant of the system.

## The Demo Flow

A live demonstration should follow a three-part structure that mirrors the product's three promises.

### Part 1: Ask and Verify (Won't Lie)

Start by showing the `POST /api/ask` endpoint or the MCP `kb_ask` tool from within Claude Code. Ask a question that the knowledge base can answer, and show:

- The answer is returned with cited sections (document ID, path, heading, excerpt).
- Confidence is reported as high, medium, or low.
- When the knowledge base does not cover the question, the system says so — it abstains rather than fabricating.
- The **Answer trace** (available in the web console) shows the routing decision, every follow-up search the model ran, and the grounding-verification outcome.

Then ask a question the knowledge base cannot answer, and show the gap detection: the question is logged, the `auto` gap is created, and it enters the gap-candidate pool.

### Part 2: From Gap to Pull Request (Won't Rot)

Demonstrate the maintenance pipeline:

1. **Gap clustering** — show the reconciler grouping related gaps into clusters on the web console's Gaps page.
2. **Proposal drafting** — trigger a draft from a gap cluster via the console or the API. Show the draft pending in the Proposals page.
3. **Publication** — mark the proposal `ready` and publish it. Show the watcher committing the Markdown to a `magpie/proposal-*` branch and opening a pull request.
4. **Review** — show the PR on GitHub (or the local-git Accept/Bin buttons in the console for `file://` destinations).
5. **Merge and verify** — after merge, show the gap-closure verification re-asking the triggering question and closing the gap when the answer now cites the merged document.

### Part 3: Security and Governance (Won't Leak)

Demonstrate the governance properties:

- **Auth fails closed**: authentication is required unless `AUTH_REQUIRED=false` is explicitly set. A blank or misspelled value keeps auth on.
- **Scoped authorization**: global scopes (`read:knowledge`, `ask:knowledge`, `manage:jobs`) gate coarse capabilities; flow-scoped capabilities (with `KNOWLEDGE_ROLE_GRANTS`) gate per-flow access to proposals, gaps, and sources.
- **No user identities stored**: questions are logged without a user identifier.
- **Provider choice**: the prospect can choose an AI provider that matches their data-governance policy, including self-hosted models.

## Deployment Models

### Self-Hosted (Docker Compose)

The default deployment. A single Docker Compose file runs Postgres, the API, watcher, web console, and optionally the MCP HTTP server. Suitable for production-like demos, internal showcases, and single-host deployments. The container image is hardened (multi-stage build, non-root user, production-only dependencies).

### Self-Hosted (Host-Based Development)

Postgres via Docker; the API, watcher, and web console run on the host with `npm run dev:*` commands. Suitable for evaluation and development.

### Key Configuration Decisions for the Prospect

The prospect must make the following choices:

1. **AI provider**: select from openai-compatible, azure-openai, codex, or claude, aligned with data-governance requirements.
2. **Embedding provider**: configured independently of chat (e.g. DeepSeek for chat, OpenAI for embeddings).
3. **Storage backend**: Postgres for production (supports hybrid vector + keyword retrieval) or in-memory for evaluation.
4. **Authentication**: configure Auth0 with scoped authorization, or disable for local evaluation.
5. **Source and destination repositories**: configure KNOWLEDGE_SOURCES, KNOWLEDGE_DESTINATIONS, and KNOWLEDGE_FLOWS.

## The Maintenance Flywheel

The product's primary value proposition is that knowledge bases improve themselves over time. This is the flywheel to articulate to prospects:

1. **Questions are asked** via the web console, API, or MCP tools.
2. **Weak answers and gaps are detected** — low-confidence answers, follow-up searches that return empty, manual gap flags, and unhelpful feedback on confident answers all create gap candidates.
3. **Gaps are clustered** — the reconciler groups related gaps by embedding similarity, then runs a model-driven reshape to propose merges, splits, or dismissals.
4. **Proposals are drafted** — source-grounded AI jobs generate Markdown documents that address each cluster, with per-claim provenance tracked.
5. **Proposals are published as PRs** — the watcher commits the Markdown to a branch and opens a pull request.
6. **Humans review and merge** — the only step that changes the source of truth.
7. **Gap closure is verified** — the merge cascade re-asks the triggering questions against the updated index. Gaps are resolved only when confident answers now cite the merged document.

This loop runs on two schedules: the gap-to-PR reconciler (~10 minutes) for demand-driven improvements, and the patrols (hourly) for proactive correctness and editorial maintenance.

## Seeding a New Knowledge Base

For prospects starting fresh, Magpie is self-seeding: it explores the flow's source repositories and proposes a document plan without requiring a human to specify topics. The flow:

1. `POST /api/flows/:flowId/outline` enqueues a source-grounded planning job.
2. The agent explores the source repositories and proposes a document plan with charter, persona, and individual document items.
3. The plan is reviewed and edited in the console (or via MCP's `kb_outline` tool).
4. On approval, one `draft_seed_document` job is enqueued per item, producing clusterless proposals that converge on the same PR review gate.

There is no topic field — the planning agent explores the flow's sources and plans the whole flow, scoped by the flow's charter when configured.

## Common Sales Scenarios

### Scenario 1: Product Documentation Maintenance

The prospect maintains a product documentation repository in Git. Documentation goes stale because engineers are too busy to update it, and users complain about outdated answers.

**Approach**: Frame Magpie as a documentation assistant that never writes to main without review. It detects knowledge gaps from real user questions, drafts updates, and opens PRs. The maintenance patrols (correctness and editorial) proactively verify documents against source code and suggest improvements.

### Scenario 2: Security Questionnaires / Compliance Documentation

The prospect answers frequent security questionnaires or maintains compliance documentation drawn from multiple internal sources (codebases, policies, architecture docs).

**Approach**: Magpie's source-grounded drafting pulls from all configured source repositories. The agent explores the sources directly and composes answers that reference the governing documents. The per-claim provenance track records which source file grounds each claim. Because Magpie is provider-neutral, the prospect can run it against a self-hosted model or Azure OpenAI within their data boundary.

### Scenario 3: Internal Knowledge Base for Engineering Teams

The prospect has an internal wiki or knowledge base in Markdown but cannot keep it current. Engineers ask questions in Slack or chat tools but the answers are not captured.

**Approach**: Deploy Magpie with MCP transport so engineers can ask questions directly from their AI agent tools (Claude Code, VS Code). The answers are cited and grounded. Weak answers become gap candidates that automatically generate documentation PRs. The prospect gets a self-improving knowledge base without asking engineers to write docs.

## Objection Handling (Summary)

The dedicated "Handling Customer Objections" document covers responses in full. Key themes for the sales conversation:

- **"We already have an internal knowledge base"** — Magpie is a complement, not a replacement. It adds gap detection, proposal generation, and maintenance patrols on top of any Markdown-based KB.
- **"Magpie seems too complex to integrate"** — the architecture is deliberately simple: API enqueues jobs, watcher completes them, Postgres stores state. The queue-only model means Magpie never blocks on a model call and the watcher has no database access.
- **"We are worried about AI quality"** — every answer is cited; confidence is logged; grounding verification strips unsupported claims; mandatory human review gates all merges. The auto-drafting produces per-claim provenance so reviewers can verify every claim against its source.
- **"What if our data leaves our control?"** — provider-neutral design means the prospect chooses where model calls go. Self-hosted models (vLLM, Ollama) via `openai-compatible` keep everything in-house. The watcher has no egress of its own.

## Competitive Positioning

Markdown Magpie is not a knowledge base — it is a knowledge maintenance system that operates on top of an existing Git-backed Markdown KB. Key differentiators:

- **Queue-only AI architecture**: the API never calls a chat model inline. This is unique among documentation tools and directly enables provider-neutrality.
- **MCP-native**: knowledge is accessible through AI agents (Claude Code, Cursor, VS Code) without a custom integration.
- **Provenance tracking**: every claim in an AI-drafted document is linked to the source file that grounds it.
- **Mandatory human review**: no AI output reaches production without a human merging a PR. This is a first-class architectural invariant, not a configurable option.
- **Verification loop**: merged proposals are re-verified by re-asking the triggering questions, so a merge that does not actually close the gap reopens it for another draft.
- **Provider-neutral**: the product imposes no model vendor. The operator chooses the provider and can switch at any time.

## Pricing Context

Markdown Magpie is open-source (MIT) and can be self-hosted at no licence cost. The costs are:

- **Infrastructure**: Postgres instance, compute for API/watcher/web (can run on a single host for small deployments).
- **AI provider costs**: model API usage proportional to question volume and maintenance frequency.
- **Git hosting**: standard GitHub, GitLab, or Azure DevOps costs for destination repositories.

For managed/cloud deployments, refer to the Magpie Pricing and Plans document.

## Technical Proof Points for Sales Engineers

These architecture facts support technical credibility in sales conversations:

- **Rate limiting and AI cost controls**: two independent layers protect against unbounded spend. Per-principal rate limiting (fixed-window counters in Postgres) limits ask/trigger endpoints. A global in-flight AI-job cap (default 20 concurrent jobs) with a reserved interactive pool (default 5 slots) ensures asks are never starved by maintenance work.
- **Structured logging and OpenTelemetry**: all services log JSON via pino. OpenTelemetry (vendor-neutral, off by default) provides distributed tracing across API→job→watcher→callback with W3C traceparent propagation.
- **Embedding-model versioning**: stored vectors carry a model stamp. On model switch, the query-time guard only matches sections with the configured model's stamp, preventing silent vector-space mismatches. Unmatched sections remain reachable through keyword search.
- **Per-claim provenance (phase 2)**: draft outputs carry a structured `provenance` array mapping each substantive claim to the source file that grounds it. The document body contains no repository paths. Merged proposals become permanent provenance events for their target paths.
- **Stale-PR regeneration**: a published proposal whose branch conflicts with main is automatically re-drafted from the new base tip. An approved PR is never rewritten, and a per-proposal retry cap stops structural conflicts from looping.
- **Sparse-flow bootstrap**: a per-flow hourly task auto-proposes a seed plan for any flow with sources but a near-empty KB. Dismissal is sticky per source configuration — a human "no" is only re-litigated when the sources change.

## The Buyer's Journey in Summary

| Stage | Activity | Key Message |
|---|---|---|
| Awareness | Prospect encounters stale or untrusted documentation problem | "Knowledge bases rot without usage-driven maintenance" |
| Interest | Prospect evaluates cited-answer systems or AI documentation tools | "Magpie answers with citations, detects gaps, and self-improves" |
| Evaluation | Technical demo and proof of concept | "Won't lie, won't leak, won't rot — provider-neutral, Git-backed, human-reviewed" |
| Commitment | Self-hosted pilot or managed deployment | "Deploy in an afternoon; the maintenance loop runs itself" |
| Expansion | Add more knowledge flows and patrol schedules | "Magpie scales across teams and repositories with per-flow isolation" |