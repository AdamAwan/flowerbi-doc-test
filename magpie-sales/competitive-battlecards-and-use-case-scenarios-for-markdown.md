---
title: Competitive Battlecards and Use Case Scenarios for Markdown Magpie
status: draft
---

# Competitive Battlecards and Use Case Scenarios for Markdown Magpie

This document complements the existing [Competitor Landscape and Differentiation](./competitor-landscape-and-differentiation.md) document by providing structured battlecards for the most common competitive situations prospects raise, alongside practical use case scenarios that illustrate Magpie's value in context.

---

## Battlecard 1: In-House Knowledge Base Solution

**Who:** A team building their own documentation maintenance tooling (scripts, custom bots, CI workflows).

**Magpie's advantage:** Magpie provides the same automation in a ready-to-deploy, Git-backed system without the ongoing maintenance burden of a custom build. The key architectural properties that a custom build would need to re-implement are:

- **Provider-neutral job queue.** Magpie models all generative AI work as durable pg-boss jobs on Postgres. The API enqueues work; a separate watcher process claims it, calls the configured provider, and posts the result back. The API never calls a chat model inline, so no single provider lock-in is hard-coded.
- **Automatic gap detection and clustering.** Low-confidence answers, unanswered follow-up searches, and manual flags are surfaced as gap candidates. An embedding-based phase-1 pre-clusterer groups near-identical gaps by cosine similarity; a subsequent reshaping step (propose merge/split/dismiss, then critic-confirm) refines the clusters semantically.
- **Automated proposal generation and publication.** Gaps are drafted into Markdown proposals through source-grounded AI jobs, published as pull requests (or local-git branches), and tracked through to merge. Publication is enqueue-only, idempotent, and gated by the human review requirement.
- **Gap-closure verification.** Merging a proposal does not blindly resolve its gaps. Each triggering question is re-asked against the freshly re-indexed knowledge base; only a confident answer that cites the merged document closes the gap.
- **Scheduled maintenance patrols.** Hourly correctness and editorial patrols fan out into verify, correct, and improve jobs that keep the knowledge base healthy without human intervention.

**Key differentiator:** Magpie is a complete system, from ingestion through gap detection, drafting, publication, and post-merge verification. A custom build needs to replicate every layer — job queue, embedding pipeline, gap clustering, proposal drafting, Git publishing, PR tracking, and closure verification — before it delivers any value.

**Supporting material:** See [ROI and Business Case for Markdown Magpie](./roi-and-business-case-for-markdown-magpie.md) for a quantified comparison against the cost of building and maintaining custom tooling.

---

## Battlecard 2: Existing Wiki or CMS Platform

**Who:** A prospect using Confluence, Notion, GitBook, or a similar platform to host their documentation.

**Magpie's advantage:** Magpie is not a replacement for the authoring environment — it is a knowledge maintenance layer that sits alongside it. It indexes Git-backed Markdown, answers questions with citations, detects gaps automatically, and generates pull requests for human review.

- **Gap detection is automatic, not manual.** While a wiki relies on users to notice and report missing content, Magpie records every low-confidence answer, every unanswered follow-up search during the agentic retrieval loop, and every "unhelpful" feedback on a confident answer — all as structured gap candidates that cluster and trigger proposals.
- **Proposals land as pull requests, not new pages.** Drafted content goes through the same review process as any other code change. The mandatory human merge is the primary prompt-injection control.
- **Verification closes the loop.** After a proposal merges, Magpie re-asks the triggering questions to confirm the new content actually answers them. If the gap persists, it re-opens — or, after repeated failures, surfaces as a parked question awaiting human attention.
- **Scheduled patrols maintain freshness.** The hourly correctness and editorial patrols verify claims against source repositories, correct unsupported statements, and improve thin but otherwise healthy documents.

**Key differentiator:** Wikis are authoring platforms; Magpie is a maintenance engine. It does not compete on authoring experience — it competes on automatically finding what is missing, drafting what should be there, and verifying that published fixes actually resolved the gap.

---

## Battlecard 3: Generic AI Chatbot / RAG System

**Who:** A prospect using a general-purpose RAG (retrieval-augmented generation) system or custom chatbot to answer questions from their documentation.

**Magpie's advantage:** A pure RAG system answers questions from whatever content exists. Magpie answers questions AND improves the content when the answer falls short.

- **Answers are a by-product, not the product.** Every question asked through Magpie feeds the maintenance pipeline. Low-confidence and unanswered questions become gap candidates. Unhelpful feedback on confident answers raises feedback gaps. Each gap clusters with related ones and triggers a proposal draft.
- **The gap-to-PR pipeline is automated.** A scheduled reconciler (~10 min cadence) clusters open gaps, drafts proposals for uncovered clusters, publishes them as pull requests, and advances proposals as their PRs merge or close — all in one task.
- **Every claim carries provenance.** Drafted documents never embed repository paths or source names in their text. Instead, each substantive claim is recorded in a structured provenance array on the proposal, mapped to the source files that ground it. This avoids leaking internal source locations into answers served to end users.
- **Post-merge verification ensures closure.** Unlike a RAG system that simply answers from whatever is indexed, Magpie verifies that a merged proposal actually closed the gap it was supposed to fix.

