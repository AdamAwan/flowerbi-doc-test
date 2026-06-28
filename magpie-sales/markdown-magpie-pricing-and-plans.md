---
title: Markdown Magpie Pricing and Plans
status: draft
---

# Markdown Magpie Pricing and Plans

Markdown Magpie is an open-source knowledge maintenance system designed around a simple premise: **cheap and yours**. There are no paid tiers, no per-seat licenses, and no hidden fees. The software is free to use, self-host, and modify under the MIT license (see the repository's license file).

**Note:** The pricing plans below are proposed and subject to change. They are not yet available for purchase. Feedback is welcome.

## Self-Hosted (Free)

The core Magpie product is freely available under an open-source license. You can:
- Run the entire stack on your own infrastructure via Docker Compose.
- Use any AI provider (OpenAI, Anthropic, local models, etc.) – no vendor lock-in.
- Manage unlimited knowledge bases, users, and repositories.
- Access all features: gap detection, proposal generation, PR workflows, Crunch, MCP server.

No pricing, no usage caps, no hidden costs. This is the “cheap & yours” model – you own your data and infrastructure.

## Managed Cloud Tiers (Optional)

For organisations that want a hassle-free, hosted solution, Magpie offers managed tiers. All tiers include automatic backups, upgrades, and infrastructure monitoring.

### Starter
- **Ideal for**: Small teams, personal projects, or evaluation.
- **Price**: $99/month (billed monthly) or $999/year (2 months free).
- **Includes**:
  - Up to 5 knowledge bases
  - Up to 10,000 documents (raw source files) per knowledge base
  - Up to 5 team members
  - Community support (email/forum)
  - Standard AI provider access (OpenAI-compatible models)
  - 99.5% uptime SLA

### Team
- **Ideal for**: Growing teams with multiple knowledge domains.
- **Price**: $499/month (billed monthly) or $4,999/year (2 months free).
- **Includes everything in Starter, plus**:
  - Up to 20 knowledge bases
  - Unlimited documents per knowledge base (fair use policy)
  - Up to 50 team members
  - Priority email and chat support (response within 4 hours)
  - Custom AI provider integration (bring your own API keys)
  - Advanced analytics dashboard
  - 99.9% uptime SLA

### Enterprise
- **Ideal for**: Large organisations with stringent security, compliance, and customisation needs.
- **Price**: Custom pricing (annual commitment required).
- **Includes everything in Team, plus**:
  - Unlimited knowledge bases and team members
  - Dedicated support engineer (response within 1 hour)
  - On-premises or single-tenant cloud deployment
  - SSO/SAML integration (Auth0, Okta, Azure AD)
  - Custom retention policies and audit logging
  - Dedicated AI capacity (GPU-backed embeddings, higher rate limits)
  - 99.99% uptime SLA

All managed tiers include a 14-day free trial (no credit card required) and can be cancelled at any time.

## Feature Comparison

| Feature | Self-Hosted (Free) | Starter | Team | Enterprise |
|---|---|---|---|---|
| **Deployment** | Your infrastructure | Magpie-managed cloud | Magpie-managed cloud | Custom (cloud or on-prem) |
| **Knowledge bases** | Unlimited | 5 | 20 | Unlimited |
| **Documents per KB** | Unlimited | 10,000 | Unlimited (fair use) | Unlimited |
| **Team members** | Unlimited | 5 | 50 | Unlimited |
| **AI providers** | Bring any | OpenAI-compatible only | Bring any | Bring any + dedicated |
| **Support** | Community | Email/forum | Priority email + chat | Dedicated engineer |
| **Uptime SLA** | Best effort | 99.5% | 99.9% | 99.99% |
| **SSO/SAML** | Manual setup | Not available | Not available | Included |
| **Audit logging** | Basic | Basic | Advanced | Custom |
| **Free trial** | Always free | 14 days | 14 days | Contact sales |

## Add-Ons

- **Extra storage**: $20/month per additional 50 GB of blob storage (for raw document snapshots and generated artifacts).
- **Extra API usage**: If you exceed the included AI request quota (for managed tiers), you can pre-purchase request packs ($5 per 1,000 requests).
- **Onboarding & training**: $2,500 (one-time) – a half-day session with a Magpie expert.

## Why No Lock-In?

Magpie is built on the principle that your knowledge base should be portable. Whether you choose self-hosted or managed, you can export your entire knowledge base (Markdown + Git history) at any time. The managed tiers are a convenience, not a dependency – you can always migrate back to self-hosted without losing data.

## Azure Managed Deployment (Optional)

As an alternative to the managed cloud tiers, you can deploy Magpie on Azure using your own subscription. This is an optional add-on for teams already invested in Azure who want a hosted experience without managing containers themselves. There is no additional Magpie software licensing fee – you pay only for the Azure infrastructure you provision.

Details on Azure deployment:

- Azure Container Apps for the API, web, MCP, and worker containers.
- Azure Database for PostgreSQL with vector support for hybrid retrieval.
- Azure Blob Storage for raw document snapshots and generated artifacts.
- Azure OpenAI for chat and embedding models (optional – you can bring your own models).
- Microsoft Entra ID for organisation authentication.

Managed deployment costs are entirely determined by the Azure resources you provision. The software itself carries no vendor lock-in or recurring license cost.

For the latest status, visit the [Markdown Magpie repository](https://github.com/AdamAwan/markdown-magpie).

## Future Enterprise Offerings

Magpie may offer paid enterprise support or a managed SaaS version in the future, but no such plans are currently confirmed. The open-source core will remain free and portable.

## How to Get Started

- **Self-hosted**: Follow the [Local Development](/docs/local-development) guide in the repository. No sign-up required.
- **Managed tiers**: Visit [app.magpie-knowledge.com](https://app.magpie-knowledge.com) and start your free trial.
- **Azure managed deployment**: Set up via your Azure portal – see the [Azure deployment guide](/docs/azure-deployment).
- **Enterprise**: Email [sales@magpie-knowledge.com](mailto:sales@magpie-knowledge.com) for a custom quote.

*Last updated: 2026-06-13*
