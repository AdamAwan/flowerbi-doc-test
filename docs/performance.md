---
title: Performance
status: draft
---

# Performance

This page documents the performance characteristics, caching behaviour, and query optimisation features of FlowerBI. As of this writing, FlowerBI does **not** include a built-in caching layer or explicit query optimisation features beyond what is delegated to the underlying database. All performance is ultimately dependent on the database engine (e.g., SQL Server, SQLite) and the schema design.

## Benchmarks

No official benchmarks are available. Performance should be evaluated by profiling actual queries against your target database with representative data volumes. The generated SQL is straightforward (aggregations, joins, GROUP BY) and generally performs as the database optimiser dictates.

## Caching

FlowerBI does not implement server-side result caching. Each query request triggers a fresh SQL execution. Client-side caching can be added at the application layer if needed (e.g., memoization of query results).

## Query Optimisation

- Query structure (joins, filters, aggregations) is defined entirely by the client; the engine translates it to SQL with no heuristic optimisations.
- The engine does not rewrite queries for performance. It relies on the database’s query planner.
- For complex queries (e.g., multiple aggregations with different filters), consider using the `fullJoins` option to avoid potential data loss (see [Full Joins](./full-joins.md)).
- Schema design (indexes, star schema) is critical. FlowerBI cannot compensate for a poorly optimised database.

## Recommendations

1. Ensure proper indexing on foreign keys and frequently filtered columns.
2. Limit the number of columns in `select` to only those needed.
3. Use filters to reduce data volume before aggregation.
4. Monitor query execution plans on your database.

Future versions may introduce explicit caching or optimisation; this page will be updated accordingly.