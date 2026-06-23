---
title: Supported Databases and SQL Dialects
status: draft
---

# Supported Databases and SQL Dialects

FlowerBI is designed to work with relational databases, but its SQL generation currently targets **Microsoft SQL Server** and the **T-SQL** dialect. This is the only officially supported database engine at this time. The engine is built in .NET and has no built-in support for PostgreSQL, MySQL, SQLite, or other database systems, although the internal architecture could be extended to support them in the future.

## Current Status

- **Supported database engine:** Microsoft SQL Server
- **Generated SQL dialect:** T-SQL (Transact-SQL)
- **Other databases:** Not supported out-of-the-box. The demo uses [sql.js](https://github.com/sql-js/sql.js) (SQLite compiled to WebAssembly) with heavy workarounds to accommodate T-SQL syntax differences, confirming that SQLite is **not** a first-class target. See the project README: *"[The demo] does some ugly hackery to make the queries compatible, as they are currently generated to target Microsoft SQL Server which has a different syntax for many basic things."*

## Schema Definition

Your YAML schema describes tables and relationships in a database-agnostic way, but the SQL produced by `FlowerBI.Engine` assumes SQL Server conventions. If you attempt to point the engine at a different database, you will likely encounter syntax errors.

## Future Possibilities

The project is open-source, and contributions to support additional dialects (e.g., PostgreSQL, MySQL) are welcome. However, no concrete plans or active development for other databases have been announced.

## Summary

FlowerBI’s SQL generation is tied to **Microsoft SQL Server / T-SQL**. For now, plan your deployment accordingly.