**Key differentiator:** A RAG chatbot tells users what the docs say. Magpie tells users what the docs say, detects when they do not say enough, drafts the missing content, publishes it for review, and confirms the fix worked.

---

## Battlecard 4: Build-Your-Own with LLM APIs

**Who:** A team with engineering capacity who is considering wiring LLM APIs directly to their documentation repository.

**Magpie's advantage:** Magpie already implements the full pipeline — developer hours spent integrating APIs, building job queues, parsing Markdown, managing Git operations, and maintaining prompt contracts are hours not spent on the actual documentation.

- **Provider neutral, not provider dependent.** Magpie supports OpenAI-compatible, Azure OpenAI, Codex, and Claude providers via a watcher model. The same pipeline works with self-hosted models (vLLM, Ollama) pointed at an OpenAI-compatible base URL.
- **All AI work is durable and retryable.** Jobs are persisted in pg-boss and survive process restarts. The watcher model includes heartbeat, timeout, and structured error reporting. Completion is never re-billed — the output is persisted before any side effect runs, and a failed side effect triggers a replay, not a regeneration.
- **Token usage and cost tracking is built in.** The watcher reports token usage and execution identity (provider + model) on every completion. Cost is priced at read time against an operator-supplied pricing table, covering priced, unpriced, and unmetered states.
- **Embedding carry-forward avoids re-computation.** On re-index, sections whose content and heading are byte-identical to the stored row keep their existing embedding. An unchanged corpus costs zero embedding calls.

**Key differentiator:** The build-vs-buy decision here is not about whether LLMs can write documentation — it is about whether the team should invest in the pipeline that connects LLMs to Git repositories, manages the job queue, handles retries and failures, tracks provenance, and verifies post-merge outcomes. Magpie ships that pipeline ready-made.

---

## Use Case Scenario 1: Bootstrapping a New Product's Documentation

**Scenario:** A product team is launching a new project with a Git repository containing source code but no documentation. They want a knowledge base to answer onboarding questions and to grow automatically as the product evolves.

**How Magpie fits:**

1. **Configure a flow.** The team configures their source repository (e.g. a `docs` folder in the product repo) as both a source and a destination, and sets up a knowledge flow linking them.
2. **Seeding bootstraps the KB.** Magpie's `seed_bootstrap` scheduled task detects the near-empty destination (fewer than 3 documents by default) and automatically enqueues an `outline_flow_seed` job. The AI agent explores the source repository and proposes a document plan — a charter, a persona, and a set of document items — all scoped by the flow's sources.
3. **Human reviews the plan.** The team sees the proposed plan on the Seed page, edits the charter and persona, approves items (or dismisses irrelevant ones), and approves the plan.
4. **Drafting begins.** Each approved item becomes a `draft_seed_document` job. The agent explores the source code and existing docs, then drafts complete Markdown documents with per-claim provenance.
5. **Proposals become pull requests.** The drafted documents are published as pull requests on the team's repository. Someone on the team reviews, merges, and the knowledge base is live.
6. **Ongoing maintenance.** As team members ask questions, Magpie logs answers, detects gaps, and automatically drafts proposals to fill them. The process repeats without manual effort.

---

## Use Case Scenario 2: Maintaining Stale Documentation in an Established Codebase

**Scenario:** An established project has a docs folder with hundreds of Markdown files, many of which are outdated. Answers to developer questions are increasingly inaccurate because the documentation has not kept pace with code changes.

**How Magpie fits:**

1. **Index the existing docs.** Magpie syncs the repository, parses each document into sections, and indexes them with embeddings (when configured) for hybrid keyword-vector retrieval.
2. **Answer questions with citations.** Developers ask questions through the web UI, the API, or the MCP server (`kb_ask`). Answers cite specific sections. When the agentic retrieval loop finds no supporting material for a follow-up search, it records a `followup` gap.
3. **Gap clustering.** The reconciler clusters related gaps using embedding similarity (phase-1 assignment) and semantic critic review (the reshape step). A cluster of related "API endpoint not documented" gaps becomes a single proposal target.
4. **Automated proposals.** The reconciler drafts proposals for uncovered clusters, publishes them as PRs, and tracks their status. Each proposal carries provenance — the source files that support each claim — so reviewers know where the AI got its information.
5. **Stale-PR regeneration.** If a proposal's PR goes stale because `main` advanced, Magpie's snapshot refresh detects the conflict and auto-regenerates the proposal against the new base — up to a cap, and only until the PR is approved.
6. **Verification locks in improvements.** After merge, Magpie re-asks each triggering question. If the answer is now confident and cites the new document, the gap resolves. If not, the gap re-opens with a note explaining what the merged document still misses.

---

## Use Case Scenario 3: Multi-Team Documentation with Per-Flow Isolation

