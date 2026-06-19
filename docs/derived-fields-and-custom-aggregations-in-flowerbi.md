---
title: Derived Fields and Custom Aggregations in FlowerBI
status: draft
---

# Derived Fields and Custom Aggregations in FlowerBI

FlowerBI provides a powerful yet simple declarative schema (YAML) and query DSL for aggregating and grouping data. However, certain advanced scenarios — such as calculated columns that don't exist in the database, or custom aggregation functions beyond `count`, `sum`, and `average` — are not directly supported out of the box. This article explains what is currently possible and how to work around these limitations using existing features or server-side extensions.

## Derived (Computed) Fields

A *derived field* is a column whose value is computed from other columns, rather than stored directly in the database. In the FlowerBI YAML schema, every column defined under `columns:` must correspond to an actual physical column in the underlying database table. There is no syntax for inline expressions like `Amount * 1.1` or `CONCAT(FirstName, ' ', LastName)`.

### Workarounds

1. **Virtual Tables and Extends**
   You can create virtual tables via `extends` to reuse column definitions, but each column still maps to a real database column. For example, if you have a physical `Date` table, you can define `DateReported` and `DateResolved` as virtual tables that share the same physical table – this is useful for semantic clarity, but does not compute new data.

2. **Pre‑computed Columns in the Database**
   The simplest approach is to add the computed column directly in your SQL Server table as a *persisted computed column* (or a view). Then declare it in the YAML schema as a regular column. The schema is just a mapping; any column your database can produce can be referenced.

3. **Client‑Side Calculation**
   After fetching query results, you can perform calculations in your TypeScript/React code. For example, if you need `Profit = Revenue - Cost`, select both columns and compute the difference in the render logic. This is the most flexible approach and does not require server changes.

4. **Server‑Side Post‑Processing**
   The `FlowerBI.Engine` runs inside your API. You can intercept the query result and add extra fields before returning to the client. This keeps the schema clean while allowing arbitrary derived data.

## Custom Aggregation Functions

FlowerBI supports the standard SQL aggregate functions: `count`, `sum`, `average`, `min`, and `max`. These are applied by appending `.count()`, `.sum()`, etc. to a query column. Custom functions like `median`, `weighted_sum`, or `stddev` are not included.

### Using Filtered Aggregations to Simulate Custom Logic

The [filtered aggregations](filtered-aggregations.md) feature lets you provide different filters per aggregation within a single query. This can be used to implement some custom metrics without changing the server. For example, to compute a conditional sum:

```ts
select: {
    customer: Customer.CustomerName,
    totalAmount: Invoice.Amount.sum(),
    highValueAmount: Invoice.Amount.sum([Invoice.Amount.greaterThan(1000)])
}
```

This gives you two sums under different conditions, but the function itself is still `sum`.

### Pre‑Aggregated Tables

For metrics like median, consider pre‑computing them in a database view or materialised table. Then map that table into your FlowerBI schema as a regular fact table. The aggregation logic lives in SQL, not in the query DSL.

### Extending the Engine (Advanced)

The `FlowerBI.Engine` is open‑source (C#). You can add custom aggregate functions by modifying the engine code and registering them. At present there is no plugin system, but the source is straightforward to fork. The engine generates T‑SQL; new aggregate functions would require adding a case in the SQL generation to emit the appropriate `OVER` or `GROUP BY` syntax.

If you need a custom aggregate widely, consider proposing it as a feature or contributing to the project.

## Summary

| Requirement | Approach |
|-------------|----------|
| Derived field (computed column) | Add as persisted computed column in DB, or compute client‑side |
| Custom aggregation (median, weighted sum) | Pre‑compute in DB view, or extend the engine |
| Conditional aggregation | Use filtered aggregations |
| Virtual computed table | Not supported – use DB views or client‑side logic |

FlowerBI intentionally keeps the schema close to the physical database and the aggregation functions limited. This simplicity is a design trade‑off that keeps query generation predictable and secure. For advanced needs, the recommended path is to push computation closer to the data (database views) or to the presentation layer (client code).