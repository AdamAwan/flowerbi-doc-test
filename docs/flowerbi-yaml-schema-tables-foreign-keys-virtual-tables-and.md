---
title: "FlowerBI YAML Schema: Tables, Foreign Keys, Virtual Tables, and Data Types"
status: draft
---

# FlowerBI YAML Schema

FlowerBI uses a YAML configuration file to describe the database schema (tables, columns, relationships) that clients can query. This file defines the **logical model** – what tables exist, their primary keys, foreign keys, column types, and optional virtual tables for multi-role dimensions.

## Basic Table Declaration

Every table is declared under `tables:` in the schema file. The simplest form:

```yaml
schema: MySchema
tables:
    Customer:
        id:
            Id: [int]
        columns:
            CustomerName: [string]
            Email: [string?]
```

- `schema`: optional top-level name for the schema.
- `tables`: dictionary of table definitions.
- Each table has an `id` block (primary key) and a `columns` block.
- Column types are specified in brackets: `[type]`. Built-in types: `bool`, `byte`, `short`, `int`, `long`, `float`, `double`, `decimal`, `string`, `datetime`.
- Append `?` for nullable columns: `[string?]`, `[int?]`.

## Mapping to Physical Database Names

If the database table or column name differs from the logical name, provide an alias:

```yaml
Vendor:
    name: Supplier          # physical table name
    id:
        Id: [int]
    columns:
        VendorName: [string, SupplierName]   # logical: VendorName, physical: SupplierName
```

## Foreign Keys

Foreign keys are defined by using another table’s name as the column type. FlowerBI automatically infers joins from these references.

```yaml
Invoice:
    id:
        Id: [int]
    columns:
        VendorId: [Vendor]          # FK to Vendor table
        DepartmentId: [Department]  # FK to Department table
        Amount: [decimal]
```

The foreign key column must have the same data type as the referenced table’s primary key. Nullable FK: `[Vendor?]`.

## Virtual (Derived) Tables

Virtual tables allow you to reuse a physical table for multiple logical roles without duplicating column definitions. Use `extends` to inherit all columns from a base table, then optionally override the physical name.

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
```

`DateRequested` and `DateCompleted` are virtual tables that share the same physical table (`Date`) but are treated as separate tables in queries. You can also assign a different physical name:

```yaml
DateReported:
    extends: Date
    name: DateReported   # physical table name differs from base
```

Virtual tables are the foundation of the [virtual tables pattern](virtual-tables.md) – ideal when the same dimension table appears as multiple foreign keys.

## Extended Column Syntax (Documentation)

Columns can be documented inline using the long form:

```yaml
Amount:
    type: decimal
    name: GrossAmount      # physical column name (optional)
    doc: Gross amount in original currency.
    see: [billing, Currency.IsoCode]
```

Only `type` is required. The short form `[type]` or `[type, physicalName]` is always valid. See [Documenting your schema](documentation.md) for full details.

## Conjoint Tables (Advanced)

Conjoint tables enable dynamic virtualisation of many-to-many relationships without declaring multiple virtual tables by hand. Add `conjoint: true`:

```yaml
AnnotationValue:
    conjoint: true
    id:
        Id: [int]
    columns:
        AnnotationNameId: [AnnotationName]
        Value: [string]
```

In queries, you can refer to a conjoint column with a suffix (e.g., `AnnotationValue.Value@shopping`) to create an ad‑hoc virtual instance. See [Conjoint Tables](conjoint.md) for details.

## Aggregations in Queries (Not in Schema)

FlowerBI does **not** declare aggregations in the YAML schema. Aggregation functions (`count`, `sum`, `average`) are applied at query time on numeric columns. The schema only defines column types (e.g., `decimal`, `int`) that are aggregatable. For example, a query can use `Bug.Id.count()` because `Id` is an integer column.

## Complete Example

```yaml
schema: SalesSchema
tables:
    Customer:
        id:
            Id: [int]
        columns:
            CustomerName: [string]

    Product:
        id:
            Id: [int]
        columns:
            ProductName: [string]
            Price: [decimal]

    Sale:
        id:
            Id: [int]
        columns:
            CustomerId: [Customer]
            ProductId: [Product]
            Quantity: [int]
            SaleDate: [datetime]

    DateOrdered:
        extends: Date
        columns:
            IsHoliday: [bool]
```

This schema defines two physical tables (`Customer`, `Product`, `Sale`) and one virtual table (`DateOrdered`) that extends a base `Date` table.

## Further Reading

- [Virtual Tables](virtual-tables.md)
- [Conjoint Tables](conjoint.md)
- [Documenting your schema](documentation.md)
- [Many-to-Many](many-to-many.md)
- [FlowerBI Overview](flowerbi-overview-what-is-flowerbi.md)
