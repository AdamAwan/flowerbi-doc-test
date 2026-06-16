---
title: "What is FlowerBI?"
status: draft
---

# What is FlowerBI?

FlowerBI is an open-source, ultra-minimal Business Intelligence (BI) analytics tool. It supports querying relational databases through a single POST route from a client application. Key features include:

- **Client‑side query definition** – queries are written in TypeScript/JavaScript using a concise DSL that specifies columns to group by and aggregate over (e.g., `Bug.Id.count()`).
- **Automatic SQL generation** – the server-side `FlowerBI.Engine` (written in C#) translates the query into SQL, handling joins, grouping, and aggregation automatically based on a declarative YAML schema.
- **Strong typing** – the schema definition is used to generate TypeScript and C# types, giving full IDE autocompletion and type safety.
- **React integration** – packages such as `flowerbi-react` provide React hooks (`useFlowerBI`, `usePageFilters`) and components (`FlowerBITable`) to easily bind query results to UI elements.
- **Security** – the server can add per‑user filters for row‑level security, and the client is only allowed to reference tables/columns defined in the server’s schema.

FlowerBI is particularly suited for applications that need to display aggregated charts and tables from a star‑schema database while keeping the server API simple and controlled.

For more details, see the [project README](https://github.com/danielearwicker/flowerbi) and the [YAML Schema documentation](./yaml.md).