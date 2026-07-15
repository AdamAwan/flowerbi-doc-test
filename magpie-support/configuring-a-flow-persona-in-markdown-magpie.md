---
title: Configuring a Flow Persona
status: draft
---

# Configuring a Flow Persona

A **persona** is an optional, admin-authored snippet attached to a knowledge flow that describes the flow's audience and answering style. It shapes *how* the model responds — tone, framing, and emphasis — without overriding the factual grounding rules every answer must obey.

This page explains what a persona is, how to set it, where it takes effect, and how it differs from a routing summary.

## What a persona does

When a configured flow answers a question, the persona is appended to the base answer-prompt instructions. The model then sees:

```
<standard answer instructions>

Persona (how to look and respond):
<your persona text>

The persona above changes tone, framing, and emphasis only. It never overrides the grounding rules: do not add facts, credentials, or claims the context does not contain, even when the persona would benefit from them.
```

For example, a support flow with the persona `"You are a friendly, patient support agent. Keep answers concise and practical."` will produce answers that are friendlier and more concise than the default. A developer-docs flow with the persona `"Factual, with code examples."` will favour technical precision and include where relevant.

### The grounding guard

The snippet above — the `PERSONA_GROUNDING_GUARD` — is always injected *after* the persona (see `packages/prompts/src/catalog.ts`). It ensures that a persuasive or marketing-flavoured persona never licenses the model to fabricate facts, certifications, figures, or capabilities that the retrieved knowledge-base context does not contain. **The persona changes tone only, never the truth.**

## How to configure a persona

Personas are set per flow in the `KNOWLEDGE_FLOWS` environment variable, which is a JSON array of flow objects (see `apps/api/src/stores/knowledge-repositories.ts`, function `normalizeFlowEntry`).

### Using the `persona` field

```jsonc
KNOWLEDGE_FLOWS=[
  {
    "id": "support",
    "name": "Customer Support",
    "sourceIds": ["product-docs", "agent"],
    "destinationId": "support-kb",
    "persona": "You are a friendly, patient support agent. Keep answers concise and practical."
  },
  {
    "id": "engineering",
    "name": "Engineering Docs",
    "sourceIds": ["codebase"],
    "destinationId": "eng-kb",
    "persona": "Factual, with code examples. Assume the reader is a developer."
  }
]
```

### Using `description` as a fallback

If the `persona` field is absent, the parser falls back to the `description` field:

```jsonc
KNOWLEDGE_FLOWS=[
  {
    "id": "dev-flow",
    "sourceIds": ["agent"],
    "destinationId": "dev",
    "description": "Factual, with code."
  }
]
```

This produces the same result as setting `"persona": "Factual, with code."` (confirmed by unit tests in `apps/api/src/stores/knowledge-repositories.test.ts`).

## Where the persona takes effect

### 1. Answer synthesis (the main effect)

When a watcher answers a question for a routed flow, it calls `withPersona(ANSWER_QUESTION.instructions, routedFlow?.persona)` in `apps/watcher/src/runners/generative.ts`. This appends the persona (and the grounding guard) to the answer instructions that guide the model's assess/answer loop. Every answer from that flow is therefore shaped by its persona.

### 2. Flow routing (embedding similarity)

The persona is part of the text used to embed a flow for the embedding-similarity router (`POST /api/route`). The routing text is the concatenation of:

```
<flow name>
<routing summary>
<persona>
```

(see `apps/api/src/features/route/service.ts`, function `flowText`). This means a persona that clearly describes the audience (e.g. "Support agent helping end users with billing and account questions") can also help the embedding router assign questions to the right flow.

### 3. Flow seeding outlines

When generating a seed outline (`POST /api/flows/:id/outline`), the persona is passed to the `outline_flow_seed` job. The outline prompt tells the model: `"persona" (optional): the flow's audience/voice.` so the proposed document list fits the right tone and scope (see `apps/api/src/features/seed/service.ts`).

### 4. Gap cluster reconciliation

During gap-cluster reconciliation (`apps/watcher/src/runners/generative.ts`), the reconciler attaches the flow's persona as scope grounding. This helps the model judge whether a gap cluster is off-topic for the knowledge base and should be dismissed rather than drafted. A cluster about cats in a product flow whose persona says "Technical product documentation" is clearly dismissible.

### 5. Web console display

If a flow has a persona, the web console's **Knowledge** panel shows a person icon (👤) next to the flow's metadata. Clicking it opens a modal that displays the persona snippet verbatim, labelled as "Appended to the base answer prompt when this flow answers a question." (see `apps/web/src/components/KnowledgePanel.tsx`, component `PersonaModal`).

