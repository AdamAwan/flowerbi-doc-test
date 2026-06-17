---
title: Database Support and Multi-Source Queries
status: draft
---

# Database Support and Multi-Source Queries

## Supported Database Engines / SQL Dialects

FlowerBI is designed to generate SQL queries that are compatible with **Microsoft SQL Server**. The `FlowerBI.Engine` (server-side, .NET) produces SQL in the SQL Server dialect. The project's primary use case, as stated in the README, is "SQL Server databases, arranged in a star schema, served up by our own API (dotnet core/C#)."

However, FlowerBI is not hard-wired to SQL Server. The engine produces parameterized SQL that can be adapted to other relational databases, provided the SQL dialect is compatible or you implement a custom SQL generation step. The demo site uses [sql.js](https://github.com/sql-js/sql.js) (SQLite compiled to WebAssembly) to run entirely in the browser, with some adjustments for incompatible syntax. This proves that the query model is portable, but there is **no built-in support for other dialects** like PostgreSQL, MySQL, or Oracle. To use such databases, you would need to modify the SQL generation logic or apply a transformation layer.

**Currently supported:**
- Microsoft SQL Server (primary target)
- SQLite (via the demo's WASM adaptation)

**Not officially supported:**
- PostgreSQL
- MySQL
- Oracle
- Other SQL dialects

If you require a different database, contributions are welcome – see [Contributing to FlowerBI](./contributing-to-flowerbi.md).

## Multiple Data Sources in a Single Query

FlowerBI is designed to query a **single relational database** per query/report. It does **not** support querying across multiple databases or combining data from different data sources (e.g., another database, a REST API, or a CSV file) within one query.

Each query is executed against one database via a single POST route. The schema (defined in YAML) maps to tables and relationships within that one database. There is no mechanism to federate queries across multiple disparate sources or to perform cross-database joins.

If you need to display data from multiple sources in one report, you must either:
- Extract and consolidate the data into a single database (e.g., via an ETL process), or
- Make separate queries to each source and combine the results in your client application.

## Summary

| Feature | Support |
|---------|---------|
| SQL Server | ✅ Primary target |
| SQLite | ✅ Demo adaptation |
| PostgreSQL / MySQL / Oracle | ❌ Not built-in |
| Multi-source (cross-DB) queries | ❌ Not supported |
| Combining data from different sources in one report | ❌ Must be done client-side or via ETL |

This page will be updated as support evolves.