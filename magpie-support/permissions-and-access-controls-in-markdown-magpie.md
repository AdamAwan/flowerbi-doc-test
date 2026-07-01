---
title: Permissions and Access Controls in Markdown Magpie
status: draft
---

# Permissions and Access Controls in Markdown Magpie

Markdown Magpie authentication **fails closed**: it is required unless explicitly disabled with `AUTH_REQUIRED=false`. In local development you can opt out by setting `AUTH_REQUIRED=false`. In production and managed deployments, the default is enforced authentication. The system supports both a simple API‑wide authentication mode and a more granular per‑tool permission model for the MCP server.

## Authentication Architecture

Authentication is provided by the `@magpie/auth` package, which validates JSON Web Tokens (JWTs) against an external OpenID Connect provider. The default and recommended identity provider is Auth0 (or Microsoft Entra ID for Azure deployments).

- **Auth0 configuration** – Authentication is required by default (fail-closed). When enabled, the API and MCP server require a valid bearer token issued by the configured Auth0 tenant. To disable authentication, set `AUTH_REQUIRED=false`. The token must carry the expected audience (`AUTH0_AUDIENCE`, default `https://markdown-magpie.local/api`).
- **Microsoft Entra ID** – For Azure deployments, Entra ID can be used as the identity provider (see `infra/azure/README.md`).
- **Local development** – With `AUTH_REQUIRED=false` (explicitly disabling auth), no token is required. All endpoints are open. When running the watcher locally, ensure that its machine-to-machine credentials (`WATCHER_API_CLIENT_ID`, `WATCHER_API_CLIENT_SECRET`, and `MCP_API_AUTH_TOKEN`) are unset or cleared so it does not send an Authorization header to the locally-running API. (The watcher only sends an Authorization header when its M2M credentials are configured.)

### Watcher Authentication

The background watcher process (`@magpie/watcher`) communicates with the API to claim and complete jobs. When authentication is enabled (the default), the watcher must authenticate to the API using a service token. For the default stdio transport, the watcher uses the `MCP_AUTH_TOKEN` environment variable as a bearer token on every API call. For Streamable HTTP deployments, the watcher uses `MCP_API_AUTH_TOKEN` instead. If these tokens are missing when authentication is enabled, the watcher fails fast at startup and cannot claim jobs. In local development with `AUTH_REQUIRED=false`, ensure these credentials are unset to prevent the watcher from sending an Authorization header.

## API‑Level Access Control

The HTTP API (port 4000) currently delegates all permission decisions to the application layer. In the current implementation:

- Most endpoints are **unauthenticated** – they accept any request. The `POST /api/admin/reset` endpoint is explicitly documented as unauthenticated and destructive, and it must not be exposed in production ([api.md](docs/api.md)).
- The API owns the “permissions, retrieval orchestration, proposal creation, and review workflow” ([architecture.md](docs/architecture.md)), but concrete permission checks are not yet implemented for every operation.
- Planned improvements include role‑based access for team members, e.g., `read:knowledge`, `write:knowledge`, `manage:settings`. These will be enforced at the API boundary using the `@magpie/auth` middleware.

## MCP Server Access Control (Granular Permissions)

The MCP server (`apps/mcp`) supports a more mature permission model when authentication is enabled. This is the primary surface for agent‑driven access.

### Transport‑Specific Authentication

- **stdio transport** – The server authenticates to the API using a single service token (`MCP_AUTH_TOKEN`). If authentication is enabled (default) and the token is missing, the server fails fast at startup.
- **Streamable HTTP transport** – The server acts as an OAuth protected resource. It exposes protected‑resource metadata at `/.well-known/oauth-protected-resource` and requires a valid bearer token on every request to the `/mcp` endpoint. Tokens are validated locally against the Auth0 JWKS and are never forwarded to the API. Instead, the HTTP server uses its own machine‑to‑machine credential (`MCP_API_AUTH_TOKEN`) for downstream API calls.

### Per‑Tool Scopes (HTTP Transport)

Each MCP tool requires a specific OAuth scope. The token presented by the client must contain the necessary scope claim. The current mapping is:

