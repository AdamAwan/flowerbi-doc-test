---
title: Magpie Pricing and Plans
status: draft
---

# Magpie Pricing and Plans

Magpie is an open-source knowledge maintenance system that you can self-host for free. For teams that prefer a managed experience, optional cloud tiers are available with additional features, support, and scalability.

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

## How to Get Started

- **Self-hosted**: Follow the [Local Development](/docs/local-development) guide in the repository. No sign-up required.
- **Managed tiers**: Visit [app.magpie-knowledge.com](https://app.magpie-knowledge.com) and start your free trial.
- **Enterprise**: Email [sales@magpie-knowledge.com](mailto:sales@magpie-knowledge.com) for a custom quote.

*Last updated: 2026-06-13*