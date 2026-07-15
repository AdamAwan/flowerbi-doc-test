---
title: Deploying Markdown Magpie on Azure
status: draft
---

# Deploying Markdown Magpie on Azure

This guide provides a complete, step‑by‑step walkthrough for deploying Markdown Magpie on **Azure** using the infrastructure templates provided in `infra/azure/`. It covers ARM/Bicep templates, networking, scaling, naming conventions, and the post‑deployment configuration steps required to run the service in a production‑ready manner.

> **Prerequisite:** Review the [Getting Started](../getting-started-onboarding-and-indexing-content-into-markdow.md) guide for local setup and environment configuration before attempting a cloud deployment.

---

## 1. Overview of Azure Resources

Markdown Magpie on Azure uses the following managed services:

- **Azure Container Apps** for the API, web UI, MCP, and watcher containers.
- **Azure Database for PostgreSQL (with pgvector)** for the primary database and pg‑boss job queue.
- **Azure Container Registry** to host the Docker images built from the monorepo.
- **Azure Key Vault** (recommended) to store secrets such as API keys and Auth0 credentials.
- **Azure AD / Microsoft Entra ID** for authentication (optional, as an alternative to Auth0).
- **Azure Log Analytics** for monitoring and diagnostics.

The reference architecture is defined in [`infra/azure/README.md`](../infra/azure/README.md) and the corresponding Bicep/ARM templates in `infra/azure/templates/`.

---

## 2. Prerequisites

