---
title: Supported Database Engines
status: draft
---

# Supported Database Engines

FlowerBI's query engine generates SQL dynamically based on a YAML-defined schema. As of the current version, the engine is primarily designed for **Microsoft SQL Server**. The generated SQL uses T-SQL syntax and features specific to SQL Server (e.g., `COUNT_BIG`, `DATEPART`, `COALESCE`).

## Currently Supported

- **Microsoft SQL Server** (primary target)

## Other Databases

- **PostgreSQL**, **MySQL**, **SQLite**, and other dialects are **not officially supported** out of the box. The engine does not include dialect-specific formatting for these systems.

- The demo site (using `sql.js`) works only because of custom hackery in the Blazor WASM host to rewrite T-SQL into SQLite-compatible syntax. This is not production-ready and is only for demonstration purposes.

## Provider-Agnostic Design

The core `FlowerBI.Engine` library (C#) separates query logic from SQL generation via an `ISqlDialect` interface (planned or partially implemented). This means that support for additional databases can be added by implementing a new dialect without changing the rest of the engine. However, no such implementations are currently provided.

## Future Plans

- Support for **PostgreSQL** and **MySQL** is a common feature request and is under consideration.
- Contributions for new dialects are welcome (see [Contributing](../contributing-to-flowerbi.md)).

## Recommendations

If you are evaluating FlowerBI for a project that uses a database other than SQL Server, you may need to:

- Fork the repository and implement a custom SQL dialect.
- Use a middleware layer to translate the generated SQL.
- Wait for official support (if planned).

For the most up-to-date information, check the project's issue tracker or the [README](../../README.md).