---
title: Caching and Performance
status: draft
---

# Caching and Performance

FlowerBI is designed to be ultra-minimal and flexible, allowing clients to construct queries that are executed against a relational database via a single POST route. This design is great for rapid development and ad-hoc analysis, but it raises natural questions about performance when handling large data volumes: Does FlowerBI cache query results? How can I reuse or store previous results to avoid redundant database work?

This article explains FlowerBI's built-in caching story (or lack thereof) and provides practical strategies for improving performance through client-side caching, server-side caching, and database optimizations.

## No Built-in Caching

FlowerBI itself does **not** include any caching mechanism. Each call to your API endpoint generates a fresh SQL query and executes it against the database. This is by design: the library focuses on enabling rich, type-safe client queries without imposing a caching layer. The `useFlowerBI` React hook (from `flowerbi-react`) manages the state of a query and the data it produces, but it does not automatically persist or reuse previous results.

> **Source:** The [main README](https://github.com/danielearwicker/flowerbi) describes the single POST route approach. The [`useFlowerBI` documentation](../react-hooks-useflowerbi-and-usepagefilters.md) explains that the hook manages query state but does not mention caching.

## Client-Side Caching Strategies

Since FlowerBI queries are defined as plain JSON, you can easily cache results on the client side. In a React application, you can use standard React techniques such as `useMemo` or `useEffect` with a dependency array that includes the query definition.

### Example: Caching with `useMemo`

```tsx
import { useFlowerBI } from "flowerbi-react";
import { useMemo } from "react";

function MyChart({ filters }) {
  const query = useMemo(() => ({
    select: {
      customer: Customer.CustomerName,
      bugCount: Bug.Id.count(),
    },
    filters: filters,
  }), [filters]);

  const { records, loading } = useFlowerBI(fetch, query);
  // ...
}
```

By keeping the query reference stable (via `useMemo`), `useFlowerBI` will avoid re-fetching as long as the query object hasn't changed. For more aggressive caching, you can store results in a React context or a state management library (e.g., Redux, Zustand) and conditionally skip the fetch.

### Offline Storage

For scenarios where you want to persist query results across sessions, you can serialize the query JSON and its result to `localStorage` or IndexedDB. Be mindful of storage limits and data freshness.

## Server-Side Caching

If your API layer serves multiple clients (or the same client repeatedly with identical queries), you can implement server-side caching. Because each query is represented as a JSON object—and can be serialised to a deterministic string—you can use it as a cache key.

### Example: In-Memory Cache (Conceptual)

```csharp
// Inside your FlowerBI API route
public async Task<IActionResult> Post([FromBody] QueryJson query, [FromServices] ICache cache)
{
    var cacheKey = JsonSerializer.Serialize(query);
    if (cache.TryGetValue(cacheKey, out var cachedResult))
        return Ok(cachedResult);

    var result = await FlowerBI.Engine.RunQueryAsync(query, ...);
    cache.Set(cacheKey, result, TimeSpan.FromMinutes(5));
    return Ok(result);
}
```

**Caveats:**
- If you apply row-level security (adding per-user filters on the server), the cache key should incorporate those filters or you should cache per user.
- Cache invalidation is tricky; use a reasonable TTL or trigger invalidation when underlying data changes.

## Database-Level Optimizations

Even without caching, many performance issues can be mitigated by ensuring your database is well-tuned for FlowerBI queries.

- **Indexes:** Create indexes on columns that appear in `filters` and join conditions (foreign keys). Composite indexes covering multiple grouped columns can help.
- **Materialized Views:** For dashboards that display the same aggregated data repeatedly, consider creating materialized views on the database side. Then point your FlowerBI schema to those views (using the `name` property in YAML to map to the view name).
- **Query Simplification:** Use FlowerBI's built-in features to limit the data volume: apply filters early, select only necessary columns, and avoid multiple aggregations with different filters unless `fullJoins: true` is set (to prevent unexpected row discarding).

## Summary

| Strategy | Where | Effort | Benefit |
|---|---|---|---|
| Client-side memoization | Browser | Low | Prevents duplicate network requests |
| LocalStorage/IndexedDB | Browser | Medium | Persists results across page reloads |
| Server-side cache (Redis, memory) | API | Medium | Reduces database load for repeated queries |
| Database indexes | Database | Low | Speeds up all queries |
| Materialized views | Database | Medium | Pre-computed aggregates |

FlowerBI's minimalism is a strength: it doesn't constrain you to a particular caching solution. You are free to choose the caching strategy that best fits your infrastructure, data sensitivity, and performance requirements. For many applications, simply optimizing the database and using client-side memoisation is sufficient to handle large data volumes.

> **Tip:** Monitor your actual query performance in production. Use database profiling tools to identify slow queries, then apply the appropriate optimisation from this guide.
