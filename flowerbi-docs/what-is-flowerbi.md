---
title: "What is FlowerBI?"
status: draft
---

FlowerBI is an open-source, ultra-minimal Business Intelligence (BI) query engine designed to be embedded into web applications. It allows client-side code (typically TypeScript/React) to query a relational database with strong typing and automatic join/aggregation logic, while the server/API retains control over schema access and row-level security.

## Key Features

- **Client-driven queries** – Define the shape of data you need in concise, type-safe JSON or TypeScript, then send it via a single POST route.
- **Automatic joins and grouping** – The engine reads foreign key relationships from a YAML schema and generates SQL with the correct joins, GROUP BY, and aggregation functions (count, sum, average).
- **Per-aggregation filters** – Apply different filters to different aggregations in the same query, enabling multi-bar charts or comparisons.
- **Schema lockdown** – The YAML schema defines exactly which tables and columns are queryable. The server can add extra filters for row-level security.
- **TypeScript and C# code generation** – `FlowerBI.Tools` emits strongly-typed client libraries and server-side models from the same YAML definition.
- **React integration** – The `flowerbi-react` package provides React hooks (`useQuery`) and components for easy rendering.

## How It Works

1. **Declare a schema** in a YAML file, describing tables, foreign keys, and column types.
2. **Generate client code** (TypeScript constants) and server-side C# models using `FlowerBI.Tools`.
3. **On the server**, configure a `POST` endpoint that passes the incoming JSON query to `FlowerBI.Engine`, which translates it into parameterised SQL and executes it against your database.
4. **On the client**, write queries using the generated helpers:

```ts
const { records } = useQuery(fetch, {
  select: {
    customer: Customer.CustomerName,
    bugCount: Bug.Id.count()
  },
  filters: [Workflow.Resolved.equalTo(true)]
});
```

The result is strongly typed with fields `customer: string` and `bugCount: number`.

## Use Cases

- Embedding interactive dashboards and charts in SaaS applications.
- Reducing the number of custom API endpoints needed for reporting.
- Enforcing strict data access policies while giving front-end developers flexibility.

## More Information

- [GitHub Repository](https://github.com/danielearwicker/flowerbi)
- [Playground / Live Demo](https://earwicker.com/flowerbi/demo/)
- [YAML Schema Reference](https://github.com/danielearwicker/flowerbi/blob/master/docs/markdown/yaml.md)