---
title: crossJoin Operator and timeTravel() Aggregation Function
status: draft
---

# crossJoin Operator

`crossJoin` is a built-in operator that generates a Cartesian product of the referenced tables. Its exact syntax is not yet finalized in this draft.

**Proposed syntax:**

```ts
// Example usage within a query
const query = {
    select: {
        product: Product.Name,
        region: Region.Name,
        sales: Sale.Amount.sum()
    },
    crossJoin: [Product, Region], // or similar property
    filters: []
};
```

*Note: The above is a placeholder. The correct property name and placement (e.g., inside `select` or as a top-level key) will be documented once confirmed.*

# timeTravel() Aggregation Function

`timeTravel()` is an aggregation function that allows computing values as of a specific point in time. It accepts the following optional parameters:

| Parameter | Type   | Description |
|-----------|--------|-------------|
| `asOf`    | Date   | The point in time to travel to. If omitted, defaults to the current time. |
| `mode`    | string | How to handle overlapping time periods (e.g., `"latest"`, `"sum"`). Default is `"latest"`. |
| `default` | number | Value to return when no data is found for the given time. Default is `0`. |

**Example:**

```ts
select: {
    count: Bug.Id.count(),
    countAsOfLastWeek: Bug.Id.timeTravel({
        asOf: new Date('2025-03-01'),
        mode: 'latest',
        default: 0
    })
}
```

*Note: Parameters and their names are subject to change. This draft will be updated as the implementation stabilizes.*