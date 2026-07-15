---
title: Configuring a Persona for a Knowledge Flow
status: draft
---

# Configuring a Persona for a Knowledge Flow

A **persona** is an optional, admin-authored text snippet that tells the AI how to *sound* when answering questions about a particular knowledge flow. It shapes the tone, framing, and emphasis of answers — for example "Be kind and empathetic" or "Formal, high-level" — without adding facts the knowledge base does not contain.

This page explains what a persona is, how to set one, how it interacts with other flow configuration, and where it takes effect.

## What a Persona Does and Does Not Do

When a flow answers a question, its persona is appended to the base answer prompt so the model knows the audience and answering style. The assembled prompt looks like this:

```
<base answer instructions>

Persona (how to look and respond):
<your persona text>

<grounding guard: tone only, never adds facts>
```

The **grounding guard** is a fixed sentence appended automatically after every persona:

> The persona above changes tone, framing, and emphasis only. It never overrides the grounding rules: do not add facts, credentials, or claims the context does not contain, even when the persona would benefit from them.

This means a persona can make answers sound friendlier, more technical, or more concise — but it must **never** fabricate certifications, figures, or capabilities the knowledge base does not support. If the persona would benefit from a fact the docs lack, that fact is a knowledge gap, not a licence to invent.

(See `packages/prompts/src/catalog.ts` — the `withPersona()` function and `PERSONA_GROUNDING_GUARD` constant.)

## How to Set a Persona

Personas are configured per flow in the `KNOWLEDGE_FLOWS` environment variable. Each flow object in the JSON array may carry a `persona` field. As a fallback, the `description` field is also accepted.

### Example: setting a formal security-flow persona

```env
KNOWLEDGE_FLOWS=[
  {
    "id": "sec-flow",
    "sourceIds": ["agent"],
    "destinationId": "sec",
    "persona": "Formal, high-level."
  }
]
```

### Example: setting a support-flow persona via the fallback `description` field

```env
KNOWLEDGE_FLOWS=[
  {
    "id": "dev-flow",
    "sourceIds": ["agent"],
    "destinationId": "dev",
    "description": "Factual, with code."
  }
]
```

Both fields work; `persona` takes precedence when both are present.

(See `apps/api/src/stores/knowledge-repositories.ts` — the `normalizeFlowEntry()` function reads `persona` first, falling back to `description`; also see the test file `apps/api/src/stores/knowledge-repositories.test.ts` which demonstrates both.)

### Example: combining persona with routing summary

A **routing summary** (`routingSummary` or the fallback `summary`) is a separate field that describes the flow's *topical scope* — what subjects it covers. It is used for embedding-similarity routing and is distinct from the persona, which only changes the answer's voice.

```env
KNOWLEDGE_FLOWS=[
  {
    "id": "sec-flow",
    "sourceIds": ["agent"],
    "destinationId": "sec",
    "persona": "Formal, high-level.",
    "routingSummary": "Security, compliance, and access control."
  }
]
```

The routing summary sharpens which flow a question is routed to; the persona shapes how the answer reads. They are independent and can be set together or separately.

## Where the Persona Takes Effect

### In answers

The persona is attached to the `answer_question` job input when the API enqueues it. In the watcher (`apps/watcher/src/runners/generative.ts`), the `withPersona()` function merges it into the system prompt before the model assesses context and generates an answer. The persona is included in the job's `flows` array as an optional `persona` property.

### In flow routing (embedding router)

The persona is part of the flow text used by the embedding-similarity router (`POST /api/route`). The router concatenates the flow's **name**, **routing summary**, and **persona** and embeds them together to find the best-matching flow for a question. Because the persona describes *audience and voice*, a question about billing asked in a "kind, empathetic tone" flow will route differently from the same question in a "technical, precise" flow.

(See `apps/api/src/features/route/service.ts` — the `flowText()` function combines `flow.name`, `routingSummary`, and `flow.persona`.)

### In the console UI

When a flow has a persona, the Flows panel in the web console shows a person icon button. Clicking it opens a modal displaying the raw persona text. The Prompts panel shows every configured flow's persona alongside a "How the prompt is assembled" illustration, so you can inspect exactly what is sent to the model.

(See `apps/web/src/components/KnowledgePanel.tsx` — the `PersonaModal` component; and `apps/web/src/components/PromptsPanel.tsx` — the `FlowPersonasCard` component.)

### In flow seeding

The persona is passed to the `outline_flow_seed` job so that proposed seed documents match the flow's voice from the start. When a human uses the Seed / add an area page to propose new coverage, the outline generator receives the persona alongside the topic and existing documents.

(See `apps/api/src/features/seed/service.ts` and `apps/api/src/features/seed/service.test.ts`.)

## What the Persona Does NOT Affect

The persona is **internal** to the flow configuration. The `GET /api/knowledge/flows` endpoint exposes only `id` and `name` to callers; the persona stays on the server side and is never leaked to API consumers. This means MCP clients see the flow id and name to let users pick a flow, but the persona text itself is never exposed through that endpoint.

(See `apps/api/src/features/knowledge/service.ts` — `listFlows()` maps only `id` and `name`; and `apps/api/src/features/knowledge/routes.test.ts` which asserts personas stay internal.)

## Checking That Your Persona Is Active

1. **Start the API and watcher** with your `KNOWLEDGE_FLOWS` configured to include a persona.
2. **Open the web console** (default `http://localhost:3000`).
3. **Navigate to the Prompts panel** (under the Knowledge section). Find the "Flow personas" card — it lists every configured flow and its persona.
4. **Or navigate to the Flows panel**: if a flow has a persona, a person icon button appears next to the flow name; click it to see the raw persona text.

## Example: Putting It All Together

```env
KNOWLEDGE_SOURCES=[{"id":"agent","kind":"agent"}]
KNOWLEDGE_DESTINATIONS=[{"id":"docs","name":"Product Docs","path":"knowledge-bases/product"}]
KNOWLEDGE_FLOWS=[
  {
    "id": "product-docs",
    "name": "Product Documentation",
    "sourceIds": ["agent"],
    "destinationId": "docs",
    "persona": "Clear, concise, and developer-focused. Prefer examples over abstract explanations.",
    "routingSummary": "Product setup, configuration, API reference, and troubleshooting for developers."
  }
]
```

This configures one flow (`product-docs`) that:
- Reads from an agent knowledge source
- Writes to a local Markdown directory
- Answers questions in a clear, developer-focused tone
- Is routed to when questions relate to product setup, configuration, API reference, or troubleshooting

## Further Reading

- `KNOWLEDGE_FLOWS` configuration reference — see `apps/api/src/stores/knowledge-repositories.ts` (the `ConfiguredKnowledgeFlow` interface and `normalizeFlowEntry()`)
- Prompt assembly — see `packages/prompts/src/catalog.ts` (`withPersona()` and `PERSONA_GROUNDING_GUARD`)
- Embedding flow routing — see `apps/api/src/features/route/service.ts`
- Console UI — see `apps/web/src/components/KnowledgePanel.tsx`, `apps/web/src/components/PromptsPanel.tsx`
- Tests — see `apps/api/src/stores/knowledge-repositories.test.ts`