### 6. API exposure

The persona is sent to the watcher as part of the `answer_question` job input. It is included in the `flows` array that the API builds via `buildAnswerQuestionInput` in `apps/api/src/platform/answer-question.ts`:

```typescript
const flows = ctx.knowledgeConfig.flows.map((flow) => ({
  id: flow.id,
  name: flow.name,
  ...(flow.persona ? { persona: flow.persona } : {})
}));
```

Note: the persona is **not** exposed through the public `GET /api/knowledge/flows` endpoint — that endpoint intentionally leaks only `id` and `name` (see `apps/api/src/features/knowledge/service.ts`).

## Persona vs. routing summary

The persona is **distinct** from the `routingSummary` (or its alias `summary`). They serve different purposes:

| Aspect | Persona | Routing summary |
|---|---|---|
| **What it does** | Shapes the answer's *voice* — tone, framing, emphasis | Describes the flow's *topical scope* — what subjects it covers |
| **Where it takes effect** | Appended to the answer prompt; influences how the model writes | Used only for embedding-similarity routing; influences which flow a question is sent to |
| **Field** | `persona` (fallback: `description`) | `routingSummary` (fallback: `summary`) |
| **Required** | No | No |

Setting one does not affect the other. You can have a persona without a routing summary, a routing summary without a persona, both, or neither. The unit test in `apps/api/src/stores/knowledge-repositories.test.ts` (`"parses a flow routing summary from the routingSummary or summary field, separately from persona"`) confirms this independence.

## Best practices

- **Keep personas short.** A sentence or two is enough to set tone. "Be kind and helpful" is more effective than a paragraph.
- **Describe the audience, not the content.** The persona tells the model *who it is talking to* and *how to sound*, not *what to say*. The routing summary covers what.
- **Avoid factual claims in the persona.** The grounding guard protects against invented facts, but a persona that asserts "You are an expert in SOC 2 compliance" may still bias the model toward answering confidently about compliance even when the knowledge base lacks the specifics. Prefer "You are a helpful guide answering from our security documentation."
- **Use personas to differentiate flows.** If you have a support flow and an engineering flow, give them contrasting personas so answers to the same question feel appropriate to each audience.

## Troubleshooting

### My persona doesn't seem to affect answers

Check:
1. The flow is correctly referenced in `KNOWLEDGE_FLOWS` and the `persona` field is a non-empty string.
2. Questions are reaching that flow (check the answer trace in the console or the question log's `flowId`).
3. The persona is not overridden by a `requestedFlowId` that routes to a different flow.
4. The persona is not empty or whitespace-only — the `withPersona` function returns the base instructions unchanged when the persona is blank (`packages/prompts/src/catalog.ts`, line `const trimmed = persona?.trim(); return trimmed ? ... : baseInstructions`).

### My persona causes factual errors

The grounding guard prevents the persona from overriding factual accuracy, but if the persona strongly biases the model (e.g. "You are a salesperson who always says yes"), the model may still attempt to comply within the guard's constraints. If you observe factual errors, make the persona more neutral and check that the knowledge base actually covers the topic.

## Reference

- Persona type definition: `apps/api/src/stores/knowledge-repositories.ts`, `ConfiguredKnowledgeFlow.persona` (optional `string`)
- Persona parsing (fallback to `description`): `apps/api/src/stores/knowledge-repositories.ts`, function `normalizeFlowEntry`
- `withPersona` function and `PERSONA_GROUNDING_GUARD`: `packages/prompts/src/catalog.ts`
- Answer-time persona injection: `apps/watcher/src/runners/generative.ts`, `answer()` function
- Routing text composition (persona as routing signal): `apps/api/src/features/route/service.ts`, `flowText()`
- Seed outline persona passing: `apps/api/src/features/seed/service.ts`, `outlineFlowSeed()`
- Gap reconciliation scope grounding with persona: `apps/watcher/src/runners/generative.ts`, `clusterSummaryLine()`
- Web console persona modal: `apps/web/src/components/KnowledgePanel.tsx`, `PersonaModal` component
- API job-input persona building: `apps/api/src/platform/answer-question.ts`, `buildAnswerQuestionInput()`
- Unit test confirming persona/description parsing: `apps/api/src/stores/knowledge-repositories.test.ts`
- Unit test confirming `withPersona` appends the grounding guard: `packages/prompts/src/catalog.test.ts`