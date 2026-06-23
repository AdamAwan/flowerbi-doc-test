---
title: Deploying FlowerBI to Production
status: draft
---

# Deploying FlowerBI to Production

This guide covers the end-to-end process for taking a FlowerBI application from development to a live production environment. FlowerBI consists of two main parts that must be deployed together:

- **Client application** – a React-based UI that uses `flowerbi-react` hooks to query data and display charts.
- **API server** – a .NET Core (or .NET 6+) application that hosts the `FlowerBI.Engine` library, accepts POST queries, and runs them against your SQL Server database.

By the end of this guide you will understand how to build, configure, and host both parts in a secure, scalable way.

---

## Prerequisites

- A FlowerBI schema defined in YAML (see [YAML Schemas](./yaml.md)).
- A running SQL Server database with the star-schema tables referenced in your YAML.
- Node.js + Yarn or npm installed for the client build.
- .NET SDK (6.0 or later) for building the server.
- Access to your production environment (e.g., cloud VM, container platform, or static hosting service).

---

## 1. Building the Client Application

The client application is typically a Create React App (CRA) project (see the `demo-site` package). In your client repository, run:

```bash
cd client/packages/demo-site
yarn install
yarn build
```

This produces an optimized production build in the `build/` folder (as documented in [demo-site/README.md](https://github.com/danielearwicker/flowerbi/blob/master/client/packages/demo-site/README.md)). The output includes:
- Minified HTML, CSS, and JavaScript.
- Hashed filenames for cache busting.
- A single `index.html` entry point.

If you have multiple client packages (e.g., separate dashboard apps), repeat the build for each. Alternatively, consider a monorepo approach using Yarn workspaces – the root `client/package.json` already defines workspace packages.

### Customising the Public URL

If you’re hosting the client under a subpath (e.g., `https://yourdomain.com/flowerbi/`), set the `PUBLIC_URL` environment variable before building:

```bash
PUBLIC_URL=/flowerbi/ yarn build
```

The demo-site build script uses `PUBLIC_URL=/flowerbi/demo` (see `client/packages/demo-site/package.json`).

---

## 2. Deploying the Client (Static Hosting)

The built client is a static single-page application (SPA). It can be served from:

- **Any static web server** – Nginx, Apache, or a CDN like AWS S3 + CloudFront, Azure Static Web Apps, Netlify, Vercel, etc.
- **The ASP.NET Core API server itself** – Useful for simple deployments. Configure static file middleware to serve the `build/` folder.

### Example: Serve from ASP.NET Core

In your server’s `Startup.cs` or `Program.cs`:

```csharp
app.UseDefaultFiles();
app.UseStaticFiles();
```

And place the client build output in the server's `wwwroot/` directory.

### Example: Deploy to Azure Storage / CDN

1. Upload the `build/` folder contents to a blob container (static website enabled).
2. Enable a CDN endpoint for global caching.
3. Set `PUBLIC_URL` during build to match the CDN domain/path.

---

## 3. Building the API Server

The server uses `FlowerBI.Engine` to accept query JSON and execute SQL. The source lives under `server/dotnet/`. To build:

```bash
cd server/dotnet
dotnet publish -c Release -o ./publish
```

This produces a self-contained or framework-dependent deployment in the `publish/` folder. Copy this to your production server.

### Configuration

The server needs:
- A connection string to your SQL Server database (read-only access is recommended).
- The YAML schema file(s) that define your data model.
- Optional: authentication middleware to identify the requesting user (for row-level security).

**appsettings.json example:**

```json
{
  "ConnectionStrings": {
    "FlowerBI": "Server=myprodserver.database.windows.net;Database=analytics; ..."
  },
  "FlowerBI": {
    "SchemaPath": "schemas/my-schema.yaml",
    "FullJoins": true
  }
}
```

Set `FullJoins: true` to use symmetrical joins in multi-aggregation queries (recommended – see [Full Joins](./full-joins.md)).

---

## 4. Setting Up the API Route

The server exposes a single POST endpoint – typically `/query` – that accepts `QueryJson` and returns `QueryResultJson`. Implementation example (from `flowerbi/home.md`):

```csharp
[HttpPost("query")]
public async Task<IActionResult> Query([FromBody] QueryJson query)
{
    // Optional: add user-based filters for row-level security
    query.Filters.Add(new Filter { Column = "TenantId", Operator = "=", Value = currentUserId });

    var engine = new FlowerBI.Engine(yourSchema, connectionString);
    var result = await engine.ExecuteAsync(query);
    return Ok(result);
}
```

Ensure the server is configured to accept JSON and enforce request size limits.

---

## 5. Environment-Specific Configuration

Use standard practices:

- **Connection strings**: Store in environment variables or a secrets manager (Azure Key Vault, AWS Secrets Manager).
- **Schema paths**: Point to the correct YAML file per environment (dev/staging/prod).
- **API URLs**: The client must know the server’s URL. In development you might use `http://localhost:5000`. In production, set the absolute URL via environment variable at build time or configure the client’s fetch function dynamically.

The example client fetch function in `flowerbi/home.md` uses `http://localhost:5000/query`. Replace this with your production endpoint, and add authentication headers as needed.

---

## 6. Security Considerations

### Authentication

Protect the `/query` endpoint with your chosen authentication scheme (JWT, OAuth, cookie-based). The client should include credentials in the fetch call:

```ts
const response = await fetch("https://api.yourdomain.com/query", {
    method: "POST",
    headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${token}`
    },
    body: JSON.stringify(queryJson),
});
```

### Row-Level Security (RLS)

As described in the main [README.md](https://github.com/danielearwicker/flowerbi#lock-down-the-schema), add extra filters server-side based on the authenticated user. Example:

```csharp
// Add tenant filter
query.Filters.Add(new Filter
{
    Column = "Tenant.Id",
    Operator = "=",
    Value = userTenantId
});
```

### Network Security

- Use HTTPS only.
- Restrict database access to the API server’s IP address.
- Consider a Web Application Firewall (WAF) in front of the API.

---

## 7. Monitoring and Logging

- Enable ASP.NET Core logging (Application Insights, Serilog, etc.).
- Log query execution times and error rates.
- Monitor database query performance (slow queries, deadlocks).
- Set up health checks for the API endpoint.

---

## 8. Scaling Considerations

FlowerBI’s server is stateless (it only depends on a database connection). You can scale horizontally by:

- Running multiple instances behind a load balancer.
- Using a connection pool with the database.
- Caching schema loading (the YAML schema is read once at startup).

The client is fully static, so it scales by distributing assets via a CDN.

---

## 9. Deployment CI/CD Example (GitHub Actions / Travis)

The repository includes `.travis.yml` as a historic example. A modern pipeline might look like:

```yaml
jobs:
  build:
    steps:
      - run: dotnet publish -c Release server/dotnet
      - run: yarn --cwd client/packages/demo-site build
      - deploy:
          # e.g., Azure WebApp deploy, S3 sync, Docker push
```

---

## 10. Next Steps

- Read the full [YAML Schemas](./yaml.md) reference for schema options.
- Explore [Virtual Tables](./virtual-tables.md) and [Conjoint Tables](./conjoint.md) for advanced modeling.
- Review [External Dependencies](./flowerbi-external-dependencies.md) to ensure your production environment has all required tools/runtimes.

If you encounter issues, refer to the [Contributing guide](./contributing-to-flowerbi.md) or open an issue on GitHub.
