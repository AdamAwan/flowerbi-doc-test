---
title: The Markdown Magpie Sales Process — From Lead to Close
status: draft
---

# The Markdown Magpie Sales Process

This document outlines the typical stages a sales team moves through when engaging a prospect for Markdown Magpie. Each stage describes the objective, key activities, and how Magpie's product characteristics inform the conversation.


## 1. Identifying Prospects (Lead Generation)

Markdown Magpie is a Git-backed Markdown knowledge maintenance system. It is best suited to teams that already maintain Markdown documentation in Git repositories and experience pain from stale or inconsistent knowledge. Prospects typically include engineering teams, product documentation teams, and knowledge-intensive functions whose source of truth lives in Markdown.

Leads may be identified through signals such as teams complaining about outdated documentation, relying on a wiki that has drifted from code, or fielding repeated questions that should be answered by existing docs. The product's audience is technically strong decision-makers who are enthusiastic about AI and can evaluate a self-hosted, open-core system.


## 2. Qualification

A prospect is well qualified when they meet several criteria:

- **Git-backed Markdown.** The prospect's knowledge lives in Markdown files within a Git repository (GitHub, GitLab, Azure DevOps, or a local Git server). Magpie ingests Markdown, splits documents into cited sections, and publishes proposals back as Git branches and pull requests. A team whose knowledge is not in Markdown or not in Git will require additional migration effort.
- **Knowledge maintenance pain.** Unanswered questions, stale documentation, duplicated content, or difficulty keeping docs in sync with code changes are signals that the gap-detection and auto-drafting loop would deliver value.
- **Technical capability to self-host or access to managed cloud.** Magpie can be self-hosted via Docker Compose (requiring Node.js 22.12, npm, and Postgres) or used through a managed cloud offering. The prospect must have the infrastructure or willingness to run the stack.
- **Authority and budget.** The typical buyer is a technical lead, engineering manager, or knowledge-team stakeholder with the authority to pilot a new tool.


## 3. Discovery and Demonstration

Once qualified, the sales conversation moves to discovery and demonstration. The core narrative is the pitch deck spine: **"Won't lie, won't leak, won't rot."**

### The Narrative

- **Won't lie** — every answer cites file, heading, and commit; the system logs confidence and abstains when unsure. Answers are grounded by construction, not by an LLM that is "usually right."
- **Won't leak** — raw source material (code, internal notes, restricted content) never reaches end users. They see a governed, curated knowledge layer. Every change passes through a reviewed Git pull request with full audit history.
- **Won't rot** — the system detects its own gaps by clustering unanswered questions, drafts proposals to fill them, and raises pull requests. Usage is the maintenance signal. Scheduled patrols verify that existing documents remain accurate against their sources.

### The Demo Flow

A typical demonstration covers:

1. **The elevator pitch.** Magpie is a Git-backed Markdown knowledge maintenance system. It indexes documentation, answers questions with citations, records weak answers to detect knowledge gaps, and automatically drafts Markdown improvements that can be published as pull requests.

2. **Ask questions via MCP.** From inside an AI agent (such as Claude), the prospect sees `kb_ask` return a cited, confident answer from their own documentation. This demonstrates immediate value without changing existing workflows.

3. **The web console.** The console provides sections for Ask, Knowledge, Gaps, Seed, Proposals, Source Map, Jobs, Activity, Insights, Schedules, Config, Data Flow, and MCP. Of particular interest:
   - **Gaps** — shows clusters of unanswered or low-confidence questions that the system has automatically detected.
   - **Proposals** — shows AI-drafted Markdown changes ready for human review, each with per-claim provenance tracing claims back to source files.
   - **Data Flow** — visual architecture diagrams showing how questions flow through retrieval, answering, gap detection, drafting, and publication.

4. **The self-healing loop.** Trace a complete cycle: a user asks a question → the system answers with citations → low-confidence answers become knowledge gaps → similar gaps are clustered → a proposal is drafted → the proposal is published as a pull request for human review → on merge the system re-indexes and verifies the gap is actually closed.

5. **Provider neutrality.** The API never calls a chat model inline. All generative work is queued via pg-boss and completed by a separate watcher process. The system works with any OpenAI-compatible endpoint, Azure OpenAI, or CLI-based agents such as Codex or Claude Code. The prospect is never locked into a single AI vendor.


## 4. Evaluation (Pilot)

The goal of this stage is to let the product speak for itself on a real corpus. The call to action from the pitch deck is to **"pick one pilot corpus and let it run."**

### Setting Up a Pilot

1. The prospect identifies one documentation repository to pilot.
2. The Magpie stack is deployed (self-hosted via Docker Compose, or as a managed instance).
3. Sources and destinations are configured via environment variables (`KNOWLEDGE_SOURCES`, `KNOWLEDGE_DESTINATIONS`, `KNOWLEDGE_FLOWS`).
4. The flow is indexed, and the system begins answering questions.

### What the Pilot Proves

