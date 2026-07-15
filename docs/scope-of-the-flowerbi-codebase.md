---
title: Scope of the FlowerBI Codebase
status: draft
---

FlowerBI is an open-source library for querying relational databases through a single POST route. It enables clients to perform aggregation and joins while the API retains control over what queries can be executed.

## What FlowerBI Is

FlowerBI supports:

- **YAML schema declarations** that define tables, columns, foreign keys, nullability, inheritance via `extends`, conjoint tables, and associative-table hints.
- **TypeScript and C# code generation** from YAML schemas so clients get strongly-typed query builders and auto-completion.
- **Client-side query construction** using a typed DSL with `select`, `aggregations`, `filters`, `orderBy`, and `offset`/`limit` pagination.
- **Server-side SQL generation** that produces optimised queries with automatic joins, grouping, aggregation, and an optional `fullJoins` mode for multi-aggregation queries.
- **Per-aggregation filters** so a single query can return multiple aggregate values (e.g. total bugs and resolved bugs) side by side.
- **Virtual table patterns** via `extends` to disambiguate multiple foreign keys to the same physical table (e.g. `DateRequested` vs `DateCompleted`).
- **Conjoint tables** for auto-virtualisation of repeated annotation-style patterns without manual YAML duplication.
- **Associative tables** for many-to-many relationships, with a hint mechanism to prevent the join engine from eliminating them.
- **Schema documentation** via `doc` and `see` annotations, surfaced in the runtime model and emitted into generated TypeScript JSDoc and C# XML doc comments.
- **A demo Playground** that runs the full stack in-browser using sql.js WASM and Blazor WebAssembly.
- **React integration** through the `flowerbi-react`, `flowerbi-react-chartjs`, and `flowerbi-react-utils` packages.
- **Dapper-based filter parameterisation** that embeds safe literals for numeric/bool types and parameterises others.
- **SQL dialect support** for SQL Server (OFFSET/FETCH NEXT, IIF) and SQLite (LIMIT/OFFSET, IIF) via interchangeable formatters.

## What FlowerBI Does Not Cover

The FlowerBI codebase is focused exclusively on the query-building, schema-definition, and SQL-generation concerns described above. Topics from domains outside that scope — including general-purpose knowledge about animals, popular culture, or other unrelated subjects — are not addressed in the repository.

## Getting Started

The [README](https://github.com/earwicker/flowerbi) describes the use case and provides a quick-start example. The [Playground](https://earwicker.com/flowerbi/demo/) lets users interactively explore query construction, generated SQL, and results.
