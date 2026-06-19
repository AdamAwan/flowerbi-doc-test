---
title: Time‑Series Aggregations: Rolling Sums, Moving Averages, and Date Bucketing
status: draft
---

# Time‑Series Aggregations in FlowerBI

FlowerBI does not provide native window functions (e.g., `ROW_NUMBER()`, `SUM() OVER`) or built‑in moving‑average operators. However, you can often achieve the same results by combining core features: **filtered aggregations**, **multiple queries**, and **date‑bucketed grouping**. This article walks through practical patterns for rolling sums, moving averages, and date bucketing.

## Date Bucketing (Grouping by Time Periods)

To aggregate over time periods (days, months, quarters, etc.), you include a date column in your query’s `select` – FlowerBI will automatically `GROUP BY` all non‑aggregated columns. For example, to count bugs per day:

```ts
const query = {
  select: {
    day: Bug.ReportedDate,
    bugCount: Bug.Id.count(),
  },
};
```

If your date dimension table contains pre‑computed buckets (e.g., `FirstDayOfMonth`, `CalendarYearNumber`), include those columns instead of the raw date:

```ts
const query = {
  select: {
    month: Date.FirstDayOfMonth,
    bugCount: Bug.Id.count(),
  },
};
```

This relies on your schema having a date dimension with columns like `FirstDayOfMonth` (see [virtual tables](./virtual-tables.md) for how to safely use the same physical date table for different roles).

## Rolling Sum Over a Fixed Window

To compute a rolling sum (e.g., total sales over the last 30 days for each day), you need to issue **multiple queries** – one per day in the window – or one query per day with a filter that covers the trailing window. Because each query runs separately, you can then merge results client‑side.

Example in TypeScript/React:

```ts
import { useQuery } from "flowerbi-react";

function RollingSum() {
  const endDate = new Date();
  const windowDays = 30;

  // Generate one query per day
  const queries = Array.from({ length: windowDays }, (_, i) => {
    const date = new Date(endDate);
    date.setDate(date.getDate() - i);
    const nextDay = new Date(date);
    nextDay.setDate(nextDay.getDate() + 1);

    return useQuery(fetch, {
      select: {
        day: Date.Id,
        sales: Invoice.Amount.sum({
          filters: [
            Invoice.InvoiceDate.greaterThanOrEqual(date),
            Invoice.InvoiceDate.lessThan(nextDay),
          ],
        }),
      },
      filters: [
        Invoice.InvoiceDate.greaterThanOrEqual(
          new Date(endDate.getTime() - windowDays * 86400000)
        ),
      ],
    });
  });

  // ... combine results into a rolling sum
}
```

> *Note:* Each query runs independently. For very large windows you may prefer a single query that groups by day, then compute the rolling sum on the client using a sliding window over the returned records. This is more efficient and avoids many round trips.

## Moving Average via Filtered Aggregations

A moving average (e.g., 7‑day average of daily signups) can be approximated with **filtered aggregations** if you pre‑compute a row for each date and then apply a filter that selects the desired window. However, because filtered aggregations can only apply different filters to each aggregated column – not to the grouping columns – you typically need one aggregation per moving‑average point.

Simpler approach: retrieve daily totals (one query), then compute the moving average on the client. Example:

```ts
const query = {
  select: {
    day: Date.Id,
    dailyCount: Bug.Id.count(),
  },
  filters: [
    Bug.ReportedDate.greaterThanOrEqual(startDate),
    Bug.ReportedDate.lessThanOrEqual(endDate),
  ],
};

// In a React component, after fetching records:
const movingAverage = records.map((rec, idx, arr) => {
  const window = arr.slice(Math.max(0, idx - 3), idx + 1); // 4‑day window
  return {
    day: rec.day,
    avg: window.reduce((s, r) => s + r.dailyCount, 0) / window.length,
  };
});
```

If you must avoid client‑side computation, you can issue one query per window point with a filtered aggregation that sums over a date range. Combine the results in a single result set using a union or by merging client‑side. This becomes expensive for long series.

## Multiple Aggregations Over Different Windows

You can compare two different window lengths in one query by using **filtered aggregations**. For example, to see the count of bugs created in the last 7 days and the last 30 days side by side:

```ts
const query = {
  select: {
    customer: Customer.CustomerName,
    last7Days: Bug.Id.count([
      Bug.ReportedDate.greaterThanOrEqual(daysAgo(7)),
    ]),
    last30Days: Bug.Id.count([
      Bug.ReportedDate.greaterThanOrEqual(daysAgo(30)),
    ]),
  },
  filters: [
    Bug.ReportedDate.greaterThanOrEqual(daysAgo(30)), // ensures only relevant rows
  ],
};
```

This works because each aggregation can have its own filter. The result includes one row per customer with both counts.

## Important Considerations

- **Performance:** For large datasets, fetching raw daily totals and computing rolling aggregates client‑side is often faster than many small queries. Use indexing on date columns and limit the overall time range.
- **Time Zones:** Date values should be stored in a consistent time zone (e.g., UTC). If your date dimension includes time‑zone aware columns, align your filter values accordingly.
- **Virtual Tables:** If your schema uses the same physical date table for multiple date roles (e.g., `RequestedDate` and `CompletedDate`), leverage [virtual tables](./virtual-tables.md) to avoid accidental cross‑filtering.
- **Full Joins:** When combining multiple aggregations with different filters, the query may generate separate CTEs. If you use `fullJoins: true` (see [full-joins documentation](./full-joins.md)), the results are symmetrical, which can simplify client‑side merging.

## Summary

FlowerBI does not have native window functions, but you can implement rolling sums, moving averages, and date bucketing by:

- Grouping by date columns to get per‑period totals.
- Using filtered aggregations to compute values over different windows in one query.
- Issuing multiple queries per window point (for true rolling/windowed results).
- Performing client‑side window computations on a saved result set for efficiency.

These patterns let you build time‑series visualisations with chart.js or other libraries (see [flowerbi-react-chartjs](https://earwicker.com/flowerbi/typedoc/flowerbi-react-chartjs)).