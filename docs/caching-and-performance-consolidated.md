---
title: Caching and Performance
status: draft
---

# Caching and Performance

FlowerBI does not include built-in caching. Performance depends on the database and SQL generated. This guide covers characteristics and caching strategies at various layers.

## Execution Model

Each query results in fresh SQL execution. Overhead is minimal; engine is stateless.

## Client-Side Caching

- `useFlowerBI` hook memoizes by query object (using `json-stable-stringify`).
- Use `react-query`, `SWR`, or `useMemo` to avoid redundant calls.
- Example: `react-query` with `queryKey` based on the query JSON.

## Server-Side Caching

- Implement in-memory or distributed cache (e.g., `IMemoryCache`, Redis) in the API controller.
- Key by hashing the normalized query JSON.
- Be mindful of user-specific filters (include user id in key).

## Database-Level Optimizations

- Index columns used in filters and joins.
- Consider materialized views for repeated aggregations.

## Scalability

- Stateless server scales horizontally.
- Client is static, served via CDN.

## Summary

| Layer | Strategy |
|-------|----------|
| Client | Memoization, react-query |
| Server | In-memory / Redis cache |
| Database | Indexes, materialized views |

*Consolidated from: caching-and-performance.md, caching-and-reusing-query-results-in-flowerbi.md, performance-and-caching-in-flowerbi.md, performance-and-scalability-of-flowerbi.md, performance-characteristics-and-caching-support.md, performance-characteristics-and-scalability.md*