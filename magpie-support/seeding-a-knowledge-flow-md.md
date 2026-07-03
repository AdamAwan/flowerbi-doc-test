---
title: Seeding a Knowledge Flow in Markdown Magpie
status: draft
---

# Seeding a Knowledge Flow in Markdown Magpie

Seeding a knowledge flow means **pre-populating it with a curated set of starter documents**, rather than relying solely on the demand-driven pipeline that evolves knowledge from real user questions. In Markdown Magpie, seeding is a direct authoring path that bypasses gap clustering and intent inference, letting you bootstrap a new flow or add a whole new area of knowledge to an existing one in a single step.

## When to Use Seeding

- **Setting up a new flow** – before users ask questions, seeds give the system authoritative answers.
- **Establishing a baseline** – seed common or foundational topics so the knowledge base is useful from day one.
- **Controlling initial answers** – you define exactly what the flow should cover, avoiding gaps until later contributions add nuance.

Seeding is not meant to replace the evolutionary gap pipeline; it is a deliberate bootstrapping mechanism. After seeding, the normal question → gap → proposal cycle continues to cover emergent needs.

## How to Configure a Seed Source

Seeding is configured via the API rather than environment variables. The primary endpoint is:

```http
POST /api/flows/:flowId/seed
Content-Type: application/json

{
  "items": [
    { "title": "Billing overview", "coverage": ["what billing is", "the plans"] },
    { "coverage": ["refund policy", "how to request a refund"] }
  ]
}
```

Each *item* has an optional `title` and a required `coverage` array of points the document should cover. When a title is omitted, one is generated automatically.

To **generate an outline** first (instead of writing items by hand), use:

```http
POST /api/flows/:flowId/outline
Content-Type: application/json

{
  "topic": "Refund handling",
  "notes": "focus on partial refunds"
}
```

This triggers an `outline_flow_seed` AI job that proposes a list of items grounded in the flow's existing documentation. The output is a job result you can inspect and then use as the basis for your own seed call.

Seeding is *not* configured via the `KNOWLEDGE_SOURCES` environment variable – that variable is used to define the git or local paths that the flow indexes for ongoing operations. Seeds are strictly inline content in API requests.

## How Seeding Integrates with Indexing and Embedding

When you submit a seed, the API enqueues `draft_seed_document` AI jobs for each item. These jobs draft Markdown content grounded in the flow's existing source material. On completion, the API creates a **clusterless proposal** – a proposal that does not originate from a gap cluster. The proposal then follows the standard review and publication flow:

1. The proposal is stored with status `draft`.
2. A human reviews it in the console (or via API) and changes its status to `ready`.
3. The human triggers publication (`POST /api/proposals/:id/publish`), which pushes the document to a new branch (and optionally opens a pull request for git-backed destinations).
4. After the pull request is merged (or the branch is accepted), the new content becomes part of the indexed knowledge base.

**Seeds are not ingested as high-priority knowledge automatically.** They must pass through the same human gate as any other proposal. The Markdown Magpie source documentation states: “seeding still ends at a reviewable pull request — the same human gate as everything else.” Only after merge and a subsequent re-index will the seed content be searchable and citable in answers.

Embedding of the new content happens when you trigger a re‑index of the flow (e.g. `POST /api/knowledge/repositories/index`). The indexed sections are then chunked and embedded inline by the API (the API holds the embedding provider). There is no special priority or separate embedding pass for seeds.

## Step-by-Step Process

1. **Identify what to seed** – either a topic (for outline generation) or a set of explicit items.
2. **Optionally generate an outline** – call `POST /api/flows/:flowId/outline` to get a machine‑proposed list of items.
3. **Review and edit the outline** – ensure the proposed items match your intent.
4. **Submit the seed** – call `POST /api/flows/:flowId/seed` with your final `items` array.
5. **Wait for drafting to complete** – the API returns enqueued job ids. Poll `GET /api/jobs/:id/wait` for each job until it finishes.
6. **Inspect the resulting proposal** – read the proposal via `GET /api/proposals` or in the web console.
7. **Review and approve the proposal** – update its status to `ready` (`POST /api/proposals/:id/status { "status": "ready" }`).
8. **Publish the proposal** – call `POST /api/proposals/:id/publish`. This creates a branch and, for GitHub destinations, a pull request.
9. **Merge the pull request** – the branch or PR is merged by a maintainer (or automatically if configured).
10. **Re‑index the flow** – trigger a fresh index (`POST /api/knowledge/repositories/index`) so the new content is searchable and embedded.
11. **Verify** – ask a question about the seeded topic (`POST /api/ask`) and confirm the answer cites the new document.

## Best Practices

- **Choose seed documents that cover the most common queries** – think about the questions new users will ask first.
- **Avoid duplicates** – if a seed document overlaps an existing open proposal on the same path, the system folds the seed into that proposal rather than creating a new one. This reduces review overhead.
- **Keep seeds up to date** – treat seeded documents as living content; update them through the normal proposal cycle when the product changes.

## Tips for Effective Seeding

- **Use clear titles and headings** – the title becomes part of the file name (e.g. `<title-slug>.md`) and helps retrieval rank the document higher for relevant queries.
- **Structure documents for retrieval** – break content into well‑named sections with `##` headings. The Markdown parser splits documents by heading, producing sections that can be independently retrieved. Good structure means the right section answers the right question.
- **Balance breadth and depth** – a seed that covers a topic broadly but lacks detail may produce shallow answers. Conversely, a very deep document on a narrow topic may be missed for broader queries. Strike a balance that matches expected usage.

## Common Pitfalls

- **Over‑seeding causing noise** – seeding too many documents in one go floods the review queue with proposals. Start with a small set of high‑value topics and iterate.
- **Forgetting to re‑index after adding seeds** – even after merge, the new content does not appear in answers until the flow is re‑indexed. Always trigger an index after a seed publication series.
- **Mixing seeds with live content** – seeds are not automatically part of the knowledge base until published and merged. During review they are invisible to the answer system, so do not rely on them until after indexing.

## Summary

Seeding in Markdown Magpie is a deliberate, human‑reviewed way to bootstrap knowledge bases. It uses an API‑driven workflow that drafts documents, creates proposals, and funnels them through the same review and publication pipeline as gap‑driven content. When used thoughtfully – with clear topics, balanced scope, and proper post‑merge indexing – seeding establishes a solid foundation for accurate, cited answers from day one.