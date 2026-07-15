---
title: Markdown Magpie — Current Adoption and Usage Context
status: draft
---

# Markdown Magpie — Current Adoption and Usage Context

## Origin and Development Model

Markdown Magpie was built by Adam Awan to solve a personal organisational pain point: earlier attempts at knowledge-sharing tooling at his company had produced what he described as "basic knowledge dumps". Magpie was conceived as a fundamentally different approach — grounded in real source material, every answer cited, knowledge curated through review, and the system self-improving and self-pruning.

The project is developed primarily by AI coding agents — Claude (Claude Code) and Codex — with human direction and review. The planning notes and specs within the repository are agent-driven artefacts that produced the code. This development model is stated explicitly in the project's README.

## Current Development Stage

The codebase itself indicates that Markdown Magpie remains in pre-production / pre-pilot development. Several signals across the repository support this:

- The **MVP plan** defines three milestones — Cited Answers, Gap Loop, and Pull Requests — followed by a "Later" section covering stale document detection, duplicate detection, contradiction detection, owner reminders, and usage scoring. This suggests the feature set is still being built out.
- The **security review** includes a checklist of items described as "must be completed for a production/internal deployment", including TLS termination, Postgres password replacement, CORS restriction, Auth0 configuration, branch protection, compliant AI provider selection, and secret management — indicating production hardening is still to be done.
- Multiple design and planning documents reference the fact that "no production data yet" exists, and that data loss during migrations or schema changes is acceptable as a result.
- The **pitch deck** was produced as "an HTML/JS keyboard-navigated slide deck to present the app to colleagues and win buy-in to pilot it". Its final slide is a call to action: "Pick one source. Let it run." The audience was described as "decision-makers who are all technically strong and enthusiastic about AI", and the goal was explicitly to "get buy-in by showing how impressive and principled the design is".

## Demo and Testing Corpus

The example knowledge base used throughout development and in the pitch deck is **FlowerBI**, a public GitHub project. The pitch deck screenshots were "real captures from a live FlowerBI knowledge base". Configuration examples in the documentation use the FlowerBI repository as a reference source and a separate FlowerBI documentation repository as the destination.

## Intended Use Cases (from the pitch deck)

Although no customer deployments exist, the pitch deck identifies several intended applications for Markdown Magpie, described as different "source material" inputs producing targeted knowledge base outputs:

- **Product code** → internal product questions answered with citations into the code
- **Product code + Azure documentation + company policies** → security questionnaires (grounded, consistent, defensible answers)
- **Product code + customer knowledge base** → front-line support with every answer cited to the product
- **Employee handbook + HR policies** → employee onboarding, self-service on policies, benefits, and processes
- **IT runbooks + known issues** → IT self-service for known fixes
- **Product documentation + pricing + competitor notes** → sales and pre-sales (consistent, cited answers to RFPs and prospect questions)
- **Existing product knowledge base** → taming a large, messy knowledge base into a refined, de-duplicated, contradiction-free distillation

## Sales Objection Handling

The existing sales documentation provides responses to common objections. Notably, one objection is that "Magpie seems too complex to integrate with our existing workflow". The recommended response highlights Magpie's flexibility, its queue-only architecture (the API enqueues AI work and a separate watcher process completes it), and its provider-neutral design.

Another documented objection is that the prospect "already has an internal knowledge base". The recommended positioning emphasises that Magpie is a complement, not a replacement — it adds gap detection, proposal generation, and self-maintenance on top of existing content.

## Target Customer Profile

The documentation positions Markdown Magpie as a "vendor-neutral knowledge maintenance system designed for teams that maintain Git-backed Markdown documentation". Best-fit buyers are described as teams that:

- Already maintain Git-backed Markdown documentation
- Want AI-assisted answers with citations
- Need automatic knowledge gap detection
- Want pull requests generated to fill those gaps
- Require vendor-neutral infrastructure (no model lock-in)
- Value MCP-native integration (knowledge appears inside tools people already use)

## Pricing and Plans

Pricing documentation exists but has been consolidated into a separate page covering self-hosted free usage, managed cloud options, and pricing details. The project is released under the MIT License.

## No Published Customer Success Stories or Testimonials

At the time of writing, the Markdown Magpie codebase and its published documentation contain no customer success stories, testimonials, or named customer references. The project has not yet reached a stage where external customer outcomes have been captured and published. When such stories become available, they would naturally document how specific teams have deployed Magpie, which knowledge bases they manage with it, and what measurable improvements in knowledge maintenance they have achieved.