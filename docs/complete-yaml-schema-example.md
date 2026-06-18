---
title: Complete YAML Schema Example
status: draft
---

# Complete YAML Schema Example

This document provides a self-contained FlowerBI YAML schema definition that includes fact tables, dimension tables, primary keys, foreign keys, nullable columns, and documentation. Use it as a template for your own schemas.

## Example: Sales Star Schema

```yaml
schema: SalesSchema
name: SalesDB  # optional physical name override for the schema
topics:
    currency: |
        Monetary columns are in the invoice's original currency unless
        the column name ends in `USD`. Aggregating across currencies is
        almost always a bug.

tables:
    # Dimension table: Date
    Date:
        id:
            Id: [datetime]
        columns:
            CalendarYearNumber: [short]
            FirstDayOfQuarter: [datetime]
            FirstDayOfMonth: [datetime]
            MonthName: [string]

    # Virtual dimension tables via extends (same physical table)
    DateOrdered:
        extends: Date

    DateShipped:
        extends: Date

    # Dimension table: Customer
    Customer:
        id:
            Id: [int]
        columns:
            CustomerName: [string]
            Segment: [string]
            Country: [string]

    # Dimension table: Product
    Product:
        id:
            Id: [int]
        columns:
            ProductName: [string]
            Category: [string]
            UnitPrice: [decimal]

    # Dimension table: Location (nullable foreign key example)
    Location:
        id:
            Id: [int]
        columns:
            City: [string]
            Region: [string?]  # nullable
            Country: [string]

    # Fact table: SalesOrder
    SalesOrder:
        id:
            OrderId: [int]
        columns:
            OrderDate: [DateOrdered]
            ShipDate: [DateShipped?]  # nullable foreign key
            CustomerId: [Customer]
            ProductId: [Product]
            Quantity: [int]
            UnitPrice: [decimal]
            Discount: [decimal?]  # nullable
            SalesAmount:
                type: decimal
                name: TotalAmount  # physical column name
                doc: Revenue after discount in original currency.
                see: [currency]
            LocationId: [Location?]  # nullable foreign key

    # Fact table: Invoice (multiple foreign keys to same dimension)
    Invoice:
        id:
            InvoiceId: [int]
        columns:
            OrderId: [SalesOrder]
            InvoiceDate: [Date]  # uses base Date (not virtualized)
            Amount:
                type: decimal
                doc: Gross amount in original currency.
                see: [currency]
            Paid: [bool?]
```

## Key Points

- **Primary keys** are defined under `id:` for each table. The key must map to a single column (composite keys are not supported).
- **Foreign keys** are declared by using another table's name as the column type. For example, `CustomerId: [Customer]` creates a many-to-one relationship.
- **Nullable columns** (including foreign keys) are indicated by appending `?` to the type, e.g., `[DateShipped?]` or `[decimal?]`.
- **Virtual tables** (e.g., `DateOrdered`, `DateShipped`) allow multiple meanings for the same physical dimension table. They extend the base table and can be used in FK definitions.
- **Documentation** via `doc` and `see` is optional but recommended. The `topics` section defines shared notes.
- Use the `name` property to override the physical table or column name when it differs from the logical name (as shown in `Invoice.SalesAmount`).

This schema is ready to be processed by `FlowerBI.Tools` to generate TypeScript and C# client code. For more advanced patterns, refer to the [Virtual Tables](./virtual-tables.md) and [Conjoint Tables](./conjoint.md) documentation.