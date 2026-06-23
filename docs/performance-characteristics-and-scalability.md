---
title: Performance Characteristics and Scalability
status: draft
---

# Performance Characteristics and Scalability

FlowerBI is an ultra‑minimal BI toolkit that delegates most of the heavy lifting to the underlying relational database. This design means its performance characteristics are primarily determined by the database engine, the generated SQL, and the client’s network round‑trip time. Below we discuss execution speed, caching, and behaviour with large data volumes.

## Architecture Overview

A FlowerBI query flows as follows:

1. The client constructs a JSON query object and sends a POST request to a single API endpoint.
2. The server-side `FlowerBI.Engine` parses the query, resolves columns and tables against a loaded schema, and generates a SQL query (e.g. with appropriate joins, aggregations, and filters).
3. The generated SQL is executed against the database via `engine.ExecuteAsync(db, query)` (see [server setup](../server-setup-net-configuration-and-query-endpoint.md#4-what-flowerbiengine-does-internally)).
4. The result set is serialised back to the client.

Because FlowerBI does not perform any server‑side data processing beyond SQL generation, the **execution speed of any query is essentially the speed at which the database can run the generated SQL** and return the result set. The engine itself adds negligible overhead (parsing, schema resolution, and string building).

## Benchmarks

FlowerBI does not include built‑in benchmarks or published performance numbers. The project’s primary goal is developer productivity and safe client‑driven querying. However, the engineering choices – such as generating a single SQL batch per query – ensure that the database optimiser can leverage its own statistics, indexes, and cache. For typical star‑schema workloads (fact table with several dimension tables) the generated SQL is straightforward and performant.

Users are encouraged to profile the generated SQL (visible in the [Playground](https://earwicker.com/flowerbi/demo/)) and use standard database tuning techniques: proper indexing, statistics updates, and limiting the result set with filters or pagination.

## Caching and Reuse of Results

FlowerBI’s server‑side engine **does not implement a result cache**. Every request triggers a fresh SQL execution. This is consistent with its philosophy of being a thin translation layer: the database’s own query cache (if enabled) may apply, but there is no application‑level cache.

On the client side, the `useFlowerBI` React hook (from `flowerbi-react`) manages the state of a query and its results. It does **not** provide built‑in caching of previous responses, but because the hook re‑renders on every parameter change, you can implement your own caching layer – for example, by wrapping the `fetch` function with a memoization helper or using a library like `react-query` to deduplicate identical requests. The core `flowerbi` package exports the `QueryFetch` type, which is agnostic about caching; see the [home page of the client library](../flowerbi-client/home.md) for an example of a fetch function.

If you need to reuse or store previous results (e.g., to avoid re‑querying unchanged filters), the recommended approach is to manage that state outside FlowerBI, for instance in a React context or a global store, and pass the cached records to your UI components.

## Behaviour with Large Data Volumes

When handling large data volumes, consider the following:

- **Database side**: All aggregation and filtering happens in the database. For millions of rows, ensure that the tables have appropriate indexes (especially on foreign keys and filtered columns). The generated SQL uses `GROUP BY` and `JOIN`s, which the optimiser can handle efficiently if indexes exist.
- **Result set size**: The query result sent over the network is limited to the aggregated records (e.g., one row per group). For high‑cardinality grouping columns, the result set can be large; consider adding filters to reduce cardinality.
- **Client side**: The records are returned as JSON and stored in memory. For extremely large result sets, client‑side performance may degrade. You may paginate or limit the query using server‑side filters (e.g., top‑N patterns).
- **Row‑level security**: The engine can inject additional filters per user (e.g., `WHERE TenantId = @currentUserTenant`), which helps narrow the data scope and improve performance.

In summary, FlowerBI is not a bottleneck itself; it relies on the database’s capacity to handle large volumes. Proper database design and query planning are essential.

## Summary

| Aspect | Details |
|--------|---------|
| Execution speed | Depends entirely on the database and the generated SQL; engine overhead is minimal. |
| Benchmarks | No published numbers; users should profile their own workloads. |
| Caching | Not built‑in on the server; client can implement caching via fetch wrapper or state management. |
| Large data volumes | Performance governed by database indexes, result set size, and available bandwidth. |

For further reading, see the [architecture overview](../flowerbi-overview-what-is-flowerbi.md) and the [server execution internals](../server-setup-net-configuration-and-query-endpoint.md#4-what-flowerbiengine-does-internally).