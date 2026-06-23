---
title: Complete YAML Schema Example
status: draft
---

# Complete YAML Schema Example

This document provides a self-contained FlowerBI YAML schema definition that includes fact tables, dimension tables, primary keys, foreign keys, nullable columns, and documentation. Use it as a template for your own schemas.

```yaml
schema: ExampleSchema

topics:
  currency: |
    Monetary amounts are stored in the invoice's original currency.
    Use the `Currency` table to convert or display.
  status: |
    Invoice status values: `Pending`, `Paid`, `Cancelled`.

tables:
  # ---- Dimension Tables ----
  
  Customer:
    doc: One row per customer, used for grouping and filtering.
    see: [Invoice]
    id:
      CustomerId: [int]
    columns:
      CustomerName: [string]
      Email: [string?]  # nullable column
  
  Date:
    id:
      DateId: [datetime]
    columns:
      CalendarYear: [short]
      MonthName: [string]
      DayOfMonth: [short]
  
  DateReported:
    extends: Date
    doc: Virtual table for the reported date in WorkItem.
  
  DateResolved:
    extends: Date
    doc: Virtual table for the resolved date in WorkItem.
  
  Currency:
    id:
      CurrencyId: [int]
    columns:
      Code: [string]
      Name: [string]
  
  # ---- Fact Tables ----
  
  Invoice:
    doc: One row per invoice.
    id:
      InvoiceId: [int]
    columns:
      CustomerId: [Customer]            # foreign key to Customer
      InvoiceDate: [Date]               # foreign key to Date
      Amount: [decimal]
      CurrencyId: [Currency]            # foreign key to Currency
      Status: [string]
    see: [currency, status, Customer, Date, Currency]
  
  WorkItem:
    doc: One row per work item.
    id:
      WorkItemId: [int]
    columns:
      RequestedDate: [DateReported]     # foreign key to virtual DateReported
      CompletedDate: [DateResolved]     # foreign key to virtual DateResolved
      Description: [string]
      AssignedTo: [string?]
    see: [DateReported, DateResolved]
  
  # ---- Associative Table (Many-to-Many) ----
  
  InvoiceTag:
    columns:
      InvoiceId: [Invoice]
      TagId: [Tag]
    associative: [InvoiceId, TagId]
    doc: Links invoices to tags.
  
  Tag:
    id:
      TagId: [int]
    columns:
      Name: [string]
```

## Explanation

- **Schema name**: `ExampleSchema` (optional, can be overridden via `name`).
- **Topics**: Shared documentation that can be referenced with `see`.
- **Tables**: Each table has an `id` (primary key) and `columns`. Foreign keys are declared by using the referenced table name as the type.
- **Nullable columns**: Append `?` to the type (e.g., `[string?]`).
- **Virtual tables**: Use `extends` to create logical copies (e.g., `DateReported` extends `Date`). The physical table name remains `Date` unless overridden.
- **Associative tables**: Declare with `associative` to prevent removal from join graphs.
- **Documentation**: Add `doc` and `see` for tables, columns, and topics. This is surfaced in generated code.

This example covers most common patterns. Adjust table and column names to match your database.
