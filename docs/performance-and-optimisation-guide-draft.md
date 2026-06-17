---
title: Performance and Optimisation Guide (Draft)
status: draft
---

# Performance and Optimisation Guide

This page is a draft placeholder for documenting FlowerBI's performance characteristics, caching strategies, and query optimisation features. Currently, no such documentation exists. Future content should include:

- **Benchmarks**: Typical query latency under various data sizes, join complexities, and concurrent loads.
- **Caching**: Whether FlowerBI provides built-in result caching, query plan caching, or relies on external caching (e.g., HTTP caching, database-level caching).
- **Query Optimisation**: How the engine optimises SQL generation (e.g., join elimination, predicate pushdown, use of CTEs vs. subqueries, index recommendations).
- **Configuration**: Any settings that affect performance (e.g., `fullJoins` flag, batch sizes, query timeout).

Until this section is populated, refer to the [main README](../README.md) for general usage and the [YAML Schema Reference](./yaml.md) for schema configuration.

## Contributing

If you have performance data or optimisation insights, please contribute by updating this document. See [Contributing to FlowerBI](../contributing-to-flowerbi.md) for guidelines.
