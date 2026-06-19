---
title: FlowerBI Query Syntax: Filters, Ordering, and Pagination
status: draft
---

# FlowerBI Query Syntax: Filters, Ordering, and Pagination

FlowerBI queries are serialised as JSON objects sent to a single POST endpoint.
This document describes how to attach filter conditions (where clauses),
specify sort order, and add pagination (offset/limit) to your queries.

## Filters (Where Clauses)

The query object accepts an optional `filters` array. Each filter is an object
with three required keys:

| Key        | Type   | Description                                  |
|------------|--------|----------------------------------------------|
| `column`   | string | Fully qualified column name (e.g. `"Workflow.Resolved"`) |
| `operator` | string | Comparison operator (`"="`, `"!="`, `"<"`, `">"`, `"<="`, `">="`, `"like"`, etc.) |
| `value`    | any    | The value to compare against (string, number, boolean, or null) |

**Example** – a filter that includes only resolved bugs:

```json
{
    "filters": [
        {
            "column": "Workflow.Resolved",
            "operator": "=",
            "value": true
        }
    ]
}
```

In TypeScript, using the generated constants, the same filter can be written
more concisely:

```typescript
const { records } = useQuery(fetch, {
    select: {
        customer: Customer.CustomerName,
        bugCount: Bug.Id.count(),
    },
    filters: [
        Workflow.Resolved.equalTo(true)
    ],
});
```

See [`flowerbi-react` hooks documentation](./react-hooks-useflowerbi-and-usepagefilters.md)
for details on the `useQuery` hook.

### Supported Operators

The engine supports the usual SQL comparison operators. The list includes but
is not limited to:

- `=` (equal)
- `!=` (not equal)
- `<` / `>` / `<=` / `>=`
- `like`
- `in` (value is an array)
- `is null` / `is not null` (value is `null`)

### Combining Multiple Filters

Multiple filters are combined with AND. For OR logic you need to craft a
single filter with an `in` operator or use multiple queries.

```json
{
    "filters": [
        { "column": "Workflow.Resolved", "operator": "=", "value": true },
        { "column": "Bug.Severity", "operator": ">=", "value": 3 }
    ]
}
```

### Aggregation-Level Filters

You can also attach different filters to each aggregation column. This is done
by passing an array of filters as the second argument to the aggregation
method in TypeScript:

```typescript
const { records } = useQuery(fetch, {
    select: {
        customer: Customer.CustomerName,
        countAllBugs: Bug.Id.count(),
        countResolvedBugs: Bug.Id.count([Workflow.Resolved.equalTo(true)])
    },
});
```

The JSON equivalent uses nested `"filters"` inside each aggregation:

```json
{
    "select": ["Customer.CustomerName"],
    "aggregations": [
        { "column": "Bug.Id", "function": "Count", "filters": [] },
        { "column": "Bug.Id", "function": "Count", "filters": [
            {"column": "Workflow.Resolved", "operator": "=", "value": true}
        ]}
    ]
}
```

## Ordering (Sort)

Post-processing of returned records is the recommended approach for simple
ordering because the query engine does not expose a standard `orderBy` field.
The `FlowerBITable` component leaves sorting to external libraries or custom
state management (see [Pagination and Sorting](./building-table-layouts-with-flowerbi.md#pagination-and-sorting)).

If your schema has appropriate indexes, you **can** add an `orderBy` key to
the query JSON. However, this is an optional, schema-dependent feature; its
behaviour may vary across versions or database back-ends.

```json
{
    "select": ["Customer.CustomerName"],
    "aggregations": [
        { "column": "Bug.Id", "function": "Count" }
    ],
    "orderBy": [
        { "column": "Customer.CustomerName", "direction": "asc" }
    ]
}
```

If your version does not support `orderBy`, sort the array of records on the
client side after receiving the response.

## Pagination (Offset / Limit)

Pagination is achieved by adding two optional numeric fields to the query
object:

| Key      | Type   | Description                          |
|----------|--------|--------------------------------------|
| `offset` | number | Number of rows to skip (default 0)   |
| `limit`  | number | Maximum number of rows to return     |

```json
{
    "select": ["Customer.CustomerName"],
    "aggregations": [
        { "column": "Bug.Id", "function": "Count" }
    ],
    "offset": 10,
    "limit": 20
}
```

This returns rows 11 through 30 (if available). As with ordering, the engine
uses `ROW_NUMBER()` or equivalent under the hood, so the order of rows without
`orderBy` is not guaranteed. For deterministic pagination, pair `offset`/`limit`
with an `orderBy` clause.

## Complete Example

Putting it all together – a query that returns the top 5 customers by bug count,
filtering only resolved bugs, ordered by bug count descending:

```json
{
    "select": ["Customer.CustomerName"],
    "aggregations": [
        { "column": "Bug.Id", "function": "Count" }
    ],
    "filters": [
        { "column": "Workflow.Resolved", "operator": "=", "value": true }
    ],
    "orderBy": [
        { "column": "Bug.Id", "function": "Count", "direction": "desc" }
    ],
    "offset": 0,
    "limit": 5
}
```

## References

- [FlowerBI Overview](./flowerbi-overview-what-is-flowerbi.md)
- [Building Table Layouts with FlowerBI](./building-table-layouts-with-flowerbi.md#pagination-and-sorting)
- [React Hooks: useFlowerBI and usePageFilters](./react-hooks-useflowerbi-and-usepagefilters.md)
- [Main README (source examples)](https://github.com/danielearwicker/flowerbi/blob/master/README.md)