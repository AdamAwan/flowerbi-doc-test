---
title: Filtered Aggregations
status: draft
---

# Filtered Aggregations

FlowerBI allows you to apply different filters to each aggregation within a single query, enabling conditional sums, counts, averages, and more. This is useful when you want to compare metrics under different conditions side by side—for example, total orders versus orders from VIP customers, or total bugs versus resolved bugs—all in one query result.

## Syntax

To apply a per-aggregation filter, pass an array of filter conditions as the last argument to the aggregation method (e.g., `.count()`, `.sum()`):

```ts
countResolvedBugs: Bug.Id.count([Workflow.Resolved.equalTo(true)])
```

Each aggregation can have its own filters, or no filters at all. The filters are specified as an array of `Filter` objects, exactly like the top-level `filters` array in a query.

## Example

Consider a bug‑tracking schema with tables `Customer`, `Bug`, and `Workflow`. You want to show, per customer, the total number of bugs and the number of resolved bugs:

```ts
const { records } = useQuery(fetch, {
  select: {
    customer: Customer.CustomerName,
    countAllBugs: Bug.Id.count(),
    countResolvedBugs: Bug.Id.count([Workflow.Resolved.equalTo(true)]),
  },
  // Optionally add a global filter (applied to all aggregations)
  filters: [],
});
```

The result contains records like:

```ts
{
  customer: "Acme Corp",
  countAllBugs: 42,
  countResolvedBugs: 15
}
```

## How It Works

When a query contains multiple aggregations with different filters, FlowerBI generates a separate Common Table Expression (CTE) for each aggregation. These CTEs are then joined on the non‑aggregated (group‑by) columns. By default, a `LEFT JOIN` is used, which means rows that only appear in a later aggregation are omitted from the result. To get a symmetric join (recommended for multiple aggregations), set `fullJoins: true` in the query object:

```ts
const { records } = useQuery(fetch, {
  select: { ... },
  fullJoins: true, // ensures all combinations are preserved
});
```

See [Full Joins](./full-joins.md) for a detailed explanation of this behaviour.

## Supported Aggregation Functions

The following aggregation functions accept per‑aggregation filters:

- `count()`
- `sum()`
- `average()`
- `minimum()`
- `maximum()`

## Notes

- Per‑aggregation filters are combined with global top‑level filters: a row must satisfy both the global filter **and** the aggregation’s own filter to be included in that aggregation’s result.
- Using filtered aggregations with many distinct filter sets may affect query performance, as each filter set is evaluated in its own CTE.

For further details, refer to the [FlowerBI Query API](https://earwicker.com/flowerbi/typedoc/flowerbi) reference.