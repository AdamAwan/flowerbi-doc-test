---
title: Authentication, Authorization, and Row-Level Security
status: draft
---

# Authentication, Authorization, and Row-Level Security

FlowerBI is authentication- and authorization-agnostic. This guide explains how to integrate your own security infrastructure, enforce row-level security (RLS), and implement multi-tenancy.

## Overview

- **Client authentication**: Provide a `fetch` function that adds credentials (JWT, cookies, etc.).
- **Server authorization**: Protect the `/query` endpoint with ASP.NET Core `[Authorize]`.
- **Row-level security**: Inject extra filters on the server before query execution, based on the current user.
- **Multi-tenancy**: Use tenant discriminators in fact tables and inject a `TenantId` filter.

## Authentication (Client Side)

Supply a `QueryFetch` function that attaches authentication headers. Example:
```ts
import { useQuery } from "flowerbi-react";

const fetchWithAuth: QueryFetch = async (queryJson) => {
  const response = await fetch("https://your-api/query", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${getAccessToken()}`,
    },
    body: JSON.stringify(queryJson),
  });
  return JSON.parse(await response.text(), jsonDateParser);
};
```

## Authorization (Server Side)

Protect the endpoint with `[Authorize]` and inspect `HttpContext.User`. You can reject queries entirely or filter columns based on roles.

## Row-Level Security (RLS)

Add filters to the `QueryJson` before passing it to the engine. Example:
```csharp
queryJson.Filters.Add(new Filter
{
    Column = "Tenant.Id",
    Operator = "=",
    Value = currentUserTenantId
});
```

For complex RLS (multiple conditions, role-based), build a filter tree and inject it server-side.

## Multi-Tenancy

- **Shared database with tenant discriminator**: Use the RLS filter pattern above.
- **Single database per tenant**: Switch connection strings per request.

## User Identity and Profiles

FlowerBI does not store user profiles. Manage them externally (ASP.NET Core Identity, OAuth provider) and derive identity from `HttpContext.User`.

## See Also

- [Server Setup: .NET Configuration and Query Endpoint](server-setup-net-configuration-and-query-endpoint.md)
- [YAML Schemas](yaml.md)

*Consolidated from: authentication-authorization-and-row-level-security-in-flowe.md, authentication-authorization-and-row-level-security.md, row-level-security-and-multi-tenancy-in-flowerbi.md*