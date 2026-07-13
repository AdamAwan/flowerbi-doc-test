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

Seeding is configured via the API and is **plan-centric**. Instead of submitting raw items directly, you first propose a plan and then approve it. The primary endpoint to propose a plan is:

```http
POST /api/flows/:flowId/outline
Content-Type: application/json

{
  "notes": "optional steer for this run"
}
```

This triggers an `outline_flow_seed` AI job that proposes a list of documents grounded in the flow's source repositories and existing documentation. **There is no topic** — the planning agent explores the flow's sources and plans the whole flow, scoped by the flow's charter when configured. The endpoint requires the `manage:jobs` scope (and `manage` on the target flow) and returns the enqueued job id.

On completion, the API persists a **seed plan** (`SeedPlan`, status `proposed`) containing proposed items, a rationale, and an optional proposed charter/persona. You can review the plan via:

```http
GET /api/flows/:flowId/seed-plans
GET /api/seed-plans/:id
```

Each item has an optional `title`, an optional `targetPath` (an explicit destination-relative path such as `billing/overview.md`), a required `coverage` array of points the document should cover, and an optional `questions` array of motivating questions/prompts that give the drafter context. When a title is omitted, one is generated automatically. When a `targetPath` is given, the drafted document is written to that exact path rather than deriving one from the title.

The schema for each item is:

| Field | Type | Required | Purpose |
|-------|------|----------|---------|
| `title` | string | no | Document title; derived if absent |
| `targetPath` | string | no | Explicit destination-relative path |
| `coverage` | string[] | yes (≥1) | Points the document must cover |
| `questions` | string[] | no | Motivating questions for the drafter |

To **edit the plan** before approving (adjust items, charter, or persona), use:

```http
PATCH /api/seed-plans/:id
Content-Type: application/json

{
  "items": [
    { "id": "item-id", "title": "Billing overview", "coverage": ["what billing is", "the plans"] }
  ]
}
```

Editing is only allowed while the plan is `proposed`. To approve the plan and start drafting:

```http
POST /api/seed-plans/:id/approve
```

This flips the plan to `approved` and enqueues one `draft_seed_document` AI job per non-dismissed item, returning `{ "plan": SeedPlan, "jobIds": string[] }`. To reject a plan without drafting:

```http
POST /api/seed-plans/:id/dismiss
```

Dismissal is sticky: the sparse-flow bootstrap will not re-propose a plan for the same flow until the flow's sources change.

Seeding is *not* configured via the `KNOWLEDGE_SOURCES` environment variable – that variable is used to define the git or local paths that the flow indexes for ongoing operations. Seeds are not passed as inline content; instead, the job carries references to the flow's configured sources (`SourceDescriptor[]`), which the watcher resolves to traversable workspaces and lets the agent explore directly.

### MCP Tools

