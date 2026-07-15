---
title: How Markdown Magpie Tracks AI Costs
status: draft
---

## Overview

Markdown Magpie tracks the monetary cost of its AI usage by combining three pieces of information: token usage reported by AI providers when they complete work, the execution identity (which provider and model performed the work), and an operator-supplied price table that maps each (provider, model) pair to per-million-token rates. Costs are computed at read time from stored token counts — they are never persisted, so correcting a price entry retroactively re-values all historical cost data.

## Token Usage Capture

When a watcher runs an AI job (such as drafting a proposal or answering a question), it captures the token usage reported by the underlying AI provider. API-backed providers (OpenAI-compatible and Azure OpenAI) surface the OpenAI-style usage block containing input and output token counts. The watcher sums the readings across all provider calls for a single job and sends the aggregated totals as part of the job's completion envelope.

CLI providers (Claude Code and Codex) emit raw text and do not report token usage. Their completions carry no usage data at all, which the system treats as unmetered — not free, but with invisible spend. This distinction is maintained throughout every cost report.

Token usage is optional end to end: a provider that reports nothing produces a completion with no usage data. The system never fabricates or estimates token counts.

## Execution Identity

Alongside the usage totals, the watcher stamps every job completion with the execution identity: the provider name and the model name (or Azure deployment name) that were configured for that run. This identity is what lets the system later convert raw token counts into monetary cost, because cost is a function of the specific model that ran. Without the model name, there would be no way to know which price to apply.

A CLI provider that has no explicit model configured reports only its provider — reporting nothing about the model beats guessing which default the CLI resolved. Jobs completed by an older watcher that predates identity reporting also carry no model.

## Operator Price Table (AI_PRICING)

The operator supplies token pricing through the `AI_PRICING` environment variable, which is a JSON array of per-model price entries. Each entry specifies the provider, the model name, the cost per million input tokens, and the cost per million output tokens.

```json
[
  {
    "provider": "openai-compatible",
    "model": "gpt-4o-mini",
    "inputPerMTok": 0.15,
    "outputPerMTok": 0.6
  }
]
```

The prices are in the deployment's billing currency — the system is currency-agnostic and makes no assumption about which currency is used. The table is validated at API startup: a malformed entry (invalid JSON, unknown provider, missing fields, duplicate provider/model pairs) causes the API to fail boot rather than silently misprice. An unset or blank `AI_PRICING` variable means no pricing is configured, and cost reporting stays off entirely.

Because costs are computed at read time and never persisted, the operator can freely update the price table. A corrected entry immediately re-values all historical usage data without any data migration.

## Cost Calculation at Read Time

The cost for an AI job is calculated as:

```
estimatedCost = (inputTokens × inputPerMTok + outputTokens × outputPerMTok) / 1_000_000
```

The division by one million converts per-million-token rates to actual costs. The calculation only produces a value when both usage data exists and a matching price entry is found. A model that is null (no identity reported) can never match a price entry, so its cost stays unknown rather than being silently attributed to some other model's rate.

## Three Cost States

The system maintains three distinct cost states throughout all reporting, ensuring that no row is ever misleadingly shown as costing zero:

- **Priced**: usage was reported and a matching entry exists in the `AI_PRICING` table, so a concrete estimated cost is shown.
- **Unpriced**: usage was reported but no price entry matches this (provider, model) pair, so the cost is unknown and no numerical estimate is displayed.
- **Unmetered**: no usage was reported at all (the CLI provider case), so there are no tokens to price.

These three states survive aggregation into every cost summary, so a flow or schedule is never misreported as having cost nothing when some of its spend is unpriced or unmetered.

## API Endpoints

The following API endpoints expose cost data:

- **`GET /api/insights/ai-usage`** — Returns token usage per (job type, provider, model) triple over the window, priced into cost at read time. One stacked bar per triple, heaviest first. Cost appears in the header total and per-bar tooltip, never as a series colour or second y-axis.

- **`GET /api/insights/ai-cost/by-flow`** — Per-flow AI cost over the window, aggregating spend by the flowId on the job input. Jobs whose input carries no flowId (answers and cross-flow folds) are grouped as Unattributed. Ordered by cost, heaviest first.

- **`GET /api/insights/ai-cost/by-schedule`** — Approximate per-schedule AI cost, summing over the AI job types each scheduled task fans out to, filtered to the task's own flow. Keyed by `ScheduledTask.key` so the Schedules page can join it on. Attribution is approximate: only job types carrying a flowId on their input are counted, and second-order proposal folds are not attributed.

All insights endpoints default to a 30-day window and require the `read:knowledge` scope.

## Source Data

The source of cost data is the pg-boss `job` table — specifically, completed rows on the provider-fanned AI work queues. The watcher sums each run's provider-reported usage and stamps its execution identity, and the API persists both on the completion envelope (`{ result, executor, usage?, provider?, model? }`). No new table was introduced for cost tracking: the existing pg-boss retention window (30 days) covers the charting window.

## Console Visualisations

The web console's Insights page displays two cost-focused charts:

- **AI token usage & cost**: A stacked bar chart showing input and output tokens per (job type, provider, model) triple, with a header for the estimated total cost and per-bar tooltips showing cost state (priced, unpriced, or unmetered) and metered job counts.

- **AI cost by flow**: A per-flow breakdown of estimated spend, with the same three-state honesty. Flow display names are resolved from the operator's `KNOWLEDGE_FLOWS` configuration; jobs without a flowId appear as Unattributed.

The Schedules page also includes a per-schedule cost column showing each scheduled task's estimated spend over the last 30 days, with tooltip breakdowns of priced, unpriced, and unmetered jobs.

## Configuration Reference

- **`AI_PRICING`** — JSON array of token price entries, read at API startup via `loadConfig()` and parsed by `parseAiPricing()`. Validated strictly: a malformed table fails boot. Unset or blank leaves cost reporting off. Prices are not secrets and are echoed verbatim in the `GET /api/config` response so operators can verify what the API loaded.