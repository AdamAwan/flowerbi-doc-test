---
title: Source Maps: Sharing Across Flows
status: draft
---

# Source Maps: Sharing Across Flows

## Overview

Source maps are per-source navigation hints, not per-flow. This means that when two or more flows reference the same source repository (by the same source ID), the source map entries accumulated by agents working in one flow are automatically visible to agents working in another flow that uses that same source. There is no flow-level isolation or scoping in the source map data model.

## Data Model

Source map entries are stored in the `source_map_entries` Postgres table, keyed on a unique constraint over `(source_id, topic)` (see `packages/db/migrations/0047_source_map_entries.sql`). The table has no `flow_id` column — the only scoping dimension is the source identity. The `SourceMapEntry` TypeScript interface in `packages/core/src/index.ts` confirms this: it carries `sourceId` but no `flowId` field.

The same independence from flows is visible in the `SourceMapStore` interface (`apps/api/src/stores/source-map-store.ts`), whose `listBySource(sourceId, limit)` method queries solely by source ID. The `upsert(update)` method likewise keys on `(sourceId, topic)`, ignoring any notion of flow.

## Read Path: How Hints Are Fetched

When a source-grounded job (e.g. `draft_markdown_proposal`, `draft_seed_document`, `verify_document`, `correct_document`, `improve_document`) runs, the watcher prepares source workspaces by resolving the job's `SourceDescriptor[]` references. It then calls `fetchSourceMapEntries` (`apps/watcher/src/source-workspace.ts`), which collects the `sourceId` from every resolved workspace and passes them all to `GET /api/source-map?sourceIds=...`. The API endpoint (`apps/api/src/features/source-map/routes.ts`) simply queries the store for each requested source ID and returns the combined list. No flow filtering is applied.

The entries are rendered into the agent prompt (see `apps/watcher/src/job-prompts.ts`, function `buildSourceGroundedPrompt`) as a block of "unverified" hints, each prefixed with its source ID (e.g. `[sourceId] topic: paths — description`). The agent sees every hint for every source the job touches, regardless of which flow the entry was originally contributed from.

Example: if Flow A and Flow B both configure a source with id `"flowerbi"`, an agent working on a proposal for Flow A will see source map hints that were contributed by an agent that previously worked on a proposal for Flow B — provided those hints reference the same `sourceId`.

## Write Path: How Hints Are Contributed

On job completion, the watcher stamps any `mapUpdates` the agent produced with the observed checkout HEAD SHA (overwriting any model-supplied value — see `stampSourceMapUpdates` in `apps/watcher/src/source-workspace.ts`) and sends them back as part of the job output. The API's completion dispatcher calls `applySourceMapUpdatesFromCompletedJob` (`apps/api/src/features/source-map/service.ts`), which validates each update against a set of allowed source IDs derived from the job's input `sources` array.

The validation checks that the update's `sourceId` is among the sources the job was actually grounded in. It does **not** check which flow the job belongs to — the guard is purely about whether the agent had access to that source. If the same source ID is configured across multiple flows, agents from any of those flows can update its map entries.

Updates are upserted by `(sourceId, topic)`, so agents from different flows that discover navigation facts about the same source+topic will overwrite each other's entries (latest write wins). The per-source cap of 200 entries (with oldest-updated eviction) applies across all flows collectively, not per flow.

## Implications

- **Cross-flow discovery.** Navigation hints learned in one flow benefit all other flows that share the same source. An agent working in a "product" flow that discovers the test suite lives in `tests/integration/` will publish a hint that a subsequent agent in the "engineering" flow can read — provided both flows reference the same source repo by the same ID.
- **No accidental isolation.** You cannot scope source map entries to a single flow by design. If two flows use different source IDs (even if they point to the same physical repository), their source maps are independent because the key is the source ID string.
- **Eviction is collective.** The 200-entry cap (`MAX_ENTRIES_PER_SOURCE` in `apps/api/src/features/source-map/service.ts`) is per source, not per flow. Agents from all flows that share a source compete for those 200 slots. A burst of map updates from one flow can evict entries contributed by another flow.
- **`observedSha` is source-level, not flow-level.** The watcher records the checkout HEAD SHA at the time of the job, so entries record the state of the source repo, not the state of any particular flow's derived knowledge base.