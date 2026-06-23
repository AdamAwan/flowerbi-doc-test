---
title: Date Bucketing and Time-Series Aggregation
status: draft
---

# Date Bucketing and Time-Series Aggregation

FlowerBI doesn't have native date-part functions. Use a date dimension table for bucketing; implement rolling sums/moving averages via client-side computation or multiple queries.

## Bucketing with Date Dimension

Include pre-computed columns like `FirstDayOfMonth` in the `select` to group by time period.

```ts
const query = {
  select: { month: Date.FirstDayOfMonth, count: Bug.Id.count() }
};
```

## Time-Series Patterns

- **Rolling sum**: Issue multiple queries or fetch daily totals and compute on client.
- **Moving average**: Filtered aggregations can approximate; simpler to compute client-side.

## Virtual Tables for Multiple Date Roles

Use `extends` to create `DateRequested`, `DateCompleted` for independent filtering.

## Example: Client-side moving average

```ts
const movingAvg = records.map((r, i, arr) => {
  const window = arr.slice(Math.max(0, i - 3), i + 1);
  return { day: r.day, avg: window.reduce((s, e) => s + e.count, 0) / window.length };
});
```

*Consolidated from: date-bucketing-and-time-series-aggregation-in-flowerbi.md, time-series-aggregations-rolling-sums-moving-averages-and-da.md*