The same seeding operations are available through the Markdown Magpie MCP server as the `kb_seed` and `kb_outline` tools. `kb_outline` (flow + optional `notes`) enqueues an `outline_flow_seed` job, waits for it to complete, and returns the **persisted plan** — `planId`, items, rationale, and proposed charter/persona with `charterProposed`/`personaProposed` flags. No items are drafted by `kb_outline`; it is a planning-only step. `kb_seed` accepts a **plan id** (from `kb_outline`'s `planId` or the console) and calls `POST /api/seed-plans/:id/approve`, returning the enqueued draft job ids. Both require the `manage:jobs` scope. The tool descriptions advise using `kb_outline` first to auto-generate a plan, then reviewing/editing it (in the console or via the API), and finally calling `kb_seed` with the plan id to approve it — keeping a human (or calling agent) in the loop between the two steps.

## How Seeding Integrates with Indexing and Embedding

When a seed plan is approved, the API enqueues `draft_seed_document` AI jobs for each approved item. These jobs draft Markdown content grounded in the flow's source repositories, which the executing agent explores directly. All seed drafting jobs are processed asynchronously by the **watcher** process, which handles all generative AI work in the system. The API never performs chat or content generation inline — it enqueues the job and returns immediately. This means seed drafting will only complete if a watcher with the appropriate provider capability (matching `AI_PROVIDER`) is running and connected to the API. If no watcher is available, the jobs remain queued indefinitely. This pattern is consistent across all generative operations in Markdown Magpie.

On completion, the API creates a **clusterless proposal** – a proposal that does not originate from a gap cluster. The proposal carries the flow's id first-class so the reconcile gate and per-flow outbox treat it as same-flow. It then passes through the shared reconcile gate:

1. The proposal is stored with status `draft`.
2. The reconcile gate (`reconcileSeedProposal`) checks whether an open proposal in the same flow already targets the same path. If so, the seed document is folded into that open proposal (a `fold_markdown_proposal` AI job merges the rival content into the survivor). Otherwise, or if the only overlap is an approved/non-touchable PR, the seed proposal self-publishes as its own PR.
3. A human reviews it in the console (or via API) and changes its status to `ready`.
4. The human triggers publication (`POST /api/proposals/:id/publish`), which pushes the document to a new branch (and optionally opens a pull request for git-backed destinations).
5. After the pull request is merged (or the branch is accepted), the new content becomes part of the indexed knowledge base.

**Seeds are not ingested as high-priority knowledge automatically.** They must pass through the same human gate as any other proposal. The Markdown Magpie source documentation states: “seeding still ends at a reviewable pull request — the same human gate as everything else.” Only after merge and a subsequent re-index will the seed content be searchable and citable in answers.

**What happens when sources do not support a coverage point.** During drafting, the agent explores the flow's source repositories to substantiate every coverage point. If a point genuinely cannot be supported by the source material, it is **omitted from the document body** (never written as a gap, placeholder, or note). Instead, it is reported in the job output's optional `uncoveredPoints` array. The API folds those points into the proposal's `rationale` so the reviewer sees what could not be covered, while the document itself remains clean and factual. This behaviour is shared with the gap-drafting pipeline and enforced by the same factual-register contract.

Embedding of the new content happens when you trigger a re‑index of the flow (e.g. `POST /api/knowledge/repositories/index`). The indexed sections are then chunked and embedded inline by the API (the API holds the embedding provider). There is no special priority or separate embedding pass for seeds.

## Step-by-Step Process

1. **Identify what to seed** – decide which documents the flow should contain.
2. **Generate an outline plan** – call `POST /api/flows/:flowId/outline` to get a machine‑proposed list of items. The persisted plan is available by polling the returned job id via `GET /api/jobs/:id/wait`, then reading the plan via `GET /api/flows/:flowId/seed-plans`.
3. **Review and edit the plan** – use `PATCH /api/seed-plans/:id` or the console to adjust items, charter, persona, or per-item status. Ensure the proposed documents match your intent.
4. **Approve the plan** – call `POST /api/seed-plans/:id/approve` to enqueue one `draft_seed_document` job per non-dismissed item.
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

- **Use clear titles and headings** – the title becomes part of the file name (e.g. `<title-slug>.md`) and helps retrieval rank the document higher for relevant queries. Use `targetPath` when you need explicit control over the file path.
- **Supply motivating questions** – the optional `questions` field on each seed item provides context to the drafter, helping it focus the document on the scenarios that matter most.
- **Structure documents for retrieval** – break content into well‑named sections with `##` headings. The Markdown parser splits documents by heading, producing sections that can be independently retrieved. Good structure means the right section answers the right question.
- **Balance breadth and depth** – a seed that covers a topic broadly but lacks detail may produce shallow answers. Conversely, a very deep document on a narrow topic may be missed for broader queries. Strike a balance that matches expected usage.

## Common Pitfalls

- **Over‑seeding causing noise** – seeding too many documents in one go floods the review queue with proposals. Start with a small set of high‑value topics and iterate.
- **Forgetting to re‑index after adding seeds** – even after merge, the new content does not appear in answers until the flow is re‑indexed. Always trigger an index after a seed publication series.
- **Mixing seeds with live content** – seeds are not automatically part of the knowledge base until published and merged. During review they are invisible to the answer system, so do not rely on them until after indexing.

## Summary

Seeding in Markdown Magpie is a deliberate, human‑reviewed way to bootstrap knowledge bases. It uses a plan-centric API workflow that proposes a seed plan, lets you review and edit it, then approves it to draft documents, creates proposals, and funnels them through the same review and publication pipeline as gap‑driven content. When used thoughtfully – with clear items, balanced scope, and proper post‑merge indexing – seeding establishes a solid foundation for accurate, cited answers from day one.