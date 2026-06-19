---
title: Markdown Magpie Pricing and Plans
status: draft
---

# Markdown Magpie Pricing and Plans

Markdown Magpie is an open-source knowledge maintenance system designed around a simple premise: **cheap and yours**. There are no paid tiers, no per-seat licenses, and no hidden fees. The software is free to use, self-host, and modify under the MIT license (see the repository's license file).

## Core Product: Free & Open Source

The entire Markdown Magpie codebase — API, web console, MCP server, watcher, and all packages — is available on GitHub and published under an open-source license. You can clone the repository, run it locally with Docker Compose, and manage your Markdown knowledge bases at zero software cost.

- **No subscription required** — All features including gap detection, proposal generation, and pull request automation are included in the open-source build.
- **No usage caps** — You control the infrastructure; there are no artificial limits on questions, documents, or users.
- **Community support** — Issues and discussions are handled through the public GitHub repository.

## Optional Managed Deployment (Azure)

While the product runs on any infrastructure, Azure is the preferred managed deployment target for teams that want a hosted solution without managing containers, databases, or scaling. This is an **optional add-on**, not a required purchase.

Details on Azure deployment:

- Azure Container Apps for the API, web, MCP, and worker containers.
- Azure Database for PostgreSQL with vector support for hybrid retrieval.
- Azure Blob Storage for raw document snapshots and generated artifacts.
- Azure OpenAI for chat and embedding models.
- Microsoft Entra ID for organisation authentication.

Managed deployment costs are entirely determined by the Azure resources you provision. There is no Markdown Magpie licensing fee on top of Azure infrastructure charges. The pitch deck closes with **"cheap & yours"** — reflecting the philosophy that the software itself carries no vendor lock-in or recurring license cost.

## Future Enterprise Offerings

Markdown Magpie may offer paid enterprise support or a managed SaaS version in the future, but no such plans are currently confirmed. The open-source core will remain free and portable.

## Summary

| Offering | Cost |
|----------|------|
| Open-source software (self-hosted) | Free — MIT license |
| Docker Compose single-host deployment | Free — your own infrastructure |
| Azure managed deployment (optional) | Infrastructure costs only; no software licensing fee |
| Enterprise support / SaaS | Not yet available — check GitHub for updates |

For the latest status, visit the [Markdown Magpie repository](https://github.com/AdamAwan/markdown-magpie).
