---
title: Authentication, Authorization, and Row-Level Security
status: draft
---

# Authentication, Authorization, and Row-Level Security

FlowerBI is deliberately agnostic about how you handle authentication and authorization. It does not provide built-in login pages, user stores, or role management. Instead, it gives you the hooks to enforce security at your API layer and to apply row-level restrictions transparently within queries. This guide explains the recommended patterns for integrating FlowerBI with your existing security infrastructure.

## Overview of the Security Model

- **Authentication** – verifying who the user is – is handled entirely by your .NET server (or any middleware you choose). The client (e.g., a React app) authenticates outside of FlowerBI and passes a token or session ID to the server using the `fetch` function you provide.
- **Authorization** – what the user is allowed to do – is enforced by the server when it processes queries. You can inspect the authenticated user’s roles or permissions and decide whether to allow the query, or even modify the query to restrict data.
- **Row‑Level Security (RLS)** – limiting which rows a user can see – is achieved by adding extra filters to the query inside your server endpoint, based on the current user’s identity. FlowerBI’s engine will then apply those filters alongside any client‑supplied filters.

FlowerBI itself never stores user identities or passwords. It trusts the server to tell it whether a request is authorised and what data constraints apply.

## Handling Authentication (Login / User Identity)

Your API endpoint that receives FlowerBI queries can be protected with standard ASP.NET Core authentication middleware. For example, you might use cookie‑based authentication, JWT bearer tokens, or an external identity provider like Azure AD or Auth0.

**Client side** – you supply the `fetch` function that the React hook `useQuery` uses. This function must send the authenticated request. A minimal example with no auth headers:

```ts
import { QueryJson, QueryResultJson } from "flowerbi";

export async function localFetch(queryJson: QueryJson): Promise<QueryResultJson> {
    const response = await fetch("https://your-api.example.com/query", {
        method: "POST",
        cache: "no-cache",
        headers: { 
            "Content-Type": "application/json",
            // Add your authentication header here, e.g.
            // "Authorization": `Bearer ${token}`
        },
        body: JSON.stringify(queryJson),
    });
    return !response.ok ? [] : JSON.parse(await response.text());
}
```

**Server side** – your controller or minimal API endpoint should require authentication (e.g., `[Authorize]` attribute). Inside the endpoint, you can access `HttpContext.User` to get the user’s identity and claims. The FlowerBI engine receives the query JSON and returns results; you can inspect and modify the query before passing it to `Engine.Execute`.

## Implementing Authorization

Authorization can be coarse‑grained (e.g., only admins can run queries) or fine‑grained (e.g., only managers can see revenue data). Because you control the server logic, you can:

- Check whether the user has the `ViewReports` claim before executing the query.
- Reject queries that reference certain tables or columns based on the user’s role.
- Return an empty result or an error if the user is not permitted.

Example of rejecting a query that touches a sensitive column:

```csharp
[HttpPost("query")]
[Authorize]
public async Task<IActionResult> ExecuteQuery([FromBody] QueryJson queryJson)
{
    if (queryJson.Select.Any(s => s.Contains("Salary")) && !User.IsInRole("HR"))
    {
        return Forbid();
    }
    var schema = _schemaProvider.GetSchema();
    var result = FlowerBI.Engine.Engine.Execute(schema, queryJson, ConnectionString);
    return Ok(result);
}
```

## Applying Row‑Level Security (RLS)

Row‑level security is one of the most common requirements described in the [FlowerBI README](https://github.com/danielearwicker/flowerbi#lock-down-the-schema). The idea is simple: **your server adds extra filters to the query** so that every query automatically restricts rows according to the current user’s scope.

For example, in a multi‑tenant application, every data table has a `TenantId` column. You can append a filter like `TenantId = currentTenantId` to every query before execution. This ensures that users never see data belonging to another tenant, even if they try to craft a query that omits such a filter.

```csharp
[HttpPost("query")]
[Authorize]
public async Task<IActionResult> ExecuteQuery([FromBody] QueryJson queryJson)
{
    var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    var tenantId = GetTenantForUser(userId);
    
    // Add a row‑level filter to restrict to the user's tenant
    queryJson.Filters.Add(new FilterClause
    {
        Column = "Tenant.Id",
        Operator = "=",
        Value = tenantId
    });

    var schema = _schemaProvider.GetSchema();
    var result = FlowerBI.Engine.Engine.Execute(schema, queryJson, ConnectionString);
    return Ok(result);
}
```

### Best Practices

- **Always sanitise user‑supplied values** in filters – the engine protects against SQL injection, but you should still validate that the user is allowed to access the columns they reference.
- **Use virtual tables** to apply RLS on filtered date dimensions or other dimensions that could appear multiple times in a query. See [Virtual Tables](virtual-tables.md) for details.
- **Keep the RLS filters separate** from client‑supplied filters – the server is the authority. Do not trust the client to specify its own security constraints.

## Conclusion

FlowerBI does not replace your authentication and authorization infrastructure – it works alongside it. By taking advantage of the server‑side control you have in the .NET API, you can enforce per‑user row‑level security and fine‑grained access control without any changes to the client code. This design keeps FlowerBI minimal while being flexible enough to fit into almost any security model.

### Related Topics

- [Server Setup: .NET Configuration and Query Endpoint](server-setup-net-configuration-and-query-endpoint.md)
- [YAML Schemas](yaml.md)
- [Virtual Tables](virtual-tables.md)
