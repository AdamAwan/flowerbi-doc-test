---
title: Performance and Scalability of FlowerBI
status: draft
---

# Performance and Scalability of FlowerBI

FlowerBI is designed as an ultra-minimal Business Intelligence toolkit. Its performance characteristics are closely tied to the underlying relational database and the queries it generates. This article covers execution speed, benchmarks, scalability with large data volumes, and caching considerations.

## Execution Model and Overhead

FlowerBI's server-side engine (`FlowerBI.Engine`) performs the following steps for each query (as described in [Server Setup – .NET Configuration and Query Endpoint](server-setup-net-configuration-and-query-endpoint.md)):

1. Parses the client JSON query.
2. Resolves columns and tables against the loaded schema.
3. Generates SQL.
4. Sends the SQL to the database for execution.
5. Returns the result set.

The overhead of steps 1–3 is minimal (parsing and AST manipulation in C#). No in-memory aggregation or post-processing occurs; the actual heavy lifting is done by the database. Therefore, **FlowerBI's raw speed is primarily the speed of the SQL database** it queries. Benchmarks of FlowerBI alone are not meaningful without specifying the database backend (e.g., SQL Server, PostgreSQL, SQLite).

## Scalability with Large Data Volumes

FlowerBI's architecture scales with your database. The engine does not load entire tables into memory; it generates SQL that the database optimizes. For large datasets (millions of rows), the database's indexing, query planner, and hardware resources dominate performance. FlowerBI's generated SQL typically includes joins, aggregations, and grouping—operations that databases handle efficiently when properly indexed.

**No built-in caching** is present in the FlowerBI engine. Each query request triggers a fresh SQL execution. However, client-side caching can be implemented at the application layer:

- The React hook `useFlowerBI` (from `flowerbi-react`) manages query state and can be combined with external caching or memoization.
- The `QueryFetch` function (as shown in [FlowerBI client home.md](client/packages/flowerbi/home.md)) is user-defined; you can add caching logic (e.g., storing results in a Map based on query JSON) to avoid redundant network calls.

For **large data volumes**, consider these best practices:

- Use database-side pagination (though FlowerBI does not natively support pagination; you can add `top`/`limit` via filters or custom SQL).
- Filter data aggressively on the server (e.g., through row-level security or query filters) to reduce result set size.
- Optimize the underlying database indexes on columns used in `select`, `filters`, and `aggregations`.

## Benchmarks and Real-World Observations

The repository does not include official benchmarks. The [Playground demo](https://earwicker.com/flowerbi/demo/) runs entirely in-browser with sql.js (WASM) and a Blazor-compiled engine, which is not representative of production performance. In production, FlowerBI adds negligible latency compared to the database round-trip.

## No Built-In Result Reuse

FlowerBI does not automatically store or reuse previous query results. Each request is independent. The `useFlowerBI` hook does cache query state (e.g., loading/error/data) per component lifecycle, but not across components or page loads. For cross-session reuse, implement a cache layer (e.g., in-memory cache or Redis) in your API, keyed by the query JSON.

## Summary

- **Speed**: Depends on database performance; FlowerBI overhead is minimal.
- **Scalability**: Scales with your database; no inherent limit, but large result sets may require client-side caching or pagination.
- **Caching**: Not built-in; easily added client- or server-side.
- **Benchmarks**: Not provided; performance is database-bound.

For further reading, see the [overview](flowerbi-overview-what-is-flowerbi.md) and [server setup](server-setup-net-configuration-and-query-endpoint.md).