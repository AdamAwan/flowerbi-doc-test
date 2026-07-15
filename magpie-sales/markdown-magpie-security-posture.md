---
title: Markdown Magpie Security Posture
status: draft
---

## Architecture and data isolation

Markdown Magpie is designed so that the API never calls a chat or generative model inline. All generative AI work is enqueued as jobs on a Postgres-backed queue; a separate watcher process claims each job, calls back to the API over HTTP for scoped context, invokes the configured provider, and posts the result back. The watcher has no direct database access, which bounds the blast radius of a compromised or prompt-injected model.

The application is provider- and infrastructure-neutral and imposes no data egress of its own. Where data is stored and where it is sent are determined entirely by operator configuration. The app requires Postgres as its datastore, but where Postgres runs (local, self-hosted, or a managed service) is the hoster’s choice. Likewise, the app calls whichever AI provider the operator configures — it mandates no particular model or vendor. The user’s question text and retrieved Markdown sections are sent to the configured AI provider; user identities, provider API keys, and Git remotes are not sent to the model.

## Authentication and authorisation

Authentication fails closed: it is required unless an operator explicitly opts out by setting `AUTH_REQUIRED=false`. An unset, blank, or misspelled value keeps authentication on, so a misconfigured deployment is locked down rather than silently exposed. When authentication is required, the API refuses to start unless Auth0 is configured with a real audience. The API, web app, and both MCP transports validate Auth0-issued JWTs.

Routes enforce per-scope checks — `read:knowledge`, `manage:knowledge`, `manage:jobs`, `ask:knowledge` — that fail closed. A flow-scoped authorisation layer can be layered on top, granting capabilities (read, manage, ask, admin) per flow via role names carried in the JWT and mapped to flow-level grants in product config. This lets operators restrict who may see or administer proposals and gaps for a given knowledge flow.

## Human review as the primary AI safety control

All AI-generated content reaches a destination repository’s default branch only after a human merges a pull request. Automated flows may draft, publish a branch, and open a PR, but the merge — the step that changes the source of truth — is always a human action on the hosting provider (GitHub, GitLab, or similar). This mandatory human review gate is the primary control against prompt injection. Production deployments must enable branch protection on destination repositories, require at least one human approval, and ensure the bot account cannot approve or merge its own PRs.

## Prompt injection mitigations

Several controls limit the attack surface from untrusted content. The agent-CLI runner spawns the provider with an argv array, never a shell string, so there is no shell to interpret injected metacharacters. The Git publisher validates every proposal target path to reject any path whose resolved location falls outside the repository checkout. Internet-kind sources are reference-only unless the operator opts a descriptor into fetching with an explicit allowlist, enforced with HTTPS-only, exact hostname matching, and a hard download cap.

## Input validation and transport security

Request bodies are validated with Zod schemas. All SQL is parameterised with no string concatenation. Request bodies are capped at 4 MB. Transport security headers set `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, HSTS, and Cross-Origin isolation headers. CORS defaults to `*` but is restricted via configuration.

## Container hardening and supply chain security

The application ships in a hardened container built with a multi-stage Dockerfile: it runs as a non-root user (uid 1001), includes production-only dependencies, and uses a pinned base image. Automated security scanning runs on every pull request, on pushes to the main branch, and weekly. Scans include dependency vulnerability auditing with npm audit (gating on high and critical advisories), committed-secret detection with gitleaks (gating), container image CVE scanning with Trivy (report-only, uploaded to the Security tab), and Dockerfile and Compose misconfiguration scanning. Dependency updates are managed by Dependabot on a weekly schedule. No secrets are stored in the repository; environment files are gitignored and only placeholder templates are tracked, enforced by gitleaks.

## Vulnerability reporting

Security vulnerabilities should be reported privately via GitHub’s private vulnerability reporting tool or by emailing the maintainers directly. The project ships from the main branch; security fixes land there and deployments should track the latest published container image. The maintainers aim to acknowledge reports within a few business days.

## Data storage and encryption

The application stores questions and answers (anonymous, with no user identifier), document sections and embeddings, and operational data such as proposals, gap clusters, and AI job records in Postgres. The application does not implement field-level encryption; encryption at rest is the responsibility of the hosting infrastructure through encrypted volumes or a managed database.

## Deployment hardening expectations

Operators are expected to complete a hardening checklist before production deployment. This includes terminating TLS upstream with a reverse proxy, keeping internal ports on the compose network, replacing default Postgres credentials with strong generated credentials, setting `CORS_ALLOWED_ORIGINS` to the deployment’s web origin, configuring Auth0 with a real identity provider and granting scopes narrowly, enabling branch protection on destination repositories, choosing a compliant AI provider based on data classification, managing secrets outside plaintext env files, and confirming log hygiene.

## Known gaps and roadmap

The documented known gaps include the absence of application-level rate limiting, no defined data-retention policy for questions and answers (job records self-purge but stored content accumulates indefinitely), a currently flat authorisation model, report-only Trivy image and config scans that are not yet gating, and the absence of SAST or CodeQL in CI and a licence-compliance gate.