---
title: What is Flower BI?
status: draft
---

FlowerBI is an open-source, ultra-minimal Business Intelligence (BI) analytics query and UI toolkit. It enables clients to query relational databases through a single POST route, supporting aggregation, joins, and strong typing via TypeScript inference. Designed for embedding BI capabilities into custom web applications, it focuses on succinct client-side query definitions, automatic SQL generation, and easy integration with charting libraries like Chart.js. FlowerBI is particularly suited for star-schema databases and provides fine-grained control over schema exposure, row-level security, and multi-tenancy.

Key features include:
- **YAML schema definitions** to describe tables, foreign keys, and virtual/derived tables.
- **Automatic joins and grouping** based on query columns and aggregation functions.
- **Filtered aggregations** to support conditional sums, counts, etc.
- **TypeScript and C# code generation** for compile-time safety.
- **Client- and server-side packages** (`flowerbi`, `flowerbi-react`, `FlowerBI.Engine`).

FlowerBI is maintained by Daniel Earwicker and is released under the MIT license.