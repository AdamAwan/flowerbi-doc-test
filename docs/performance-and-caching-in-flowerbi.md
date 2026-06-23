# Performance and Caching in FlowerBI

FlowerBI is designed to be an ultra-minimal BI toolkit. Its performance characteristics are closely tied to the efficiency of the generated SQL and the underlying database. As of the current version, FlowerBI does **not** include built-in caching mechanisms for query results or cache configuration. This article explains the default performance profile, why caching is not part of the core, and how you can add caching at different layers of your application.

## Performance Characteristics

- **No runtime overhead beyond SQL execution**: The engine parses a JSON query, resolves columns, generates SQL, and executes it against the database. There is no intermediate transformation or caching layer that could add latency.
- **Single stateless endpoint**: Query execution is a simple POST request handled by `FlowerBI.Engine`. Each request is processed independently with no retained state between calls.
- **SQL generation is deterministic**: The same input JSON always produces the same SQL, making it safe to cache by query hash.
- **Joins and aggregations happen in the database**: FlowerBI relies on the database’s query optimizer, so performance depends on indexing, schema design, and database tuning (e.g., star schemas with appropriate foreign keys).

Because each query is built from client input, two requests may differ only by a filter value. Without caching, the same aggregate data may be computed repeatedly.

## Why No Built-in Caching?

FlowerBI intentionally keeps the server-side simple. The engine itself is a pure function: given a query string and a database connection, it returns results. It does not manage caches, expiry policies, or invalidation. This design leaves caching decisions to the application layer where you have full control:

- You know the freshness requirements of your data.
- You can choose between memory caches, distributed caches (Redis, SQL Server), or HTTP cache headers.
- You can implement per-user or tenant-aware caching.

The FlowerBI server (ASP.NET Core) is just a thin controller that calls `engine.ExecuteAsync`. Adding caching middleware there is straightforward.

## Adding Caching to Your FlowerBI Application

### 1. Client-Side Caching (React apps)

The `flowerbi-react` package provides a `useQuery` hook. By default, it does not cache results. You can implement a simple memoization layer using `useMemo` or a third-party library like `react-query` or `SWR`. Example with `react-query`:

```tsx
import { useQuery } from '@tanstack/react-query';
import { localFetch } from './fetch';
import { MyQuery } from './schema';

function useFlowerBIQuery(query: MyQuery) {
  return useQuery({
    queryKey: ['flowerbi', query],
    queryFn: () => localFetch(query),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
}
```

The `queryKey` should include the full query object (or a stable hash) to avoid redundant network requests while the user interacts with filters.

### 2. Server-Side Response Caching (ASP.NET Core)

Your FlowerBI API controller can apply `[ResponseCache]` attributes or use middleware like `Microsoft.AspNetCore.ResponseCaching`. Because every query is unique, you must generate a cache key from the request body (JSON). Example using middleware:

```csharp
app.UseResponseCaching();

app.MapPost("/query", async (HttpContext context) =>
{
    // ... parse body, generate SQL, etc.
});
```

You can override the cache key by setting `context.Response.GetTypedHeaders().CacheControl = ...` based on a hash of the query JSON. However, be cautious: cached responses may be served to different users if they construct identical queries; combine with authorization if needed.

### 3. Result Caching in the Engine (Advanced)

If you want to cache results inside `FlowerBI.Engine`, you can wrap the `engine.ExecuteAsync` call with a caching decorator. For example:

```csharp
public class CachedEngine : IQueryEngine
{
    private readonly IQueryEngine _inner;
    private readonly IMemoryCache _cache;

    public async Task<QueryResult> ExecuteAsync(DbConnection db, QueryJson query)
    {
        var cacheKey = ComputeHash(query);
        if (_cache.TryGetValue(cacheKey, out QueryResult cached))
            return cached;

        var result = await _inner.ExecuteAsync(db, query);
        _cache.Set(cacheKey, result, TimeSpan.FromMinutes(10));
        return result;
    }
}
```

Use this approach only when the same query JSON will be repeated frequently and the underlying data changes slowly.

### 4. Database-Level Caching

For enterprise scenarios, consider enabling query result caching in SQL Server (e.g., result set caching in Azure SQL Database). This is transparent to FlowerBI because it only generates standard SQL. Your DBA can set up caching policies at the database level independently.

## Cache Invalidation Considerations

- **User‑specific filters**: If your API adds row‑level security filters, the cache key must incorporate the user identity or the resulting SQL will differ. Include a user identifier in the cache key.
- **Stale data**: Decide acceptable staleness. For near‑real‑time dashboards, disable caching or use a short expiry.
- **Eviction**: Use absolute or sliding expiration. If you implement event-driven invalidation (e.g., when data is updated), you can remove entries by query hash pattern.

## Configuration

FlowerBI does not expose any caching‑related configuration options in its YAML schema or runtime. All caching is application‑level configuration: you choose the strategy, duration, and storage medium.

## Summary

FlowerBI’s performance is good by design, but it does not cache results automatically. Caching is a cross‑cutting concern that fits naturally in your application’s infrastructure layer. The most common approaches are:

1. **Client‑side**: Use a React data‑fetching library with caching.
2. **Server‑side**: Add ASP.NET Core response caching or a custom middleware.
3. **Database‑side**: Leverage native DBMS result caching.

For a minimal setup, sticking with client‑side caching (e.g., `react-query`) is often sufficient and keeps the server stateless. Evaluate your performance needs and add caching where repeat queries are likely.
