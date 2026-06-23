---
title: Caching and Reusing Query Results in FlowerBI
status: draft
---

# Caching and Reusing Query Results in FlowerBI

FlowerBI is designed to be ultra-minimal and efficient. While it does not include a built-in server-side cache, the client-side React hook `useFlowerBI` (from the `flowerbi-react` package) automatically stores and reuses previous query results to avoid redundant network requests. This section explains how caching works and how you can further optimise performance for large data volumes.

## Client-Side Caching via `useFlowerBI`

The `useFlowerBI` hook manages the state of a query and the data it produces. Internally, it uses the `json-stable-stringify` library (see `flowerbi-react/package.json`) to create a stable string representation of the query object. Whenever the query configuration changes, the hook compares the stringified version with the previous one. If the query is identical, the hook returns the previously fetched result without making a new network request.

```ts
// Example usage – hook automatically caches based on identical query object
const { records, loading } = useFlowerBI(fetch, {
    select: { customer: Customer.CustomerName, bugCount: Bug.Id.count() },
    filters: [Workflow.Resolved.equalTo(true)],
});
```

This behaviour means that if your component re-renders with the same query parameters (e.g., due to state changes that do not affect the query), no additional API call is made. The cached data is returned immediately, keeping the UI responsive.

## Server-Side: No Built-In Caching

The `FlowerBI.Engine` running on the server generates and executes SQL queries on each request. There is no built-in server-side result cache. However, you can implement your own caching layer (e.g., using an in-memory cache like `IMemoryCache` in .NET, or a distributed cache like Redis) by wrapping the engine call. For example:

```csharp
public async Task<QueryResultJson> ExecuteQuery(QueryJson query)
{
    var cacheKey = ComputeHash(query);
    if (_cache.TryGetValue(cacheKey, out QueryResultJson cached))
        return cached;

    var result = await FlowerBI.Engine.ExecuteAsync(query);
    _cache.Set(cacheKey, result, TimeSpan.FromMinutes(5));
    return result;
}
```

Consider the nature of your data – if it changes infrequently, a short-lived cache can reduce database load. For real-time dashboards, you may prefer to skip caching to ensure freshness.

## Network-Level Caching

The example client fetch function in the `flowerbi` package overview (see `flowerbi/home.md`) uses `cache: "no-cache"` to disable HTTP caching. This is intentional – the server should always honour the latest query. If you wish to enable HTTP caching (e.g., for read-only queries that are identical across users), you can modify the fetch call accordingly, but be cautious of stale data.

## Performance with Large Data Volumes

FlowerBI is designed to be fast by generating optimised SQL queries that leverage the database's indexing and aggregation capabilities. Client-side caching of identical queries reduces round-trips. For repeated queries that differ only in parameters (e.g., a date range filter), the client will re-fetch. To further improve perceived speed, you can:

- Combine `useFlowerBI` with a data-fetching library like React Query or SWR for advanced caching, retries, and background refetching.
- Use the `fullJoins` option for multi-aggregation queries to avoid unnecessary null rows (see [Full Joins](./full-joins.md)).
- Ensure your database is properly indexed on columns used in filters and joins.

## Summary

| Layer          | Caching Mechanism                                  | How to Customise               |
| -------------- | -------------------------------------------------- | ------------------------------ |
| **Client**     | Automatic memoization via stable stringify of query | Use external libraries (React Query, SWR) |
| **Server**     | None built-in – implement your own cache           | Wrap engine with `IMemoryCache` / Redis |
| **Network**    | HTTP cache disabled by default                     | Modify fetch headers if needed |

FlowerBI's minimalist design leaves caching decisions to the developer. The built-in client-side caching avoids redundant calls for unchanged queries, making it efficient for typical dashboard use. For heavy or repeated queries, consider adding a server-side cache tuned to your data freshness requirements.
