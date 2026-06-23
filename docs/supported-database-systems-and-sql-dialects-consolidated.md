---
title: Supported Database Systems and SQL Dialects
status: draft
---

# Supported Database Systems and SQL Dialects

FlowerBI generates T-SQL for Microsoft SQL Server. Currently, this is the only officially supported dialect.

## Current Support

- **Primary:** Microsoft SQL Server (T-SQL)
- **Other databases:** Not natively supported. The demo uses sql.js (SQLite) with heavy workarounds.

## Adaptability

The engine is database-agnostic at the schema level, but SQL generation is tied to T-SQL. Contributions for other dialects are welcome.

*Consolidated from: supported-database-systems-and-sql-dialects.md, supported-databases-and-sql-dialects.md*