| Tool | Required Scope |
|------|----------------|
| `kb.search` | `read:knowledge` |
| `kb.ask` | `ask:knowledge` |
| `kb.feedback` | `feedback:questions` |

If the token lacks the required scope, the server returns a `403 Forbidden` response. Tools that do not require special scopes (e.g., `initialize`, `tools/list`) need only a valid token.

## Web Console Authentication

The web administration console (port 3000) also validates Auth0-issued tokens when authentication is enabled. It uses the `@auth0/auth0-react` React SDK to handle login, logout, and token acquisition. The console requires the following environment variables:

| Variable | Purpose |
|----------|---------|
| `NEXT_PUBLIC_AUTH0_ISSUER` | Auth0 issuer URL for the browser-side SDK. |
| `NEXT_PUBLIC_AUTH0_CLIENT_ID` | Auth0 client ID for the web application. |
| `NEXT_PUBLIC_AUTH0_AUDIENCE` | Expected API audience (must match `AUTH0_AUDIENCE`). |

When `AUTH_REQUIRED=false`, the web console skips authentication and operates without login. In production deployments, set these variables to match the Auth0 application registered for the web console.

### Auth Variables Summary

All authentication variables are shared between the API and MCP server via the `@magpie/auth` package:

| Variable | Default | Purpose |
|----------|---------|---------|
| `AUTH_REQUIRED` | `true` (fail-closed) | Set `false` to disable authentication; any other/unset value keeps it required. |
| `AUTH0_ISSUER_BASE_URL` | – | Full Auth0 issuer (e.g., `https://your-tenant.eu.auth0.com`). |
| `AUTH0_DOMAIN` | – | Alternative to issuer base; issuer becomes `https://<domain>/`. |
| `AUTH0_AUDIENCE` | `https://markdown-magpie.local/api` | API identifier the token must carry. |
| `AUTH0_JWKS_URI` | Derived from issuer | JWKS endpoint for token validation. |
| `MCP_AUTH_TOKEN` | – | stdio only: bearer token presented to the API. Required when auth is enabled. |
| `MCP_API_AUTH_TOKEN` | – | HTTP only: service token for API calls. Required when auth is enabled. |

## Future Directions

- **Role‑based API access** – The architecture document states that “the API owns permissions … and review workflow.” Future iterations will add role‑based checks (e.g., admin, editor, viewer) to API endpoints such as proposal creation, gap management, and system configuration.
- **Team and organization support** – Integration with Microsoft Entra ID or Auth0 organizations will allow scoping permissions to specific teams, ensuring that only authorised team members can modify the knowledge base.
- **Audit logging** – All authentication decisions and permission checks will be logged for compliance and troubleshooting.

## Configuration Examples

### Enable Simple Auth (API + MCP stdio)

```env
AUTH_REQUIRED=true
AUTH0_ISSUER_BASE_URL=https://markdown-magpie.auth0.com
AUTH0_AUDIENCE=https://markdown-magpie.local/api
MCP_AUTH_TOKEN=eyJhbGci...
```

### Enable Per‑Tool Scopes (MCP HTTP)

Add the MCP HTTP server service token and ensure the client tokens contain the appropriate scopes. The HTTP server’s own token is used for API calls and does not need scopes.

```env
AUTH_REQUIRED=true
MCP_API_AUTH_TOKEN=eyJhbGci...
# Also set Auth0 issuer and audience as above.
```

## Summary

- By default, authentication is required (fail-closed). Set `AUTH_REQUIRED=false` to run without authentication for local development.
- In production, ensure Auth0 (or Entra ID) is configured and the required tokens are set.
- The MCP HTTP server provides granular per‑tool scopes for agent access.
- The API is evolving toward full role‑based access control; the current codebase enforces authentication only at the MCP layer and for the admin reset endpoint.
- The web console also validates Auth0 tokens when authentication is enabled, using its own set of environment variables.
- The background watcher authenticates to the API with a service token (`MCP_AUTH_TOKEN` or `MCP_API_AUTH_TOKEN`) when authentication is enabled.

---

*This article covers the permissions and access controls available as of the current release. For the most up‑to‑date information, refer to the [docs/mcp.md](docs/mcp.md) and [docs/architecture.md](docs/architecture.md) source files.*