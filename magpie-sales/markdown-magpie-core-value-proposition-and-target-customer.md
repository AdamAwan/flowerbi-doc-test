---
title: Markdown Magpie — Core Value Proposition and Target Customer
status: draft
---

# Markdown Magpie: Core Value Proposition and Target Customer

## Overview

Markdown Magpie is a **vendor-neutral knowledge maintenance system** for Git-backed Markdown documentation. It helps teams keep their documentation accurate, complete, and up-to-date by automatically answering questions with citations, detecting knowledge gaps, proposing Markdown edits, and raising pull requests for human review. The system is designed to be **cheap & yours** — running on your own infrastructure with no vendor lock-in and no data leakage.

## Core Value Proposition

Markdown Magpie’s promise is summarised in three phrases used in its pitch deck:

- **Won’t lie** — Every answer is grounded in the team’s own documentation and comes with citations to exact file paths, headings, and commits. When the knowledge base is insufficient, the system abstains and flags the gap rather than fabricating an answer.
- **Won’t leak** — Your documentation, questions, and generated proposals stay within your own infrastructure. No data is shipped to third-party services unless you explicitly configure an external AI provider (and even then, only minimal context is shared).
- **Won’t rot** — The system continuously monitors documentation health. It detects gaps, consolidates overlapping documents, splits bloated ones, and proposes updates — all through reviewable pull requests. Your knowledge base stays fresh without manual upkeep.

Beyond these principles, the value proposition includes:

- **Automatic gap detection and resolution** — Questions that receive low-confidence answers are automatically logged, clustered, and turned into draft Markdown proposals. This closes the loop between “we don’t document that” and “now we do.”
- **Vendor‑neutral design** — The core packages define interfaces for chat, embeddings, git sync, and job execution. Concrete adapters can target local services, Docker, Azure, GitHub, GitLab, OpenAI-compatible APIs, or any other provider. No hard dependency on one model vendor.
- **Human‑in‑the‑loop workflow** — All proposed changes go through pull requests. Maintainers review, edit, and merge before any documentation is updated. The system never writes directly to the main branch.
- **Low operational overhead** — Runs on Docker Compose for single‑host deployments, with optional managed cloud (Azure) for scaling. No proprietary databases or exotic runtimes.

## Target Customer

The primary target customer is **any team that maintains a body of Markdown documentation in a Git repository** and wants to:

- Reduce the time spent answering repetitive questions (e.g. “How do I rollback a hotfix?”) from team members, support, or new hires.
- Ensure documentation stays accurate and comprehensive as the product, process, or team evolves.
- Eliminate stale or neglected documentation without adding a manual editorial cycle.
- Keep sensitive or proprietary documentation behind their own firewall.

Concrete examples of target users:

- **Internal platform or DevOps teams** documenting runbooks, deployment procedures, and troubleshooting guides.
- **Product engineering teams** maintaining API docs, user guides, or developer onboarding material.
- **Customer success or support teams** with an internal knowledge base they need to keep current.
- **Open‑source projects** that want to automatically surface gaps in their contributor documentation.
The solution is designed for teams of moderate size (e.g. 5–50 contributors) who already use Git and Markdown and are comfortable with pull‑request workflows. It is less suited to teams that publish documentation exclusively through a CMS or wiki, or those without a Git‑based editing process.

## How It Works (Briefly)

1. **Sync** — The system mirrors a Git repository of Markdown files (the “knowledge base”).
2. **Index** — It parses frontmatter, splits documents by heading, and indexes sections for both keyword and vector retrieval.
3. **Answer** — When a question is asked (via API, MCP, or web UI), it retrieves relevant sections and synthesises a cited answer using a configurable AI provider.
4. **Detect gaps** — Low‑confidence answers are logged and clustered into knowledge‑gap topics.
5. **Propose fixes** — The system generates draft Markdown additions or edits and opens a pull request for human review.
6. **Maintain** — Scheduled jobs (fix-patrol, improve-patrol, source‑change sync) keep the knowledge base correct and tidy via rolling cursor checks.

## Background and Rationale

This article was drafted to address the question “What is Magpie’s core value proposition and who is the target customer?” — a gap identified as having no source material in the existing knowledge base. The content is synthesised from the project’s own documentation:

- [`README.md`](https://github.com/AdamAwan/markdown-magpie/blob/main/README.md) describes the product loop and the “won’t lie, won’t leak, won’t rot” narrative.
- [`presentation/README.md`](https://github.com/AdamAwan/markdown-magpie/blob/main/presentation/README.md) documents the pitch deck and its core messages.
- [`docs/architecture.md`](https://github.com/AdamAwan/markdown-magpie/blob/main/docs/architecture.md) details the provider‑neutral design and the human‑in‑the‑loop workflow.
- [`docs/ingestion.md`](https://github.com/AdamAwan/markdown-magpie/blob/main/docs/ingestion.md) explains how sources and destinations are configured, reinforcing the “yours” aspect.

This article is intended for the `magpie-sales-docs` knowledge base and should be reviewed and iterated upon by the product team.