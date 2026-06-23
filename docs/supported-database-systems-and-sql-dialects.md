---
title: Supported Database Systems and SQL Dialects
status: draft
---

# Supported Database Systems and SQL Dialects

FlowerBI is designed to query relational databases through a single POST route. The toolkit currently targets **Microsoft SQL Server** and its Transact-SQL (T-SQL) dialect. The primary use case documented in the project README is data stored in SQL Server databases arranged in a star schema. The generated SQL uses T-SQL syntax for joins, aggregation, filtering, and Common Table Expressions (CTEs).

## Current Support

- **Primary database system:** Microsoft SQL Server (all supported versions as of .NET Core 3.1+).
- **SQL dialect:** T-SQL (including CTEs, `FULL JOIN`, `COALESCE`, `GROUP BY`, etc.).
- **Schema definition:** YAML files with types (`int`, `decimal`, `string`, `datetime`, etc.) that map directly to SQL Server column types.

## Adaptability and Notes

FlowerBI's engine (the `FlowerBI.Engine` NuGet package) generates SQL strings and executes them against any ADO.NET-compatible `DbConnection`. While the default SQL generation targets T-SQL, the schema and query model are database-agnostic. In practice, the live demo runs entirely in the browser using **sql.js** (a WASM port of SQLite) with custom hacks to make T-SQL queries compatible. This demonstrates that the engine can be adapted to other dialects, but no officially supported adapters exist at this time.

The following databases and dialects are **not** explicitly supported by the current codebase but could be used with custom SQL transformations:

- PostgreSQL (PL/pgSQL)
- MySQL / MariaDB
- SQLite
- Other relational databases via ADO.NET providers

## Future Plans

There is no published roadmap for adding native support for other databases. Contributions are welcome (see [Contributing to FlowerBI](./contributing-to-flowerbi.md)). If you need to support a different SQL dialect, you would need to modify the SQL generation logic in the `SqlQueryGenerator` class or apply post-processing.

## References

- [Project README – Use Case](https://github.com/danielearwicker/flowerbi#use-case) – states data is in SQL Server databases.
- [Live Demo explanation](https://github.com/danielearwicker/flowerbi#live-demo) – mentions modifying generated SQL to work with sql.js (SQLite).
- [Server Setup – Configuration and Query Endpoint](./server-setup-net-configuration-and-query-endpoint.md) – describes engine executing `engine.ExecuteAsync(db, query)`, where `db` is an `IDbConnection`.
- [YAML Schemas](./yaml.md) – lists built-in data types aligned with SQL Server.