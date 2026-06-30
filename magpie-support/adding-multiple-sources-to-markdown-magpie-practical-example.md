---
title: Adding Multiple Sources to Markdown Magpie – Practical Examples
status: draft
---

# Adding Multiple Sources to Markdown Magpie – Practical Examples

This page provides ready-to-use examples of multi-source configurations. For full setup instructions, see [Setting Up Multiple Knowledge Sources](./setting-up-multiple-knowledge-sources-in-markdown-magpie.md). All examples assume you have set `MAGPIE_CHECKOUT_ROOT` if using git sources.

## Example 1: Local + Git + Internet

```env
MAGPIE_CHECKOUT_ROOT=.magpie/checkouts

KNOWLEDGE_SOURCES=[
  {"id":"local-docs","name":"Local Product Docs","kind":"local","path":"knowledge-bases/product"},
  {"id":"flowerbi","name":"FlowerBI Source","kind":"git","url":"https://github.com/danielearwicker/flowerbi.git","subpath":"src"},
  {"id":"api-docs","name":"API Documentation","kind":"internet","url":"https://example.com/api-docs"}
]

KNOWLEDGE_DESTINATIONS=[
  {"id":"kb-output","name":"Curated KB","kind":"local","path":"knowledge-bases/kb-output"}
]

KNOWLEDGE_FLOWS=[
  {"id":"main","name":"Main Knowledge Flow","sourceIds":["local-docs","flowerbi","api-docs"],"destinationId":"kb-output"}
]
```

## Example 2: Two Git Repositories + Agent Feedback

```env
KNOWLEDGE_SOURCES=[
  {"id":"user-guide","name":"User Guide","kind":"git","url":"https://github.com/org/user-guide.git","subpath":"docs"},
  {"id":"api-ref","name":"API Reference","kind":"git","url":"https://github.com/org/api-ref.git","subpath":"reference"},
  {"id":"agent-feedback","name":"Agent Feedback","kind":"agent"}
]

KNOWLEDGE_DESTINATIONS=[
  {"id":"combined-docs","name":"Combined Docs","kind":"local","path":"knowledge-bases/combined"}
]

KNOWLEDGE_FLOWS=[
  {"id":"full-kb","name":"Full Knowledge Base","sourceIds":["user-guide","api-ref","agent-feedback"],"destinationId":"combined-docs"}
]
```

## Example 3: Single Source, Multiple Destinations

```env
KNOWLEDGE_SOURCES=[
  {"id":"core","name":"Core Docs","kind":"local","path":"knowledge-bases/core"}
]

KNOWLEDGE_DESTINATIONS=[
  {"id":"public-kb","name":"Public KB","kind":"local","path":"knowledge-bases/public"},
  {"id":"internal-kb","name":"Internal KB","kind":"local","path":"knowledge-bases/internal"}
]

KNOWLEDGE_FLOWS=[
  {"id":"public","name":"Public Flow","sourceIds":["core"],"destinationId":"public-kb"},
  {"id":"internal","name":"Internal Flow","sourceIds":["core"],"destinationId":"internal-kb"}
]
```

For verification and indexing, refer to the main setup guide.