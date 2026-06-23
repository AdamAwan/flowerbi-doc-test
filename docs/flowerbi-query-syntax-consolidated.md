---
title: FlowerBI Query Syntax: Filters, Ordering, and Pagination
status: draft
---

# FlowerBI Query Syntax: Filters, Ordering, and Pagination

FlowerBI queries are JSON objects sent via POST. This guide details the syntax for filters, ordering, pagination, and per-aggregation filters.

## Query Structure

```json
{
  "select": ["Customer.CustomerName"],
  "aggregations": [
    { "column": "Bug.Id", "function": "Count" }
  ],
  "filters": [],
  "orderBy": [],
  "offset": 0,
  "limit": 20
}
```

## Filters

Each filter: `{ "column": "Table.Column", "operator": "=", "value": ... }`

Operators: `=`, `!=`, `<`, `>`, `<=`, `>=`, `like`, `in`, `is null`, `is not null`.

Multiple filters are ANDed.

### Per-Aggregation Filters

```ts
countResolved: Bug.Id.count([Workflow.Resolved.equalTo(true)])
```

## Ordering

`orderBy` array with `{ column, direction }` (direction defaults to `asc`). Example:

```json
"orderBy": [{ "column": "Bug.Id", "direction": "desc" }]
```

Columns used in ordering must be in the select list or aggregations.

## Pagination

`offset` (rows to skip) and `limit` (max rows). Always pair with `orderBy` for deterministic pagination.

## Example: Complete Query

```json
{
  "select": ["Customer.CustomerName"],
  "aggregations": [{ "column": "Bug.Id", "function": "Count" }],
  "filters": [{ "column": "Workflow.Resolved", "operator": "=", "value": true }],
  "orderBy": [{ "column": "Customer.CustomerName", "direction": "asc" }],
  "offset": 0,
  "limit": 10
}
```

*Consolidated from: flowerbi-query-syntax-filters-ordering-and-pagination.md, implementing-ordering-pagination-and-filtering-in-flowerbi-q.md, ordering-pagination-and-sorting-in-flowerbi-queries.md, pagination-and-sorting-in-flowerbi-queries.md, pagination-sorting-and-limiting-results-in-flowerbi-queries.md, query-dsl-filters-ordering-and-pagination.md*