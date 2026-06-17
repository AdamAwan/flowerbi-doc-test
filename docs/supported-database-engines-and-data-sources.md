---
title: Supported Database Engines and Data Sources
status: draft
---

# Supported Database Engines and Data Sources

This document clarifies which database engines and SQL dialects `FlowerBI` supports, and whether it can query across multiple data sources in a single report or query.

## Supported Database Engine
`FlowerBI`'s server-side query engine (`FlowerBI.Engine`) generates SQL queries targeting **Microsoft SQL Server** syntax. The engine was built for and is tested against SQL Server, as stated in the project’s README: *“Our users' data is in SQL Server databases… [The engine] currently generate[s] queries to target Microsoft SQL Server.”*

- The generated SQL uses T-SQL features (e.g., CTEs, `COALESCE`, `LEFT JOIN`, `FULL JOIN`).
- The schema YAML definitions map logical table/column names to physical SQL Server tables and columns.
- The client library (`flowerbi`) is database-agnostic – it sends a JSON query payload to any API endpoint. The server side is where dialect matters.

### Other SQL Dialects
**Currently, only SQL Server is officially supported.** While the engine’s SQL generation is not abstracted for multiple dialects, it is possible to adapt the generated SQL manually or use a custom middleware. However, there is no built-in support for PostgreSQL, MySQL, Oracle, or other databases. The demo Playground uses `sql.js` (SQLite via WASM) to run locally in the browser, but that is a testing/demo trick not intended for production — it relies on hacky compatibility adjustments.

## Single Data Source per Query
`FlowerBI` is designed to query a **single relational database** per request. The client sends a JSON query to a single API endpoint, and the server executes one SQL query against one database. There is no built-in capability to combine data from multiple databases or different data sources (e.g., joining a SQL Server table with a PostgreSQL table) in the same query.

If you need to combine data from multiple sources, you would need to extract and load the data into a single database before querying it with `FlowerBI`, or build a custom service that merges results on the client side.

## Summary
| Feature | Support |
|---------|---------|
| SQL Server (T-SQL) | ✅ Primary, fully supported |
| PostgreSQL | ❌ Not built-in |
| MySQL | ❌ Not built-in |
| Other SQL dialects | ❌ Not built-in |
| Multiple databases per query | ❌ Not supported |
| Single relational database per query | ✅ Yes |

For any questions about extending support to other engines, refer to the [contributing guide](./contributing.md) or open an issue on the GitHub repository.