---
title: Ideal Buyer Profile for Markdown Magpie
status: draft
---

# Ideal Buyer Profile for Markdown Magpie

Markdown Magpie is a vendor-neutral knowledge maintenance system for Git-backed Markdown documentation. It answers questions with citations, detects knowledge gaps, and automatically proposes Markdown updates via pull requests. This profile defines the industries and company sizes that benefit most from the product.

## Best-Fit Industries

- **Software & SaaS Companies** – Engineering teams that maintain extensive Markdown documentation in Git repositories (e.g., API docs, developer guides, internal runbooks).
- **DevOps & Platform Teams** – Groups that manage infrastructure documentation, on-call runbooks, and architectural decisions in markdown files.
- **Internal Knowledge Management** – Any organisation that uses Git-based wikis or documentation repositories for internal processes, compliance, or training.
- **Tech-Forward Enterprises** – Large companies with multiple product documentation sets where review cycles and Git workflows are already established.
- **Consulting & Professional Services** – Teams that produce client-facing or internal knowledge bases and need to keep them current with minimal manual effort.

## Best-Fit Company Sizes

Small to medium-sized teams (5–50 knowledge contributors) are the primary audience, because they have enough documentation to benefit from automation but lack dedicated technical writing staff. Larger enterprises also benefit, especially those with distributed engineering teams and multiple documentation repositories.

Specifically:
- **Startups (10–50 employees)** – They move fast and often accumulate documentation debt; Magpie’s automated gap detection and proposal generation cuts maintenance overhead.
- **Mid-Market (50–500 employees)** – Teams that have adopted Git and Markdown for docs but struggle to keep content comprehensive and up to date.
- **Enterprise (500+) – Single teams or entire organisations where documentation is spread across many repos; Magpie’s federated source and destination model (see [ingestion.md](../docs/ingestion.md)) lets one deployment serve multiple flows.

## Why These Buyers Fit

Magpie’s core principles align with these profiles:
- **Vendor-neutral** – No lock-in; works with any Git provider (GitHub, GitLab, Azure DevOps).
- **Git-native workflow** – Proposals are pull requests; knowledge is authored and reviewed through the same process as code.
- **Deployment simplicity** – Single Docker Compose file or npm-based local dev; no cloud dependency required for operation.
- **Low cost** – “Cheap & yours” as stated in the pitch deck; no per-seat licensing, only the infrastructure cost of Postgres and optional AI API usage.
- **Extensible** – Provider strategy (mock, OpenAI-compatible, Azure OpenAI) allows cost control and choice (see [chat-providers.md](../docs/chat-providers.md)).

For organisations already using Git and Markdown for documentation, Magpie reduces the effort of maintaining a living knowledge base by automating the detection of gaps and the drafting of updates, while keeping humans in the loop for review.

## Industries That Are Currently Out of Scope

- Physical product documentation (hardware manuals) – unless expressed in Markdown and Git.
- Regulated industries with strict document control (e.g., pharmaceutical, aerospace) – the automated PR workflow may need additional compliance validation before adoption.
- Non-technical teams without Git familiarity – Magpie assumes users are comfortable with Git, Markdown, and PR workflows.

Refer to the [product README](../README.md) and [architecture](../docs/architecture.md) for the full product vision and feature set.