**Scenario:** An organisation has several product teams, each maintaining its own documentation repository. Different teams need to see only their own proposals and gaps, and only certain people should be able to edit each knowledge area.

**How Magpie fits:**

1. **Configure multiple flows.** Each team's repository is configured as a separate knowledge flow with its own sources, destination, and optional charter and persona.
2. **Flow-scoped authorisation.** With `KNOWLEDGE_ROLE_GRANTS`, an operator maps IdP role names to per-flow capabilities. A "customer-docs-curators" role might have `read` and `manage` on the customer docs flow, while an "engineering-askers" role has only `ask` on the engineering flow.
3. **Independent maintenance.** Each flow has its own scheduled tasks — gap drafting, source sync, snapshot refresh, correctness patrol, editorial patrol, and seed bootstrap — all serialised per flow via Postgres advisory locks.
4. **Per-flow cost attribution.** AI token usage and cost are attributed per flow (from the `flowId` on job inputs), so each team sees its own AI spend on the Insights pages.
5. **Isolation without multi-tenancy.** The application remains single-tenant by design — one deployment serves one organisation — but per-flow capabilities ensure that a human-curator for HR documentation never accidentally sees or edits engineering proposals.

---

## Use Case Scenario 4: Compliance-Conscious Deployment with Self-Hosted AI

**Scenario:** A regulated industry team needs to maintain documentation from proprietary source code, but data governance policy prohibits sending code or proprietary content to third-party AI APIs.

**How Magpie fits:**

1. **Self-hosted AI provider.** The operator sets `AI_PROVIDER=openai-compatible` and points `OPENAI_COMPATIBLE_BASE_URL` at an in-house model server (vLLM, Ollama, or a fine-tuned model). The embedding endpoint is similarly self-hosted or an on-premises Azure OpenAI deployment.
2. **No data egress.** The security review confirms that the application imposes no data egress of its own. Where data is stored (Postgres) and where it is sent (the configured AI provider) are determined entirely by operator configuration. With a self-hosted provider, no proprietary content leaves the deployment boundary.
3. **Mandatory human review.** The threat model's primary control is that no AI-generated content reaches the default branch without a human merging a pull request. Automated flows may draft, publish a branch, and open a PR, but the merge is always a human action on the Git host.
4. **Auth fails closed.** Authentication is required unless explicitly disabled. With Auth0 configured, API, web app, and MCP transports all validate bearer tokens. Flow-scoped authorisation layers on top of global scopes.
5. **Branch protection.** The operator enables branch protection on destination repositories, requiring at least one human approval and ensuring the bot account cannot approve or merge its own PRs.

---

## Use Case Scenario 5: MCP-Enabled Knowledge Base for AI Agents

**Scenario:** An organisation wants its internal AI coding agents to have direct access to the documentation knowledge base so they can answer questions, search content, and even propose new documentation — all through the Model Context Protocol.

**How Magpie fits:**

1. **Deploy the MCP server.** The @magpie/mcp package provides both stdio and Streamable HTTP transports. Agents connect via `.mcp.json` (Claude Code) or a client configuration pointing at the HTTP endpoint.
2. **Agent tools.** The MCP server exposes `kb_ask` (answer a question with citations), `kb_search` (search indexed sections), `kb_flows` (list available knowledge flows), `kb_feedback` (report helpful/unhelpful or flag a gap), `kb_citation` (fetch full cited sections), `kb_outline` (propose a seed plan), and `kb_seed` (approve a plan and enqueue drafts).
3. **Seed planning from agents.** An agent can call `kb_outline` with a flow id and optional notes — no topic needed. The tool explores the flow's source repositories and returns a complete document plan (charter, persona, items). The agent or a human reviews and approves via `kb_seed`.
4. **Authentication and scoping.** The Streamable HTTP transport acts as an OAuth protected resource. Per-tool scopes enforce access: `kb_search` requires `read:knowledge`, `kb_ask` requires `ask:knowledge`, and `kb_outline`/`kb_seed` require `manage:jobs`. On-behalf-of delegation allows the API to authorise as the end user rather than the shared MCP service identity.

---

## Summary of Competitive Positioning

Magpie occupies a distinct space: it is not a wiki, not a chatbot, and not a custom integration project. It is a knowledge maintenance system that:

- **Detects gaps automatically** from real user behaviour (low-confidence answers, unanswered follow-up searches, rejected answers).
- **Drafts proposals grounded in actual source repositories**, with per-claim provenance for reviewer confidence.
- **Publishes through the standard Git PR workflow**, keeping the mandatory human review gate that is the primary control against prompt injection.
- **Verifies that merged fixes actually resolved the gap**, re-opening or escalating to a human when they do not.
- **Maintains itself through scheduled patrols** that verify correctness and improve thin documents.
- **Is provider-neutral** — the same pipeline works with any OpenAI-compatible, Azure OpenAI, Claude, or Codex provider, including self-hosted models.

For the built-in ROI analysis, cost tracking capabilities, pricing details, and a full competitor landscape overview, refer to the companion documents already published in this knowledge base.