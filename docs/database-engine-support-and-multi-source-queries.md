---
title: Database Engine Support and Multi-Source Queries
status: draft
---

# Database Engine Support and Multi-Source Queries

FlowerBI is designed to work with relational databases. Currently, the primary supported SQL dialect is **Microsoft SQL Server**. The server-side engine (`FlowerBI.Engine`) generates SQL queries that target SQL Server syntax. The project's own use case documentation explicitly states that user data is stored in SQL Server databases and served via a .NET Core/C# API.

## Supported SQL Dialects

- **Microsoft SQL Server** – fully supported and tested. All generated SQL is optimized for SQL Server.
- **SQLite** – used in the [live demo](https://earwicker.com/flowerbi/demo/) via a WASM build of `sql.js`. This is not a production-ready target; the demo uses workarounds to adapt SQL Server–style queries to SQLite syntax. Official production deployment should use SQL Server.

There is **no official support** for PostgreSQL, MySQL, Oracle, or other database engines. The engine's SQL generation is tightly coupled to SQL Server conventions (e.g., `TOP`, `WITH`, bracket-quoted identifiers). Extending to other dialects would require modifications to the `FlowerBI.Engine` or a new SQL dialect adapter.

## Multi-Data Source Queries

FlowerBI does **not** support querying across multiple databases or data sources within a single query or report. Each FlowerBI server instance is configured with a single database connection and a single YAML schema describing that database's tables and relationships. The API endpoint accepts queries that operate exclusively on that one database. There is no built-in mechanism for combining data from different databases, federating queries, or connecting to multiple sources.

If you need to combine data from multiple sources, you would typically need to create a separate API endpoint per source or implement an ETL process to unify the data into a single database before querying it with FlowerBI.

## Future Considerations

While the current implementation is SQL Server–centric, the project is open-source and a community contribution could add support for other SQL dialects. The `FlowerBI.Engine` is written in C# and uses a flexible SQL builder; a pull request to add dialect abstraction would be welcome. Similarly, multi-source querying is not on the current roadmap but could be explored via schema federation or custom middleware.

## Summary

| Database Engine    | Production Support | Notes                                    |
|--------------------|--------------------|------------------------------------------|
| Microsoft SQL Server | ✅ Yes            | Primary target, fully tested.            |
| SQLite             | ❌ No              | Demo only; not suitable for production.  |
| PostgreSQL         | ❌ No              | Not supported – would require adaptation.|
| MySQL              | ❌ No              | Not supported.                           |

- **Multiple data sources in a single query:** Not supported. One database per FlowerBI instance.
- **Recommendation:** Ensure your data resides in a single SQL Server database before using FlowerBI. For multi-source needs, consolidate data upstream.