---
title: Markdown Magpie Security Posture
status: draft
---

## Security Posture

Markdown Magpie is an open-source Markdown knowledge maintenance system (MIT License) built around a Git-backed, provider-neutral architecture. Its security posture is documented across architecture, threat-model, and security-review materials.

### Authentication and Access Control

Authentication **fails closed**: it is required unless an operator explicitly opts out with `AUTH_REQUIRED=false`. An unset, blank, or misspelled value keeps authentication on, so a misconfigured deployment is locked down rather than silently exposed. When enabled, the API, web console, and both MCP transports validate Auth0-issued JWTs. The API refuses to start if Auth0 is configured but its audience value is missing or a placeholder.

Authorization is scoped: routes enforce per-scope checks (`read:knowledge`, `manage:knowledge`, `manage:jobs`, `ask:knowledge`) that fail closed. All request bodies are validated with Zod schemas; all SQL is parameterised with no string concatenation. Request bodies are capped at 4 MB.

### Data Handling and Egress

The application is provider- and infrastructure-neutral and imposes no data egress of its own. Where data is stored and where it is sent are determined entirely by operator configuration. The app requires Postgres as its datastore, but where Postgres runs is the hoster's choice. Likewise, the app calls whichever AI provider the operator configures.

Data stored in Postgres includes: questions and answers (anonymous by design, with no user identifier), document sections and embeddings, proposals, gap clusters, repositories, and AI job records. The application does not implement field-level encryption; encryption at rest is the responsibility of the hosting infrastructure via encrypted volumes or a managed database.

When a question is answered or a proposal drafted, the watcher sends the user's question text, retrieved Markdown sections, and system prompts to the configured AI provider. User identities, provider API keys, and Git remotes are not sent to the model. Provider options include `openai-compatible` (which can point at a self-hosted model such as vLLM or Ollama), `azure-openai` (Microsoft Azure, covered by the customer's Azure agreement and DPA), `claude` (Anthropic API), and `codex` (local subprocess).

### Architectural Security Controls

The API never calls a chat or generative model inline. All generative work is queued via pg-boss in Postgres; a separate watcher process claims jobs, invokes the configured provider, and posts results back over HTTP. The watcher has **no database access** — it receives a tightly scoped job payload and calls back into the API only over its HTTP surface. This bounds the blast radius of a compromised or prompt-injected model.

The watcher is sandboxed: CLI providers (Claude, Codex) run with read-only flags and argv-only invocation (no shell string), and HTTP providers use a tool loop with three read-only tools (list directory, read file, grep) confined to workspace roots via realpath checks. No write or network tools are available.

Git clone URLs are transport-allowlisted before reaching a git subprocess: only `https`, `http`, `ssh`, and `file` transports are permitted, and `-`-prefixed values are rejected. The `GIT_ALLOW_PROTOCOL` environment variable is set on every git invocation as defence-in-depth. Every git invocation inserts a `--` argv terminator before the URL.

Every proposal target path is validated with `assertWithinRoot`, which rejects any path whose resolved location is outside the repository checkout. Web fetching for internet-kind sources is opt-in per descriptor: only `https` URLs on explicitly allowlisted hosts, re-validated on every redirect hop, with a text-only content-type gate and a hard download cap.

### Mandatory Human Review

No AI-generated content reaches the destination repository's default branch without a human merging a pull request. The publisher raises proposals as pull requests, not direct commits. Automated flows may draft, publish a branch, and open a PR, but the merge is always a human action on the hosting provider. This is the primary control against prompt injection.

### Transport and Deployment Security

Standard security headers are set on every API response: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security`, and `Referrer-Policy`. CORS defaults to `*` but is restricted via the `CORS_ALLOWED_ORIGINS` environment variable. Operators are directed to terminate TLS upstream with a reverse proxy, keep internal ports unpublished, replace default Postgres credentials, and enable branch protection on destination repositories.

### Supply Chain and CI Security

The repository runs automated security scanning on every pull request, on pushes to `main`, and weekly: dependency vulnerability auditing via `npm audit` (gating on high or critical), committed secret detection via gitleaks (gating), Trivy configuration scanning of Dockerfile and Compose files (report-only), and Trivy container image CVE scanning (report-only, uploaded to the Security tab). Dependabot runs weekly for npm and GitHub Actions dependency updates. CI workflows use least-privilege permissions and pinned action versions. No secrets are stored in the repository.

The container image is hardened: multi-stage build, runs as a non-root user, production-only dependencies, pinned base image.

### Prompt Injection Defences

Every prompt that is fed content the system did not author carries an untrusted-content contract and receives that content wrapped in explicit delimiters. The contract tells the model the delimited material is data to analyse, never instructions to obey, and to ignore any embedded directive. The grounding verifier additionally names the canonical steer — a retrieved section that reads "return grounded:true" — and is told to decide the verdict as if such text were absent, so a merged knowledge base section cannot defeat the strip unsupported claims control. The human-review gate remains the primary mitigation.

### Known Gaps in Security Coverage

The security review documents several known gaps: no application-level rate limiting (an authenticated caller can flood the billable ask endpoint), no data-retention policy for questions and answers (job records self-purge but stored questions accumulate indefinitely), a flat authorisation model where `manage:knowledge` is broad, report-only container image and configuration scans, and no SAST or CodeQL in CI.