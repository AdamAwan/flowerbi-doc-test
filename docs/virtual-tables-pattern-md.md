---
title: Virtual Tables: Logical Table Aliases via extends
status: draft
---

# Virtual Tables: Logical Table Aliases via `extends`

A single database table often holds data that has multiple distinct semantic meanings within a business domain. For example, a work item has both a requested date and a completed date — both date values live in the same physical `Date` table, but they need to be filtered and joined independently. FlowerBI solves this with **virtual tables**: lightweight YAML declarations that clone an existing table's structure under a new name, without creating a new physical table.

## The problem: one table, two meanings

Consider a schema where `WorkItem` has two foreign keys to `Date`:

```yaml
Date:
    id:
        Id: [datetime]
    columns:
        CalendarYearNumber: [short]
        FirstDayOfQuarter: [datetime]
        FirstDayOfMonth: [datetime]

WorkItem:
    id:
        Id: [int]
    columns:
        RequestedDate: [Date]
        CompletedDate: [Date]
```

Both `RequestedDate` and `CompletedDate` reference the same `Date` table. If you apply a filter on `Date.Id`, it constrains both columns at once — which is almost never what you want. You need to treat them as separate dimensions, each filterable on its own terms.

## Declaring a virtual table with `extends`

The solution is to declare virtual table aliases using the `extends` keyword:

```yaml
Date:
    id:
        Id: [datetime]
    columns:
        CalendarYearNumber: [short]
        FirstDayOfQuarter: [datetime]
        FirstDayOfMonth: [datetime]

DateRequested:
    extends: Date

DateCompleted:
    extends: Date

WorkItem:
    id:
        Id: [int]
    columns:
        RequestedDate: [DateRequested]
        CompletedDate: [DateCompleted]
```

`DateRequested` and `DateCompleted` are **virtual tables**. They inherit every column (including the primary key) and the physical database table name from `Date`. In the database, only one `Date` table exists. But within FlowerBI's schema and query model, they are treated as entirely separate join targets.

Now you can filter on `DateRequested.Id` without affecting `DateCompleted.Id`, and vice versa. The generated SQL produces two independent joins to the physical `Date` table, each with its own table alias, so the filter expressions never collide.

## What `extends` inherits

When a table specifies `extends: Parent`, it receives:

- **All columns** from the parent table (id column included)
- **The parent's physical database table name** (`NameInDb`), unless overridden
- **The parent's `doc` and `see`** documentation entries, unless the child provides its own

Columns that the child table declares in its own `columns:` block are added alongside the inherited ones. If a child declares a column with the same name as an inherited column, the child's definition takes precedence.

## Overriding the physical table name with `name`

If a virtual table points to a genuinely different database table that happens to share the same column structure, you can override the physical name:

```yaml
DateRequested:
    extends: Date
    name: DateRequested
```

This tells FlowerBI to generate SQL referencing the `DateRequested` table in the database instead of `Date`. The column structure (id, columns, types) is still inherited from `Date`.

The same override works at the schema level: `name: MySchemaDb` changes the physical schema name without affecting the logical schema name used in queries.

## Foreign keys resolve to virtual table names

Foreign key columns reference the virtual table name, not the physical table. In the example above, `WorkItem.RequestedDate` is typed as `[DateRequested]`. The join engine sees `DateRequested` as a distinct join target from `DateCompleted`, so each foreign key produces its own join clause in the generated SQL.

This means filters, aggregations, and select columns all use the virtual table name in dotted notation, e.g. `DateRequested.CalendarYearNumber` or `DateCompleted.FirstDayOfMonth`.

## Virtual tables versus conjoint tables

FlowerBI provides two mechanisms for giving a table multiple logical identities:

| Aspect | Virtual tables (`extends`) | Conjoint tables (`conjoint: true`) |
|---|---|---|
| Declaration | Explicit YAML entries per alias | Single `conjoint: true` flag in the base table |
| Identity | Distinct named tables in the schema | Single table, labelled on-the-fly via `@suffix` in queries |
| Use case | Fixed, known-in-advance semantic roles (e.g. DateRequested vs DateCompleted) | Ad-hoc, query-time discrimination (e.g. `AnnotationValue.Value@math`) |
| Schema repetition | One YAML block per alias (but minimal content) | Zero additional YAML |
| Foreign keys | FK references the specific virtual table name | FK references the original table; labels applied in the query |

Both approaches produce independent joins to the same physical table. Choose virtual tables when the distinct meanings are known at schema-design time; choose conjoint tables when the labelled variants are determined dynamically by each query.

## Relationship to the many-to-many with common dimensions pattern

In schemas where multiple many-to-many associative tables share a common dimension (such as `Department` linking `Vendor`, `Invoice`, `Category`, `AnnotationName`, and `AnnotationValue` — as seen in the complicated test schema), the same core problem arises: a single physical dimension table appears in multiple join paths. The many-to-many documentation describes how `associative` hints prevent the join engine from eliminating associative tables when a shorter path through a common dimension would create a cartesian product.

Virtual tables extend this idea to dimension tables themselves. Instead of relying on associative hints to keep join paths distinct, you can declare separate virtual aliases for each semantic role a dimension plays. This gives you explicit control over which joins the engine generates and lets you filter each role independently.

For the common-dimensions pattern specifically, virtual tables are best suited when you have a small, fixed set of roles. When the number of roles is unbounded or query-dependent, conjoint tables are the more practical solution — they avoid the need to pre-declare N virtual table stanzas in the YAML schema.

## Summary

- Declare virtual tables with `extends: TableName` to create a schema-level alias for a physical table.
- The alias inherits all columns and the physical table name; override `name` to point to a different DB table.
- Foreign keys reference the virtual table name, making each alias an independent join target.
- Use virtual tables when semantic roles are known at schema-design time; use conjoint tables when they are ad-hoc.
- The pattern complements the associative-table hint system, offering another tool for controlling join behaviour in schemas with shared dimension tables.