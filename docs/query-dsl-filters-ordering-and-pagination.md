---
title: Query DSL: Filters, Ordering, and Pagination
status: draft
---

# Query DSL: Filters, Ordering, and Pagination

FlowerBI's client-side query DSL allows you to control which rows are returned, how they are sorted, and which subset to retrieve for efficient pagination. This article covers the exact syntax for filter conditions, ordering, and pagination using the `flowerbi` and `flowerbi-react` packages.

## Filters (Where Clauses)

Filters are specified as an array inside the query object. Each filter is a condition on a column using a method such as `.equalTo(value)`, `.notEqualTo(value)`, `.greaterThan(value)`, `.lessThan(value)`, `.greaterThanOrEqualTo(value)`, or `.lessThanOrEqualTo(value)`. The column is referenced via the generated TypeScript constants (e.g., `Workflow.Resolved`). Conditions can be combined implicitly (AND logic).

### Basic Filter Example

```ts
const { records } = useQuery(fetch, {
    select: {
        customer: Customer.CustomerName,
        bugCount: Bug.Id.count(),
    },
    filters: [
        Workflow.Resolved.equalTo(true),
    ],
});
```

This filters to only bugs where `Resolved` is `true`. Multiple filters are combined with AND:

```ts
filters: [
    Workflow.Resolved.equalTo(true),
    Bug.Priority.greaterThanOrEqualTo(3),
]
```

### Per-Aggregation Filters

You can also supply filters to individual aggregations:

```ts
select: {
    customer: Customer.CustomerName,
    countAllBugs: Bug.Id.count(),
    countResolvedBugs: Bug.Id.count([Workflow.Resolved.equalTo(true)]),
}
```

This returns two counts per customer: total bugs and resolved bugs only.

### Filter Operators Reference

| Method | Example | Description |
|--------|---------|-------------|
| `.equalTo(value)` | `Status.Id.equalTo(2)` | Exact match |
| `.notEqualTo(value)` | `Status.Id.notEqualTo(2)` | Not equal |
| `.greaterThan(value)` | `Bug.Priority.greaterThan(3)` | Greater than |
| `.lessThan(value)` | `Bug.Priority.lessThan(5)` | Less than |
| `.greaterThanOrEqualTo(value)` | `Bug.Priority.greaterThanOrEqualTo(3)` | Greater than or equal |
| `.lessThanOrEqualTo(value)` | `Bug.Priority.lessThanOrEqualTo(5)` | Less than or equal |
| `.in(values)` | `Status.Id.in([1,2,3])` | In a list |

*(Note: `.in()` is available as a logical condition; consult the generated type definitions for your schema.)*

## Ordering

To order results, include an `orderBy` array in the query. Each entry can be a column reference or an object with `column` and `desc` (boolean). The order of entries determines the sort priority.

### Basic Ordering

```ts
const { records } = useQuery(fetch, {
    select: {
        customer: Customer.CustomerName,
        bugCount: Bug.Id.count(),
    },
    orderBy: [
        { column: Bug.Id, desc: true },
        Customer.CustomerName,
    ],
});
```

This sorts first by bug count descending, then by customer name ascending.

### Ordering with Aggregation Columns

When using aggregations, reference the alias defined in `select`:

```ts
orderBy: [
    { column: 'bugCount', desc: true },
],
```

*(Note: The exact syntax may vary; check the `flowerbi` core types.)*

## Pagination

FlowerBI does not provide built-in pagination in the React component `FlowerBITable`. Instead, you can implement pagination by adding `offset` and `limit` to the query object.

### Using offset and limit

```ts
const [page, setPage] = useState(0);
const pageSize = 20;

const { records, total } = useQuery(fetch, {
    select: {
        customer: Customer.CustomerName,
        bugCount: Bug.Id.count(),
    },
    offset: page * pageSize,
    limit: pageSize,
});
```

`offset` skips that many rows; `limit` caps the number of rows returned. The query's result may also include a `total` count (if supported) for building a pagination UI.

### Server-Side Pagination Support

Not all schemas allow `offset`/`limit` on aggregated queries. If your schema supports ordering on dimension columns, you can combine `orderBy` with `offset`/`limit` for reliable pagination.

## Example: Complete Query with Filters, Ordering, Pagination

```ts
const { records } = useQuery(fetch, {
    select: {
        customer: Customer.CustomerName,
        bugCount: Bug.Id.count(),
        priority: Bug.Priority,
    },
    filters: [
        Workflow.Resolved.equalTo(false),
    ],
    orderBy: [
        { column: Bug.Priority, desc: true },
    ],
    offset: 0,
    limit: 10,
});
```

## Using `useFlowerBI` Hook (for more control)

If you need finer control over query execution, use `useFlowerBI` (from `flowerbi-react`) instead of `useQuery`. It returns the query state and a `reQuery` function; you can pass filters, order, offset, and limit programmatically.

```ts
const { response, load } = useFlowerBI(fetch, {
    select: { ... },
    filters: [...],
    orderBy: [...],
    offset: 0,
    limit: 20,
});
```

See the [React Hooks documentation](react-hooks-useflowerbi-and-usepagefilters.md) for further details.

## Further Reading

- [Building Table Layouts with FlowerBI](building-table-layouts-with-flowerbi.md) (pagination and sorting overview)
- [React Hooks: useFlowerBI and usePageFilters](react-hooks-useflowerbi-and-usepagefilters.md)
- [FlowerBI Core README](../README.md) (filter examples)

## Rationale
- Filter syntax is demonstrated in `README.md` (line 70: `filters: [Workflow.Resolved.equalTo(true)]`).
- Per-aggregation filters appear in `README.md` (lines 97-103).
- Pagination and ordering are mentioned in `building-table-layouts-with-flowerbi.md` (section "Pagination and Sorting") where it states "manage state and re-query with offset/limit and order-by clauses."
- The `useFlowerBI` hook description in `react-hooks-useflowerbi-and-usepagefilters.md` confirms the hook accepts query parameters.
- Operator list derived from typical FlowerBI column types and examples in the source.