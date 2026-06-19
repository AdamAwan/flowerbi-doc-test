---
title: Authentication, Authorization, and Row-Level Security in FlowerBI
status: draft
---

# Authentication, Authorization, and Row-Level Security in FlowerBI

FlowerBI is designed to be **authentication- and authorization-agnostic**. It does not provide built-in login, user management, or role-based access control. Instead, it gives you the tools to integrate your own security infrastructure and to enforce **row-level security** (RLS) by adding filters to queries on the server side.

This article explains how to combine FlowerBI with your existing authentication/authorization layer and how to implement per-user data restrictions.

## Overview of FlowerBI’s approach

- **Client‑side authentication**: You supply a `fetch` function that connects to your API. That function is responsible for adding whatever authentication credentials (e.g., JWT tokens, cookies, API keys) your server expects. See [Creating a fetch function](#creating-a-fetch-function).
- **Server‑side authorization**: The .NET server that hosts `FlowerBI.Engine` can examine the authenticated user’s identity and decide which queries are allowed. The server can also **inject extra filters** into every query to enforce row-level security.
- **No built‑in user management**: FlowerBI has no concept of users, roles, or sessions. All of that is delegated to the hosting application (e.g., ASP.NET Core Identity, OAuth, or a custom middleware).

This design keeps FlowerBI lightweight and flexible: you can reuse your existing security model without being forced into a specific one.

---

## Authentication – client side

From the [FlowerBI `flowerbi` README](https://github.com/danielearwicker/flowerbi/blob/master/README.md):

> You supply the `fetch` function to call your API, with your choice of authentication.

The client‑side `useQuery` React hook (or the underlying `flowerbi` query builder) expects a function that sends a POST request with a JSON body (the query definition) and returns a parsed `QueryResultJson`. This is the perfect place to attach authentication headers.

### Example: adding a JWT token

```ts
import { useQuery } from "flowerbi-react";
import { QueryFetch } from "flowerbi";
import { jsonDateParser } from "json-date-parser";

const fetchWithAuth: QueryFetch = async (queryJson) => {
    const response = await fetch("https://your-api/query", {
        method: "POST",
        cache: "no-cache",
        headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${getAccessToken()}`,
        },
        body: JSON.stringify(queryJson),
    });

    if (!response.ok) {
        // Handle auth errors (e.g., redirect to login)
        throw new Error("Query failed: " + response.statusText);
    }

    return JSON.parse(await response.text(), jsonDateParser);
};

// In your component:
const { records } = useQuery(fetchWithAuth, {
    select: { ... },
    // ...
});
```

`getAccessToken()` is your own function that retrieves the current user’s token from wherever you store it (localStorage, memory, etc.).

---

## Authentication & authorization – server side

On the .NET server, you protect the `/query` endpoint using standard ASP.NET Core mechanisms. For example, decorate your controller or minimal API handler with `[Authorize]` or use middleware.

```csharp
[Authorize]
[HttpPost("/query")]
public async Task<IActionResult> Query([FromBody] QueryRequest request)
{
    var user = HttpContext.User; // from JWT or cookie auth
    // ...
}
```

FlowerBI’s `FlowerBI.Engine` does not enforce any authorization itself – it’s up to your application to reject unauthorized requests before they reach the engine.

---

## Row-level security (RLS)

Row-level security means that a user should only see a subset of rows (e.g., only data belonging to their organisation). In FlowerBI, you achieve this by **adding filters to every query** on the server before passing it to the engine.

From the [FlowerBI README](https://github.com/danielearwicker/flowerbi/blob/master/README.md):

> Your API can also easily add extra filters to the query, to impose "row-level security" on a per-user basis.

The engine exposes an `IQuery` interface that allows you to modify the incoming query JSON. The recommended approach is to parse the client’s query, append additional filters based on the current user’s context, and then execute the modified query.

### Implementation steps

1. **Identify the user** – extract user identity from the HTTP context (e.g., claims, roles, tenant ID).
2. **Build the RLS filter** – create a filter expression that restricts data to what the user is allowed to see (e.g., `TenantId = @currentTenant`).
3. **Inject the filter** – modify the `QueryRequest` object (or the deserialized JSON) to include the RLS filter alongside any user-supplied filters.
4. **Execute the query** – pass the modified request to `FlowerBI.Engine`.

### Example server-side code (ASP.NET Core)

Assume your schema has a `Tenant` dimension and every fact table has a foreign key to it. The current user’s tenant ID is stored in a claim named `"tenant_id"`.

```csharp
[Authorize]
[HttpPost("/query")]
public async Task<IActionResult> Query([FromBody] QueryRequest clientQuery)
{
    // Extract tenant ID from the authenticated user
    var tenantId = User.FindFirstValue("tenant_id");
    if (string.IsNullOrEmpty(tenantId))
        return Unauthorized();

    // Add a filter that restricts to this tenant
    var rlsFilter = new Filter
    {
        Column = "Tenant.Id",
        Operator = "=",
        Value = tenantId
    };

    // Combine: ensure the client’s filters are still respected
    clientQuery.Filters = clientQuery.Filters?.Append(rlsFilter).ToArray() ?? new[] { rlsFilter };

    // Execute via FlowerBI.Engine
    var result = await _engine.ExecuteQueryAsync(clientQuery, cancellationToken);
    return Ok(result);
}
```

> **Note**: The exact syntax of `QueryRequest` and `Filter` depends on the version of `FlowerBI.Engine`. Consult the [server setup guide](server-setup-net-configuration-and-query-endpoint.md) for details.

### More complex RLS

- Multiple conditions: `UserRegion IN ('NA', 'EU') AND IsDeleted = false`.
- Role-based: Admins might see all rows, managers see their department, employees see only their own records.
- Dynamic: Use `HttpContext.User` to query a separate authorization database for allowed dimension IDs.

All of these are implemented by constructing the appropriate filter tree in your server code.

---

## User identity and profile management

FlowerBI does not store user profiles or roles. You should manage those in your own system (e.g., ASP.NET Core Identity, a custom database, or an external identity provider). The relationship between user identity and the data they can see is entirely defined by your RLS filters.

For example, if your `User` table contains `UserId`, `TenantId`, and `Role`, you can query that table during request processing to determine what filters to apply. The `User` table itself can also be part of your FlowerBI schema if you want to include user‑related dimensions in reports (but be careful not to expose sensitive data).

---

## Summary

| Aspect | How FlowerBI handles it |
|--------|-------------------------|
| **Authentication** (login) | Not built in. The client’s `fetch` function adds credentials; the server uses standard ASP.NET Core auth. |
| **Authorization** (who can query) | Not built in. The server endpoint should be protected with `[Authorize]` or middleware. |
| **User identity & profiles** | Not built in. Manage users externally and extract identity from the HTTP context. |
| **Row‑level security** | Implemented server‑side by injecting additional filters into the query before execution. |

By keeping security outside the engine, FlowerBI stays lean and lets you use the best practices of your hosting environment.

---

## See also

- [Server Setup: .NET Configuration and Query Endpoint](server-setup-net-configuration-and-query-endpoint.md)
- [YAML Schemas](yaml.md)
- [FlowerBI README](https://github.com/danielearwicker/flowerbi/blob/master/README.md)
