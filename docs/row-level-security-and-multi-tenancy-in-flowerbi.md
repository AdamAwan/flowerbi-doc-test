---
title: Row-Level Security and Multi-Tenancy
status: draft
---

# Row-Level Security and Multi-Tenancy

FlowerBI supports row-level security (RLS) and multi-tenancy by allowing the server to inject mandatory per-user filters **after** receiving a client query and **before** SQL generation. This guide explains the mechanism, how tenant keys are threaded through, and how to implement it in your API.

## Overview

The client sends a query JSON that may include user-chosen filters (e.g., date ranges, category selections). On the server, an RLS layer appends additional filters that are **always applied** — for example, a `TenantId` filter that restricts results to the authenticated user's tenant. These mandatory filters are transparent to the client and cannot be overridden.

## How the Server Injects Filters

1. **Client sends the query** to your API endpoint (typically a single POST route).
2. **Authentication/authorisation** resolves the current user and their tenant(s).
3. **Your API code** calls `FlowerBI.Engine.Execute` with the query JSON and a callback or configuration that adds RLS filters.
4. **Engine generates SQL** that includes both client filters and injected RLS filters — the RLS filters become part of the WHERE clause and are logically ANDed.

### Engine API

The C# `FlowerBI.Engine` class provides an overload that accepts a `Schema` and a `QueryJson`, but also a delegate or a separate filter list for mandatory filters. For example:

```csharp
var engine = new FlowerBI.Engine(schema);
var queryJson = JsonConvert.DeserializeObject<QueryJson>(requestBody);

// Add RLS filter for the current tenant
var tenantFilter = new Filter
{
    Column = "Invoice.TenantId",
    Operator = FilterOperator.Equal,
    Value = currentUser.TenantId.ToString()
};

queryJson.Filters = queryJson.Filters ?? new List<Filter>();
queryJson.Filters.Add(tenantFilter);

var sql = engine.GenerateSql(queryJson);
// Execute the SQL against your database and return results
```

This approach works because `FlowerBI.Engine` treats all filters symmetrically — it has no way to distinguish “client” filters from “server” filters. The server simply prepends/ appends its own filters before passing the query to the engine.

## Threading the Tenant Key

The tenant key (or user ID, or any other security dimension) must be accessible at the point where filters are injected. Typically this is done through the request context:

- **ASP.NET Core:** Use `HttpContext.Items` or dependency injection to retrieve the user’s tenant from a JWT claim or session.
- **Middleware:** A custom middleware can parse the token, set the tenant on the `HttpContext.Items`, and the filter-injection code reads it from there.

Example in an ASP.NET Core controller:

```csharp
[HttpPost("query")]
public async Task<IActionResult> ExecuteQuery([FromBody] QueryJson queryJson)
{
    var tenantId = User.FindFirst("tenant_id")?.Value;
    if (string.IsNullOrEmpty(tenantId))
        return Unauthorized();

    var engine = new FlowerBI.Engine(_schema);
    queryJson.Filters ??= new List<Filter>();
    queryJson.Filters.Add(new Filter
    {
        Column = "Invoice.TenantId",
        Operator = FilterOperator.Equal,
        Value = tenantId
    });

    var sql = engine.GenerateSql(queryJson);
    var results = _dbConnection.ExecuteQuery(sql);
    return Ok(results);
}
```

## Schema Considerations

For RLS to work, your schema must include the columns used for filtering (e.g., `TenantId` in your fact/ dimension tables). You can define these as normal foreign keys or simple columns. It is common to add a `Tenant` table and reference it from relevant tables:

```yaml
tables:
  Tenant:
    id:
      Id: [int]
    columns:
      Name: [string]
  Invoice:
    id:
      Id: [int]
    columns:
      TenantId: [Tenant]    # FK to Tenant
      Amount: [decimal]
```

Then the RLS filter on `Invoice.TenantId` works naturally via join paths.

## Multi-Tenancy Patterns

- **Single database per tenant:** The RLS filter can be omitted entirely; the connection string itself points to the correct database. FlowerBI is connection-agnostic — you just need to pass the correct `IDbConnection` or SQL execution context.
- **Shared database with tenant discriminator:** Use the filter injection described above. The tenant column should ideally be present in all tables that contain sensitive data, and the schema should reflect those foreign keys.
- **Per-user row-level security (not just tenants):** The same mechanism applies — inject filters like `SalesRepId = @currentUserId` or `Region IN (@allowedRegions)`.

## Limitations and Best Practices

- **The injected filter is always ANDed** with client filters. This ensures that a client cannot widen access by omitting the tenant condition.
- **Do not trust the client to provide tenant information** — always derive it from the authentication context.
- **If you use conjoint tables or virtual tables**, ensure the RLS filter references the correct table alias (usually the physical table). The filter column path must match the schema definition (e.g., `Invoice.TenantId`).
- **For performance**, consider indexing the tenant column heavily, as every query will include it.

## Example: Full Server-Side Implementation

See the [Playground demo](https://earwicker.com/flowerbi/demo/) for a simplified in-browser example. In a real .NET API, the code flow is:

1. Authenticate → extract tenant.
2. Deserialize client query.
3. Append mandatory filters (tenant, user scope).
4. Call `Engine.GenerateSql`.
5. Execute SQL via `SqlConnection` (or your preferred O/RM).
6. Return `QueryResultJson`.

This pattern keeps the client simple and secure, while allowing flexible and creative queries within the security boundary.
