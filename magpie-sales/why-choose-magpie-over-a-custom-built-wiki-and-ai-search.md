---
title: Why Choose Markdown Magpie Over a Custom-Built Wiki and AI Search
status: draft
---

# Why Choose Markdown Magpie Over a Custom-Built Wiki and AI Search?

Prospects sometimes tell us they've already built their own internal wiki with AI-powered search. That's a legitimate investment, but it often lacks the maintenance loop that keeps knowledge bases accurate and up to date. Here's why Markdown Magpie delivers more value than a custom solution.

## The Real Problem: Knowledge Decay, Not Just Search

A custom wiki with AI search solves *retrieval* — it finds content. But content gets stale. Teams drift, processes change, and old answers mislead. Magpie completes the loop: it **detects gaps**, **proposes updates**, and **raises pull requests** for human review. The result is a knowledge base that stays correct, not just findable.

## What Magpie Does That a Custom Wiki Doesn't

| Capability | Custom Wiki + AI Search | Markdown Magpie |
|---|---|---|
| **Git-native source of truth** | Usually a database or CMS with no version history | Markdown in Git; every change is a commit, every proposal is a branch |
| **Cited answers** | Often a raw AI summary with no provenance | Every answer links to exact file paths, headings, and commits |
| **Confidence scoring** | Binary: found or not | Three-tier confidence (high/medium/low) derived from retrieval relevance |
| **Gap detection** | Manual or non-existent | Automatically logs low-confidence answers and clusters repeated gaps |
| **Proposal generation** | You write the fix yourself | AI drafts Markdown additions; you review and merge |
| **Pull request workflow** | No structured approval | Changes go through Git branches, PRs, and reviews |
| **Vendor-neutral** | Locked to your stack | Works with any Git host, any OpenAI-compatible API, any LLM |
| **Hybrid search** | Often keyword-only or vector-only | Reciprocal Rank Fusion of vector + keyword for best results |
| **Scheduled maintenance** | None | Patrol lenses (fix-patrol, improve-patrol) run on rolling cursors, verifying claims, deduplicating, splitting bloated files, and growing fine-but-thin documents with source-backed content. Crunch consolidates overlapping docs; source-change sync rewrites outdated content. |

## Key Differences Explained

### 1. Your Wiki Doesn't Learn From Its Own Weaknesses

Magpie's **question logging and gap clustering** turn every unanswered question into an improvement opportunity. When the system answers with low confidence, it records the gap. Repeated gaps are clustered and a proposal is drafted automatically. Your custom wiki would need to build all of that from scratch.

### 2. Proposed Changes Are Safe and Reviewable

Magpie never writes directly to your knowledge base. Every proposal is committed to a new Git branch and raised as a pull request. Your team reviews, edits, and merges. No surprise edits, no broken content. This **human-in-the-loop** model ensures accuracy while still automating the heavy lifting.

### 3. You Own Your Data, Not a Proprietary CMS

Magpie stores nothing in a proprietary format. Everything is standard Markdown in a standard Git repository. You can switch search providers, embedders, or LLMs without migrating data. Compare that to a custom wiki tied to a single database schema and a single search backend.

### 4. Hybrid Search Without Vendor Lock-In

Many custom solutions pick one search method. Magpie uses **Reciprocal Rank Fusion** to combine pgvector vector search with keyword scoring (heading and content matches). You can swap the embedding provider (OpenAI, Azure, or any OpenAI-compatible API) without changing the retrieval logic. AI providers are behind `AgentRunner` adapters; you can switch from OpenAI to Azure to a local model without changing the knowledge pipeline.

### 5. Scheduled Tidying Prevents Bloat

Over time, knowledge bases accumulate overlapping documents and long pages. Magpie's **Crunch** feature analyzes your destination repository and proposes splits and consolidations. In addition, Magpie runs **patrol lenses** on rolling cursors — checking the least-recently-visited files each run — with a fix-patrol that checks correctness (verify, dedupe, split) and an improve-patrol that grows fine-but-thin documents with source-backed additions. This keeps the knowledge base accurate and tidy without a whole-KB maintenance pass. Magpie's **source-change sync** automatically rewrites outdated content when the underlying source information changes, keeping docs in sync with evolving systems.

### 6. Cited, Honest Answers

Every answer comes with citations to the exact file path and heading. The AI is trained to say "I don't know" when it can't find reliable source material — no hallucination, no made-up content. This is part of Magpie's core promise: "Won't lie, won't leak, won't rot."

## Why Your Engineering Team Will Thank You

- **Markdown-native** — Your wiki stays in git. Your developers can edit with their favorite editor, review via PR, and keep history.
- **No proprietary data format** — Magpie reads and writes standard Markdown. You own your docs.
- **Self-hosted friendly** — Runs on Docker Compose or Azure. No vendor lock-in for the platform either.
- **Low operational overhead** — Postgres + pgvector is the only required infrastructure beyond the containers themselves.

## When a Custom Wiki Makes Sense

A custom wiki works well if:
- You have a dedicated team to maintain it (write content, fix gaps, prune old docs).
- You don't need AI-assisted gap detection or proposal generation.
- You're willing to build your own search, confidence scoring, and maintenance pipeline.

But if you want **continuous improvement** without a full-time doc team, Magpie's loop of **ask → detect → propose → review → merge** is far more efficient than building that pipeline yourself.

## The Bottom Line

Your custom wiki + AI search answered questions. Markdown Magpie **improves the knowledge base itself**. Your custom wiki gets you 50% of the way: storing and searching Markdown. Magpie delivers the other 50%: the closed-loop maintenance system that keeps that Markdown accurate, current, and easy to maintain. The friction of switching from your current Git-based wiki is minimal — Magpie treats Git as the source of truth — and the return on investment comes from the hours saved on manual gap-fixing and content upkeep. If you're tired of manually plugging holes that your AI can't fill, or of maintaining bespoke ingest pipelines that break with every model update, Magpie is the upgrade.

> "The Git repository is the source of truth for published knowledge. Postgres stores indexes, logs, metadata, proposals, and audit history." (Source: `docs/architecture.md`)

> "Won't lie, won't leak, won't rot" — from the Magpie pitch deck. (Source: `presentation/README.md`)

Don't just find information. Keep it right.

---

Ready to stop maintaining and start improving? [Try the demo](https://github.com/AdamAwan/markdown-magpie) or [contact us](mailto:info@markdownmagpie.com) for a guided walkthrough.
