---
title: Markdown Magpie — Value Proposition and Target Customer
status: draft
---

# Markdown Magpie — Core Value Proposition and Target Customer

## Who We Are

Markdown Magpie is a **vendor-neutral knowledge maintenance system** for Git-backed Markdown documentation. It answers questions with citations, records where the knowledge base is weak, proposes Markdown changes, and raises pull requests for human review — all without locking your documentation into any single platform or AI provider. The system is designed to be **cheap & yours** — running on your own infrastructure with no vendor lock-in and no data leakage.

## The Core Value Proposition: "Won't Lie · Won't Leak · Won't Rot"

This three-word promise, carried through our pitch deck and product narrative, captures what Markdown Magpie delivers:

1. **Won't lie.** Every answer is grounded in your actual documentation. When we cannot find a reliable source, we say so — we abstain rather than fabricate. Citations point to exact file paths, headings, and commits so you can verify every claim.

2. **Won't leak.** Your knowledge stays in your Git repositories. We never train on your content, and our architecture is designed around local-first, self-hostable deployments. All embedding and chat can run against your own model endpoints (OpenAI-compatible or Azure OpenAI) or a local mock provider.

3. **Won't rot.** Markdown Magpie actively fights knowledge decay. It logs low-confidence answers and user feedback, clusters repeated gaps, generates proposed Markdown additions or edits, and opens pull requests for maintainers to review. The system keeps your docs fresh without waiting for a stale-content audit.

**Cheap & Yours.** The product is designed to be low-cost to operate (Docker Compose for single-host, optional Azure for scale) and fully under your control. There are no per-seat fees, data egress charges, or proprietary formats.

## The Product Loop

Markdown Magpie operates on a continuous maintenance cycle:

1. **Sync** a Git repository of Markdown docs.
2. **Parse** frontmatter and split documents by heading.
3. **Index** sections for keyword and vector retrieval.
4. **Answer** questions with citations to file paths, headings, and commits.
5. **Log** low-confidence answers and user feedback.
6. **Cluster** repeated gaps.
7. **Generate** proposed Markdown additions or edits.
8. **Open** pull requests for maintainers to review.

This loop means your knowledge base actively improves itself — the more questions it receives, the sharper its gap detection becomes, and the better the proposals it delivers.

## Target Customer

Markdown Magpie is built for **teams that live in Git** and rely on Markdown for documentation. The ideal customer falls into one of these profiles:

### 1. Engineering teams with product documentation

- They maintain a docs-as-code workflow (e.g., Docusaurus, MkDocs, or plain Markdown in a repo).
- They want an internal Q&A tool that answers accurately from their docs — not a public chatbot that hallucinates.
- They are tired of stale documentation and want automated gap detection and pull requests.

### 2. Open-source project maintainers

- They publish documentation in a GitHub/GitLab repo and want to ensure newcomers can find answers.
- They need a low-maintenance, self-hostable knowledge assistant that respects their existing contribution workflow.

### 3. Internal knowledge base curators

- They manage a private Git repository of operational runbooks, tribe knowledge, or policy documents.
- They require data sovereignty — no third-party training, no data leaving their infrastructure.
- They need to track what the team actually asks about and see where docs fall short.

### 4. AI-augmented documentation maintainers

- They already use Claude Code, Copilot, or similar tools but want a structured way to close knowledge gaps.
- They value the human-in-the-loop model: Markdown Magpie drafts proposals, a human reviews and merges.

Concrete examples of target users include:
- **Internal platform or DevOps teams** documenting runbooks, deployment procedures, and troubleshooting guides.
- **Product engineering teams** maintaining API docs, user guides, or developer onboarding material.
- **Customer success or support teams** with an internal knowledge base they need to keep current.
- **Open‑source projects** that want to automatically surface gaps in their contributor documentation.

The solution is designed for teams of moderate size (e.g. 5–50 contributors) who already use Git and Markdown and are comfortable with pull‑request workflows. It is less suited to teams that publish documentation exclusively through a CMS or wiki, or those without a Git‑based editing process.

## Key Differentiators

| Feature | What it means for you |
|---|---|
| **Vendor-neutral** | No lock-in to any AI provider, cloud, or documentation platform. Run with OpenAI, Azure, DeepSeek, mock, or your own model. |
| **Cited answers** | Every answer includes document ID, heading, and excerpt — not just a blurb. |
| **Gap-driven proposals** | The system doesn't just answer questions; it writes new documentation to fill gaps and opens a PR. |
| **Pull request workflow** | Changes go through the same review process as your code. No unsanctioned updates. |
| **Self-hostable** | Docker Compose for small deployments, Azure for managed scale. All data stays in your network. |
| **Granular access** | MCP client authentication with per-tool scopes (read, ask, feedback). Integrates with Auth0. |

## Use Cases That Drive Real Value

- **Onboarding new team members** — They ask questions against internal docs and get immediate, cited answers.
- **Keeping runbooks current** — When procedures change, the system detects gaps and proposes updates.
- **Open-source community Q&A** — A self-hosted knowledge base that answers accurately without exposing private data.
- **Preventing documentation drift** — Automated gap clustering flags missing topics before they become stale.

## How It Works (Detailed)

1. **Sync** — The system mirrors a Git repository of Markdown files (the “knowledge base”).
2. **Index** — It parses frontmatter, splits documents by heading, and indexes sections for both keyword and vector retrieval.
3. **Answer** — When a question is asked (via API, MCP, or web UI), it retrieves relevant sections and synthesises a cited answer using a configurable AI provider.
4. **Detect gaps** — Low‑confidence answers are logged and clustered into knowledge‑gap topics.
5. **Propose fixes** — The system generates draft Markdown additions or edits and opens a pull request for human review.
6. **Maintain** — Scheduled jobs (fix-patrol, improve-patrol, source‑change sync) keep the knowledge base correct and tidy via rolling cursor checks.

## Background and Rationale

This article was originally written to address the question “What is Magpie’s core value proposition and who is the target customer?” — a gap identified as having no source material in the existing knowledge base. The content is synthesised from the project’s own documentation:

- [`README.md`](https://github.com/AdamAwan/markdown-magpie/blob/main/README.md) describes the product loop and the “won’t lie, won’t leak, won’t rot” narrative.
- [`presentation/README.md`](https://github.com/AdamAwan/markdown-magpie/blob/main/presentation/README.md) documents the pitch deck and its core messages.
- [`docs/architecture.md`](https://github.com/AdamAwan/markdown-magpie/blob/main/docs/architecture.md) details the provider‑neutral design and the human‑in‑the‑loop workflow.
- [`docs/ingestion.md`](https://github.com/AdamAwan/markdown-magpie/blob/main/docs/ingestion.md) explains how sources and destinations are configured, reinforcing the “yours” aspect.

This article is intended for the `magpie-sales-docs` knowledge base and should be reviewed and iterated upon by the product team.

## Summary

Markdown Magpie is for any team that wants a **truthful, private, self-maintaining documentation system**. It transforms a static Markdown repo into a living knowledge base that learns from real questions and actively improves itself — all without vendor lock-in or recurring per-user fees.