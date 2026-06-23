---
title: Describing Database Structure in FlowerBI YAML Schemas
status: draft
---

# Describing Database Structure in FlowerBI YAML Schemas

FlowerBI uses a YAML schema file to define the tables, columns, relationships, and other metadata that the query engine uses to generate SQL. This article covers the full syntax for describing tables, foreign keys, virtual (derived) tables, and how columns can be used in aggregations.

## Basic Table Definition

Each table is a top-level key under `tables:`. The simplest table has an `id` block (defining the primary key) and a `columns` block listing its scalar columns and foreign keys.

```yaml
schema: MySchema
tables:
    Customer:
        id:
            Id: [int]
        columns:
            CustomerName: [string]
            Email: [string]
            CreatedDate: [datetime]
```

- The `id` block must contain exactly one column; that column becomes the table's primary key.
- The column value is a YAML list: the first element is the data type, and an optional second element specifies the physical database column name if it differs from the logical name. See [Physical Column Names](#physical-column-names).

## Built‑in Data Types

FlowerBI supports these simple scalar types:

- `bool`
- `byte`
- `short`
- `int`
- `long`
- `float`
- `double`
- `decimal`
- `string`
- `datetime`

## Foreign Keys

To define a many‑to‑one relationship, set the column’s type to the name of the referenced table. The column must have the same data type as the primary key of that table (typically `int` or `string`).

```yaml
Invoice:
    id:
        Id: [int]
    columns:
        CustomerId: [Customer]          # FK to Customer.Id
        Amount: [decimal]
        Paid: [bool?]                   # nullable column
```

- The type name `[Customer]` tells FlowerBI that `Invoice.CustomerId` is a foreign key referencing `Customer.Id`.
- Nullable columns are indicated by appending `?` after the type, e.g. `[bool?]` or `[Customer?]`.

## Physical Table and Column Names

By default, the logical table name (the YAML key) is used as the physical database table name. To override it, add a `name` property:

```yaml
Vendor:
    name: Supplier      # physical table name is Supplier
    id:
        Id: [int]
    columns:
        VendorName: [string, SupplierName]  # physical column is SupplierName
```

Similarly, a column’s physical name can be given as a second element in the type list: `[string, SupplierName]`. If omitted, the logical column name is used.

## Virtual Tables (Derived Tables) Through `extends`

Virtual tables are logical aliases for a real physical table. They are declared using the `extends` keyword, which inherits all columns and the primary key from the parent table. Virtual tables are essential for distinguishing multiple foreign keys that point to the same physical table (e.g., `RequestedDate` and `CompletedDate` both use the `Date` table).

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
        RequestedDate: [DateRequested]    # FK to virtual table DateRequested
        CompletedDate: [DateCompleted]    # FK to virtual table DateCompleted
```

- `DateRequested` and `DateCompleted` share the same physical table `Date`, but FlowerBI treats them as separate join subjects, allowing filters on `DateRequested.Id` to affect only the `RequestedDate` column.
- If a virtual table needs a different physical table name, provide `name:` explicitly:

```yaml
DateReported:
    extends: Date
    name: DateReported
```

## Associative Tables (Many‑to‑Many Hints)

For many‑to‑many relationships, you declare an **associative** (junction) table with two foreign keys. Normally FlowerBI can infer the correct join path, but if multiple paths exist (e.g., both via the associative table and another shared dimension), you can add the `associative` property to force the engine to keep the junction table in the join graph.

```yaml
BlogPostTag:
    columns:
        BlogPostId: [BlogPost]
        TagId: [Tag]
    associative: [BlogPostId, TagId]
```

The `associative` value is a list of the column names that form the link. This prevents the table from being eliminated during join path inference.

## Conjoint Tables

Conjoint tables allow you to create “virtual copies” of a table on the fly within a single query, without declaring multiple `extends` tables in the schema. This is useful for schemas where the same logical table (e.g., `AnnotationValue`) appears multiple times in a query with different roles.

```yaml
AnnotationName:
    conjoint: true
    id:
        Id: [int]
    columns:
        Name: [string]

AnnotationValue:
    conjoint: true
    id:
        Id: [int]
    columns:
        AnnotationNameId: [AnnotationName]
        Value: [string]
```

In a query you can suffix a column reference with `@tag` (any identifier) to create an isolated instance of the conjoint table. See the [Conjoint Tables](../docs/markdown/conjoint.md) documentation for full details.

## Columns Used in Aggregations

Aggregations (`count`, `sum`, `average`) are defined in **client queries**, not in the YAML schema. However, the schema defines which columns can meaningfully be aggregated. Numeric columns (`int`, `decimal`, `float`, etc.) can be summed or averaged. Foreign key columns can be counted. The schema does not restrict aggregation; any numeric or FK column is aggregable.

For example, in a client query:

```ts
const { records } = useQuery(fetch, {
    select: {
        customer: Customer.CustomerName,
        invoiceCount: Invoice.Id.count(),
        totalAmount: Invoice.Amount.sum(),
    }
});
```

The `.count()`, `.sum()`, and `.average()` functions are available on every column reference. The schema defines the column types that make these operations semantically valid.

## Summary

- **Tables** are declared under `tables:` with an `id` and `columns` block.
- **Foreign keys** are defined by setting a column’s type to another table’s name.
- **Virtual tables** are created by `extends: <parentTable>`; they share the physical table but have independent join paths.
- **Associative tables** use the `associative` hint to force inclusion in many‑to‑many joins.
- **Conjoint tables** (`conjoint: true`) enable on‑the‑fly table aliasing in queries.
- **Aggregation** is a client‑side concern; the schema merely provides the column definitions that queries operate on.

All schema files are processed by `Schema.FromYaml(...)` on the server, and the `FlowerBI.Tools` code generator produces strongly typed TypeScript and C# definitions.

## References

- Main README YAML example: [README.md](https://github.com/danielearwicker/flowerbi/blob/master/README.md)
- YAML Schemas: [docs/markdown/yaml.md](https://github.com/danielearwicker/flowerbi/blob/master/docs/markdown/yaml.md)
- Virtual Tables: [docs/markdown/virtual-tables.md](https://github.com/danielearwicker/flowerbi/blob/master/docs/markdown/virtual-tables.md)
- Conjoint Tables: [docs/markdown/conjoint.md](https://github.com/danielearwicker/flowerbi/blob/master/docs/markdown/conjoint.md)
- Many‑to‑Many: [docs/markdown/many-to-many.md](https://github.com/danielearwicker/flowerbi/blob/master/docs/markdown/many-to-many.md)
