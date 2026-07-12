---
title: Source Maps and Flow Sharing in Markdown Magpie
status: draft
---

## Overview

A source map is a per-source-repository store of topic-indexed navigation hints maintained by the agents themselves. Each entry pairs a topic with one or more file paths and a one-line description, and is unique on `(sourceId, topic)` (see `docs/ai-jobs.md` → Source map (agent navigation hints)). Source maps are internal metadata for source-grounded prompts — they never enter answer retrieval or user-facing output.

## Per-Source Design

Source map entries are scoped to a source repository, not to a flow. The underlying store — whether the in-memory implementation (`InMemorySourceMapStore` in `apps/api/src/stores/source-map-store.ts`) or the Postgres implementation (`PostgresSourceMapStore` in `apps/api/src/stores/postgres-source-map-store.ts`) — keys entries on the pair `(sourceId, topic)`. The store interface exposes `listBySource(sourceId, limit)` to retrieve entries for one source, and the write path accepts updates keyed by `sourceId` and `topic`.

## Sharing Across Flows

A flow references one or more sources through its `sourceIds` field (see `ConfiguredKnowledgeFlow.sourceIds: string[]` in `apps/api/src/stores/knowledge-repositories.ts`). When two or more flows list the same source ID in their `sourceIds` array, they share that source's map entries.

For example, given a configuration where both a "Product" flow and an "Engineering" flow reference the same source `"core-repo"`:

```env
KNOWLEDGE_SOURCES=[{"id":"core-repo","name":"Core Repository","kind":"git","url":"https://github.com/org/core.git"}]

KNOWLEDGE_FLOWS=[
  {"id":"product","name":"Product Docs","sourceIds":["core-repo"],"destinationId":"product-kb"},
  {"id":"engineering","name":"Engineering Docs","sourceIds":["core-repo"],"destinationId":"eng-kb"}
]
```

Both flows would see the same source map entries for `"core-repo"` when agents navigate that source during source-grounded jobs.

## Read Path

At workspace preparation for any source-grounded job, the watcher fetches source map entries via `GET /api/source-map?sourceIds=…` (scope: `manage:jobs`). The endpoint iterates over each requested source ID and returns the ≤100 most-recently-updated entries per source (see `apps/api/src/features/source-map/routes.ts`). The watcher then renders these entries into the agent prompt as unverified hints after the repository list (see `apps/watcher/src/job-prompts.ts`, function `buildSourceGroundedPrompt`).

## Write Path

The five source-grounded job types — `draft_seed_document`, `draft_markdown_proposal`, `verify_document`, `correct_document`, and `improve_document` — accept an optional `mapUpdates` field in their output. When a job completes, `applySourceMapUpdatesFromCompletedJob` in `apps/api/src/features/source-map/service.ts` processes each update:

- Updates are validated against the source IDs that the job was actually grounded in (taken from the job's input `sources`).
- Each valid update is upserted into the store by `(sourceId, topic)`.
- The per-source cap of 200 entries is enforced with oldest-updated eviction.
- Updates are capped at 20 per job.

Because the source map store has no concept of a flow, an agent working on one flow can update source map entries that will be visible to agents working on another flow that shares the same source.

## Practical Implications

- **Shared navigation context**: When flows share a source, navigation hints accumulated by one flow's agents benefit the other flow's agents, reducing redundant exploration.
- **No per-flow isolation**: Source map entries are not partitioned by flow — if two flows reference the same source ID, they share the same hint set.
- **Staleness tracking**: Each entry records `observedSha` (the Git HEAD SHA of the source at write time, stamped by the watcher — see `apps/watcher/src/source-workspace.ts`, function `stampSourceMapUpdates`). Entries from non-git sources keep `observedSha` as `null`. Staleness invalidation via source-change-sync is a follow-up tracked in issue #215 (see `docs/ai-jobs.md`).

## Boundaries

The source map is strictly internal metadata for agent navigation. It never enters answer retrieval, the indexed knowledge base, or user-facing output (see `docs/architecture.md` → AI Job Execution, and `docs/ai-jobs.md` → Source map boundaries).

## Configuration and Capacity

- **Per-source cap**: 200 entries per source, enforced with oldest-updated eviction on writes (`MAX_ENTRIES_PER_SOURCE` in `apps/api/src/features/source-map/service.ts`).
- **Prompt injection cap**: 100 most-recently-updated entries per source are injected into the agent prompt (`PROMPT_ENTRY_LIMIT` in `apps/api/src/features/source-map/routes.ts`).
- **Per-job cap**: Maximum 20 map updates per job output (`MAX_UPDATES_PER_JOB` in the source-map service).
- **Storage backend**: Defaults to Postgres; can be overridden with `SOURCE_MAP_STORE` environment variable (see `docs/ai-jobs.md` → Storage).