- **Gap detection.** Even from a small corpus, repeated or low-confidence questions will surface knowledge gaps. The console shows what topics are not adequately covered.
- **Proposal quality.** The AI-drafted Markdown proposals carry per-claim provenance, so reviewers can verify every claim against its source.
- **The review workflow.** Proposals land as pull requests (or pushed branches for local-git flows). A human must review and merge — no AI content enters the default branch without approval. This is the primary control against prompt-injection.
- **Self-improvement.** Once a proposal is merged, the merge cascade re-indexes the destination and re-asks the triggering questions to verify the gap is actually closed.

### Duration and Success Criteria

A pilot typically runs for two to four weeks. Success is measured by: the volume of gaps detected, the quality of drafts, the proportion of proposals accepted and merged, and the team's confidence in the system.


## 5. Proposal and Closing

With a successful pilot, the conversation moves to formal adoption.

### Deployment Model Decision

The prospect chooses between:

- **Self-hosted.** The Magpie container image runs on the prospect's own infrastructure via Docker Compose. Components include the API server, web console, watcher, optional MCP server, and Postgres database. The operator sets up TLS termination, configures Auth0 for authentication, enables branch protection on destination repositories, and chooses an AI provider that meets their data-governance requirements.
- **Managed cloud.** A hosted Magpie instance provided and maintained by the vendor.

### Pricing

Pricing covers self-hosted free usage and managed cloud plans. The exact price points are documented separately.

### Security and Compliance

The security review pack documents data flows, controls, and an operator hardening checklist. Key items for the prospect's IT and security teams:

- Authentication fails closed (auth is required unless explicitly disabled; a misconfigured deployment stays secure).
- The API never calls a chat model inline — all generative work is queued.
- No user identifiers are stored with questions.
- The mandatory human review gate is the primary control against prompt injection.
- The operator must choose a compliant AI provider: self-hosted models (via an OpenAI-compatible endpoint) or Azure OpenAI keep data within contractual boundaries.

### Closing

The decision to adopt is typically made by a technical decision-maker who has seen the pilot results and verified that Magpie meets the organisation's documentation maintenance needs within its security and compliance constraints.


## 6. Onboarding

Once the deal is closed, the onboarding process ensures the system is set up correctly for ongoing use.

### Infrastructure Setup

1. Deploy the stack using Docker Compose with production configuration (TLS termination, strong Postgres credentials, restricted CORS, Auth0 integration).
2. Configure the environment: `DATABASE_URL`, `AI_PROVIDER` with matching credentials, `KNOWLEDGE_SOURCES`, `KNOWLEDGE_DESTINATIONS`, `KNOWLEDGE_FLOWS`.
3. Set `AUTH_REQUIRED=true` and configure Auth0 with real credentials. Enable flow-scoped authorization via `KNOWLEDGE_ROLE_GRANTS` if needed.
4. Enable branch protection on destination repositories — require human approval, disallow direct pushes, and ensure the bot account cannot approve or merge its own PRs.

### Team Training

1. **Asking questions.** End users can ask questions through the web console, the REST API, or via MCP tools (`kb_ask`) from within an AI agent.
2. **Reviewing proposals.** Knowledge maintainers learn to review proposals in the console, check per-claim provenance, edit if needed, and approve publication.
3. **Using the Seed page.** To bootstrap a new area of knowledge, the team learns the plan-centric seeding workflow: propose a plan (via `kb_outline` or the Seed page), review and edit the charter, persona, and document plan, then approve to generate drafts.
4. **Monitoring the system.** The Jobs, Activity, Insights, and Schedules pages provide visibility into what the system is doing and the health of the maintenance pipeline.

### Ongoing Operation

- The scheduled maintenance tasks (gap drafting, source sync, snapshot refresh, correctness patrol, editorial patrol, seed bootstrap) run automatically on their configured cadences.
- The operator can monitor watcher health, review maintenance run history on the Schedules page, and adjust configuration as needed.
- When a gap-closure verification fails, the system parks the question for human attention rather than endlessly re-drafting.


## Summary of Stages

| Stage | Objective | Key Activities |
|---|---|---|
| Lead generation | Identify teams with Git-backed Markdown docs and knowledge maintenance pain | Signal detection from engineering teams, documentation teams, knowledge-intensive functions |
| Qualification | Confirm the prospect fits Magpie's model | Assess Git-backed Markdown presence, knowledge pain, self-hosting capability, authority/budget |
| Discovery and demo | Demonstrate the product's value narrative | Present "Won't lie, won't leak, won't rot" spine, live MCP demo, web console walkthrough, self-healing loop explanation |
| Evaluation / pilot | Let the product prove itself on a real corpus | Deploy pilot on one repository, observe gap detection and proposal quality, validate the review workflow |
| Proposal and closing | Agree deployment model and terms | Decide self-hosted vs managed cloud, address security and compliance, finalise pricing |
| Onboarding | Set up production infrastructure and train the team | Deploy stack, configure flows and auth, train users and maintainers, enable branch protection |