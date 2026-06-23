---
title: Performance Characteristics and Caching Support
status: draft
---

# Performance Characteristics and Caching Support

FlowerBI is designed as an ultra-minimal BI analytics toolkit that executes queries directly against a relational database on every request. As of the current version, **FlowerBI does not include built-in caching mechanisms** such as result caching, query caching, or caching configuration. This document covers the performance characteristics of the default setup and discusses strategies for adding caching when needed.

## Default Request Flow

When a client sends a query to the `POST` route, the `FlowerBI.Engine` performs the following steps (see [server-setup-net-configuration-and-query-endpoint.md#4-what-flowerbi-engine-does-internally](server-setup-net-configuration-and-query-endpoint.md#4-what-flowerbi-engine-does-internally)):

1. Parse the client JSON query.
2. Resolve columns and tables against the loaded schema.
3. Generate SQL.
4. Execute the SQL against the database.
5. Return the result set.

Each step is stateless; there is no in‑memory cache that stores generated SQL or query results. Every unique request triggers a fresh database round‑trip.

## Performance Considerations

- **Database load**: Repeated identical queries (e.g., dashboard initialisation, polling) will hit the database each time. Without caching, this can increase latency and database pressure.
- **Latency**: Network round‑trips, query generation, and result serialisation contribute to response time. Aggressive caching can reduce p95 latency significantly.
- **Generated SQL**: The engine does not cache generated SQL strings. While SQL generation is fast, caching prepared statements could reduce overhead in high‑throughput scenarios.

## Caching Strategies (Outside FlowerBI)

Although FlowerBI does not provide caching out‑of‑the‑box, you can implement caching at several layers:

### 1. HTTP Response Caching
If your API uses standard HTTP, you can add response cache headers (`Cache-Control`, `ETag`) on the server side. The `FlowerBI.Engine` returns JSON; placing a reverse proxy (e.g., Nginx, Varnish) or using ASP.NET Core `ResponseCaching` middleware can cache responses for identical queries. Note that queries are POST requests, so cache keying must be based on the request body (requires cache implementations that support POST caching).

### 2. Application‑Level Query Cache
Wrap the `engine.ExecuteAsync` call with a caching layer (e.g., using `IMemoryCache` in .NET). The cache key can be a hash of the normalized query JSON. This approach caches results for repeated identical queries, but you must handle cache invalidation when underlying data changes.

### 3. Database‑Level Caching
Rely on the database’s own query plan cache and buffer pool. Repeating identical SQL queries will benefit from cached execution plans and data pages if the database is configured appropriately. This is the simplest passive caching strategy.

### 4. Client‑Side Caching
The `flowerbi-react` package does not implement caching by default, but you can use React’s `useMemo` or a state management library to avoid re‑fetching data for unchanged query parameters. The `useQuery` hook can be wrapped to debounce or cache previous results.

## Sample Configuration for Result Caching (ASP.NET Core)

Below is an illustrative example of adding a simple in‑memory cache to the server endpoint (not part of FlowerBI itself):

```csharp
public async Task<IActionResult> Query([FromBody] QueryJson query)
{
    var cacheKey = ComputeHash(query);
    if (_cache.TryGetValue(cacheKey, out var cached))
        return Ok(cached);

    var result = await _engine.ExecuteAsync(_db, query);
    _cache.Set(cacheKey, result, TimeSpan.FromMinutes(5));
    return Ok(result);
}
```

This pattern is not provided by FlowerBI but can be layered on top without modifying the engine.

## Future Directions

FlowerBI’s roadmap does not currently specify built‑in caching. However, contributions could introduce:
- A caching middleware or plugin interface.
- Automatic cache invalidation via schema‑aware hooks.
- Configurable TTL per query or per table.

Until then, caching remains an external responsibility.

## Summary

- FlowerBI currently has **no built‑in caching support**.
- Each request results in a full query generation and database execution.
- Caching can be added at the HTTP, application, database, or client layer.
- The simplest approach is to cache query results in the application server using a key derived from the query JSON.

For further details on query execution, refer to [server-setup-net-configuration-and-query-endpoint.md](server-setup-net-configuration-and-query-endpoint.md).