- An Azure subscription with Contributor or Owner role.
- Azure CLI (`az`) installed and logged in (`az login`).
- Bicep CLI (`az bicep install`) if using `.bicep` files directly.
- Docker installed locally to build and push container images.
- A GitHub account (or other Git provider) for the knowledge base repository syncing.
- Auth0 tenant credentials (or Azure AD application) for authentication (see [Authentication Architecture](../permissions-and-access-controls-in-markdown-magpie.md#authentication-architecture)).
- OpenAI‑compatible API key or Azure OpenAI resource credentials (see [Azure OpenAI provider](../supported-ai-models-and-providers.md#azure-openai)).

---

## 3. Naming Conventions

The templates in `infra/azure/` adhere to the following naming conventions. When customizing, use these patterns consistently:

| Resource               | Pattern                          | Example                                    |
|------------------------|----------------------------------|--------------------------------------------|
| Resource group         | `rg-<env>-<region>-<app>`        | `rg-prod-eastus-magpie`                    |
| Container App Environment | `cae-<env>-<region>-<app>`    | `cae-prod-eastus-magpie`                   |
| Container App          | `ca-<component>-<app>`           | `ca-api-magpie`, `ca-web-magpie`           |
| PostgreSQL server      | `psql-<env>-<region>-<app>`      | `psql-prod-eastus-magpie`                  |
| Container Registry     | `acr<app><env><region>`          | `acrmagpieprodeastus` (lowercase, no hyphens) |
| Key Vault              | `kv-<env>-<region>-<app>`        | `kv-prod-eastus-magpie`                    |
| Log Analytics workspace | `law-<env>-<region>-<app>`      | `law-prod-eastus-magpie`                   |

Region abbreviations follow the Azure standard (e.g., `eastus`, `westeurope`). Environments: `dev`, `staging`, `prod`.

---

## 4. ARM / Bicep Templates

The Bicep templates are located at `infra/azure/templates/`. The main file is `main.bicep` which orchestrates all resources. Key modules:

- **`modules/container-apps.bicep`** – Defines the Container Apps environment and the four container apps (API, Web, Watcher, MCP).
- **`modules/postgres.bicep`** – Deploys Azure Database for PostgreSQL with pgvector extension enabled.
- **`modules/registry.bicep`** – Creates an Azure Container Registry.
- **`modules/networking.bicep`** – Sets up a virtual network with subnets for the Container Apps environment and database private endpoint (if private networking is enabled).
- **`modules/monitoring.bicep`** – Configures Log Analytics and diagnostics.

To deploy using the provided parameters:

```bash
az deployment group create \
  --resource-group rg-prod-eastus-magpie \
  --template-file infra/azure/templates/main.bicep \
  --parameters \
      environmentName=prod \
      location=eastus \
      containerRegistryName=acrmagpieprodeastus \
      postgresAdminPassword='<secure-password>' \
      openAiEndpoint='https://example.openai.azure.com' \
      openAiApiKey='<key>' \
      auth0Domain='<tenant>.auth0.com' \
      auth0Audience='https://api.magpie.example.com' \
      auth0ClientId='<client-id>' \
      logAnalyticsWorkspaceName=law-prod-eastus-magpie
```

Refer to `infra/azure/templates/parameters.example.json` for a full parameter list. Secrets (like API keys) should be injected via Key Vault references in the Bicep file rather than plaintext parameters.

---

## 5. Networking

By default, the Bicep template deploys resources within a virtual network with the following settings:

- **Virtual network** with address space `10.0.0.0/16`.
- **Subnets:**
  - `ContainerAppsSubnet` (`10.0.1.0/24`) – for the Container Apps environment (must be delegated to `Microsoft.App/environments`).
  - `DatabaseSubnet` (`10.0.2.0/24`) – for PostgreSQL private endpoint (if `enablePrivateEndpoint` is `true`).
- **Private endpoint for PostgreSQL** – When enabled, the database is accessible only from within the VNet, reducing public exposure.
- **Network Security Groups (NSGs)** – Applied to subnets to restrict traffic. The Container Apps environment requires outbound internet access for container images and secrets.

If you prefer a simpler setup (public endpoint for PostgreSQL), set `enablePrivateEndpoint: false`; the template then skips the private DNS zone and associated resources.

> **Important:** The Container Apps environment must have `inboundNetworkRestriction` set to `Allow` for the API/Web to be reachable from the internet (or your corporate network). For internal‑only deployments, set `inboundNetworkRestriction` to `Deny` and use Private Endpoints or VPN.

---

## 6. Scaling

Each Azure Container App can scale independently based on HTTP traffic or other metrics.

### API & Web
- **Scale rule:** HTTP requests per second (default target: 10 requests per second per replica).
- **Min replicas:** 1 (always keep one warm instance).
- **Max replicas:** 10 (adjust based on expected load).

### Watcher
- **Scale rule:** Custom queue‑based scaling using the pg‑boss queue depth. The template includes a `keda`‑compatible HTTP scale rule; for async queue‑length scaling, you can configure a KEDA scaler separately.
- **Min replicas:** 0 (can scale to zero when idle).
- **Max replicas:** 3.

### MCP
- **Scale rule:** HTTP requests per second (default target: 5 RPS).
- **Min replicas:** 0 (on‑demand).
- **Max replicas:** 5.

Scaling configuration is in `modules/container-apps.bicep` under each `Microsoft.App/containerApps` resource:

```bicep
scale: {
  minReplicas: 1
  maxReplicas: 10
  rules: [
    {
      name: 'http-rule'
      http: {
        metadata: {
          concurrentRequests: '10'
        }
      }
    }
  ]
}
```

For production, also configure **Azure Monitor autoscale** for the Container App Environment to add additional metrics (CPU, memory).

---

## 7. Container Images

Build and push images to your Azure Container Registry:

```bash
# Login to ACR
az acr login --name acrmagpieprodeastus

# Build all images (use the monorepo build scripts)
npm run build

# Tag and push each app
# Example for the API
docker tag magpie-api:latest acrmagpieprodeastus.azurecr.io/magpie/api:latest
docker push acrmagpieprodeastus.azurecr.io/magpie/api:latest

# Repeat for web, watcher, mcp
```

The Bicep templates reference images by `image` parameter; provide the full URL (e.g., `acrmagpieprodeastus.azurecr.io/magpie/api:latest`). Use specific tags (e.g., commit SHA) for production deployments.

---

## 8. Environment Variables & Secrets

Each container app requires environment variables. The following are critical:

| Variable | Source | Description |
|----------|--------|-------------|
| `DATABASE_URL` | PostgreSQL connection string | Should be a secret (Key Vault reference) |
| `STORAGE_BACKEND` | `postgres` | Always `postgres` for Azure |
| `AI_PROVIDER` | `azure-openai` | Must match the provider configured below |
| `AZURE_OPENAI_ENDPOINT` | Azure OpenAI resource endpoint | e.g., `https://example.openai.azure.com` |
| `AZURE_OPENAI_API_KEY` | Azure OpenAI key | Secret |
| `AZURE_OPENAI_CHAT_DEPLOYMENT` | Deployment name | e.g., `gpt-4o` |
| `AUTH_REQUIRED` | `true` | Enforce authentication |
| `AUTH0_DOMAIN` | Auth0 tenant domain | Or Azure AD tenant ID |
| `AUTH0_AUDIENCE` | API identifier | e.g., `https://api.magpie.example.com` |
| `AUTH0_CLIENT_ID` | Auth0 app client ID | For web frontend |
| `MAGPIE_CHECKOUT_ROOT` | `/data/checkouts` | Path for synced Git repos (mounted volume) |
| `GITHUB_TOKEN` | GitHub PAT | For creating PRs (secret) |
| `MAGPIE_GIT_AUTHOR_NAME` | `Markdown Magpie` | Git commit author name |
| `MAGPIE_GIT_AUTHOR_EMAIL` | `bot@magpie.example.com` | Git commit author email |

In the Bicep template, secrets are referenced from Azure Key Vault using the `keyVaultReference` property on the Container App. Use the `secrets` collection at the container app level and then reference them as environment variables.

---

## 9. Authentication with Azure AD (Microsoft Entra ID)

If you choose Azure AD over Auth0, configure:

- An App Registration in Azure AD with the web redirect URI (`https://<web-app-url>/api/auth/callback`).
- An API scope (e.g., `api://magpie-backend/access_as_user`).
- An application role or delegated permission.

Update the deployment parameters:

```
authProvider=azure-ad
azureAdClientId='<client-id>'
azureAdTenantId='<tenant-id>'
azureAdAudience='api://magpie-backend'
```

The watcher also uses a client credentials flow; create a separate App Registration with a client secret and configure `WATCHER_API_CLIENT_ID` and `WATCHER_API_CLIENT_SECRET`.

See [Authentication Architecture](../permissions-and-access-controls-in-markdown-magpie.md) and the provider documentation for details.

---

## 10. Post‑Deployment Steps

1. **Apply database migrations:** On first deploy, run `npm run db:migrate` inside the API container (or via a one‑off job). The Bicep template includes an init container that runs migrations before the main container starts, but if you need to run them manually:

   ```bash
   az containerapp exec -n ca-api-magpie -g rg-prod-eastus-magpie --command "node scripts/migrate.mjs"
   ```

2. **Configure knowledge flows:** Set `KNOWLEDGE_SOURCES`, `KNOWLEDGE_DESTINATIONS`, and `KNOWLEDGE_FLOWS` in the API’s environment variables (see [Getting Started](../getting-started-onboarding-and-indexing-content-into-markdow.md)).

3. **Index the knowledge base:**

   ```bash
   curl -X POST https://api.magpie.example.com/api/knowledge/repositories/index \
     -H 'Content-Type: application/json' \
     -d '{"flowId":"docs"}'
   ```

4. **Verify health:**

   ```bash
   curl https://api.magpie.example.com/api/health
   curl https://api.magpie.example.com/api/ready
   ```

5. **Secure the deployment:**
   - Enable TLS by adding a custom domain and certificate in the Container App.
   - Restrict network access using IP whitelisting or Private Endpoints.
   - Set up Azure Monitor alerts for job queue depth and error rates.

---

## 11. Troubleshooting

- **Container fails to start:** Check logs via `az containerapp logs show`. Common issues: missing environment variables, incorrect `DATABASE_URL`, or migration script failure.
- **Watcher cannot claim jobs:** Ensure `WATCHER_API_CLIENT_ID` and `WATCHER_API_CLIENT_SECRET` are set correctly and correspond to a service principal with permission to claim jobs.
- **Database connection timeout:** Verify the PostgreSQL firewall rules allow connections from the Container Apps subnet (or use Private Endpoint).
- **Images not found:** Ensure images are pushed to the correct repository and the `image` parameter matches.

For further assistance, refer to `infra/azure/README.md` and the Azure Container Apps documentation.

---

## 12. Reference

- [Azure Container Apps documentation](https://learn.microsoft.com/en-us/azure/container-apps/)
- [Azure Database for PostgreSQL documentation](https://learn.microsoft.com/en-us/azure/postgresql/)
- [Bicep language reference](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Markdown Magpie `infra/azure/README.md`](../infra/azure/README.md)
- [Supported AI models and providers](../supported-ai-models-and-providers.md)
- [Model configuration and integration](../model-configuration-and-integration.md)