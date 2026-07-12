---
title: Source Maps and Flow Sharing in Markdown Magpie
status: draft
---

## Overview

A source map is a per-source-repository store of topic-indexed navigation hints maintained by the agents themselves. Each entry pairs a topic with one or more file paths and a one-line description, and is unique on `(sourceId, topic)` (see `docs/ai-jobs.md` → Source map (agent navigation hints)). Source maps are internal metadata for source-grounded prompts — they never enter answer retrieval or user-facing output.

## Per-Source Design

Source map entries are scoped to a source repository, not to a flow. The underlying store — whether the in-memory implementation (`InMemorySourceMapStore` in `apps/api/src/stores/source-map-store.ts`) or the Postgres implementation (`PostgresSourceMapStore` in `apps/api/src/stores/postgres-source-map-store.ts`) — keys entries on the pair `(sourceId, topic)`. The store interface exposes `listBySource(sourceId, limit)` to retrieve entries for one source, and the write path accepts updates keyed by `sourceId` and `topic`.

## Entry Properties

Each source map entry carries the following fields (see `SourceMapEntry` in `packages/core/src/index.ts`):

- **`id`** — a stable UUID, preserved across upserts of the same `(sourceId, topic)`.
- **`sourceId`** — the source repository the hint belongs to.
- **`topic`** — a short label for what the path(s) cover (max 120 characters).
- **`paths`** — one or more file or directory paths relevant to the topic (max 8 entries, each ≤260 characters).
- **`description`** — a one-line summary (max 240 characters).
- **`observedSha`** — the Git HEAD SHA of the source at write time (stamped by the watcher, never trusted from the model); `null` for non-git sources.
- **`consensusCount`** — how many agents have independently contributed the same topic→paths mapping (capped at 5), indicating credibility.
- **`createdAt`** / **`updatedAt`** — ISO-8601 timestamps.

### Consensus Count

The consensus count is a credibility signal distinct from the currency signal (`observedSha`). On every upsert, the new hint's paths are compared to the existing entry's paths via Jaccard similarity (see `apps/api/src/stores/source-map-consensus.ts`, function `nextConsensusCount`):

- **Agreement** — when the Jaccard similarity strictly exceeds 0.5, the new hint is treated as an independent confirmation, and the count increments (capped at 5).
- **Contradiction** — when the similarity is at or below 0.5, the new hint disagrees with the stored mapping, and the count resets to 1.
- **First write** — a new `(sourceId, topic)` pair starts at 1.

Higher consensus counts mean more agents agree on the same location, making the hint more trustworthy. Surfacing or filtering hints by consensus count is a follow-up (issue #219).

### Deterministic Ordering

When two entries for the same source have identical `updatedAt` timestamps (possible with millisecond resolution), both stores break the tie deterministically: the in-memory store uses a monotonic write-sequence counter (`apps/api/src/stores/source-map-store.ts`), and the Postgres store uses an auto-incrementing `seq` column (`apps/api/src/stores/postgres-source-map-store.ts`). This ensures that the "most-recently-updated first" ordering is consistent across backends even for concurrent writes in the same clock tick.

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

Source map fetch is best-effort: if the API is unreachable or returns partial results, the job continues without hints rather than failing (see `fetchSourceMapEntries` in `apps/watcher/src/source-workspace.ts`).

## Write Path

The six source-grounded job types — `draft_seed_document`, `draft_markdown_proposal`, `outline_flow_seed`, `verify_document`, `correct_document`, and `improve_document` — accept an optional `mapUpdates` field in their output (see `apps/api/src/features/source-map/service.ts`, constant `SOURCE_GROUNDED_JOB_TYPES`). When a job completes, `applySourceMapUpdatesFromCompletedJob` processes each update:

- Updates are validated against the source IDs that the job was actually grounded in (taken from the job's input `sources`).
- Each update is validated for size constraints: topic ≤120 chars, paths (1–8 entries, each ≤260 chars), description ≤240 chars. Malformed updates are dropped with a structured log warning.
- Each valid update is upserted into the store by `(sourceId, topic)`. On upsert, the entry's `consensusCount` is recomputed atomically against the existing paths via Jaccard similarity.
- The per-source cap of 200 entries is enforced with oldest-updated eviction.
- Updates are capped at 20 per job.

### Atomicity

In the Postgres store, the consensus count is computed inside a transaction with `SELECT ... FOR UPDATE` on the matching row (see `apps/api/src/stores/postgres-source-map-store.ts`). This serialises concurrent upserts for the same `(sourceId, topic)`, preventing two simultaneously completing jobs from losing each other's consensus increment (issue #219).

### `observedSha` Stamping

The watcher overwrites any `observedSha` the model may have supplied with the checkout HEAD SHA it actually observed during workspace preparation (see `stampSourceMapUpdates` in `apps/watcher/src/source-workspace.ts`). For non-git sources, or when the workspace list is empty (ungrounded fallback path), the `observedSha` is stripped out entirely. The SHA is an infrastructure fact, never trusted from the model.

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
- **Field-level constraints** (enforced by `rejectReason` in the service):
  - `topic`: max 120 characters.
  - `paths`: 1 to 8 entries, each max 260 characters.
  - `description`: max 240 characters.
- **Consensus cap**: Maximum consensus count of 5 (`MAX_CONSENSUS_COUNT` in `apps/api/src/stores/source-map-consensus.ts`).
- **Storage backend**: Defaults to Postgres; can be overridden with `SOURCE_MAP_STORE` environment variable (see `docs/ai-jobs.md` → Storage).