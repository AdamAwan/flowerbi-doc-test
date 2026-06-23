---
title: Date Bucketing and Time-Series Aggregation in FlowerBI
status: draft
---

# Date Bucketing and Time-Series Aggregation in FlowerBI

FlowerBI does not include built-in date-part extraction functions in its query language. Instead, the recommended way to group or bucket dates (e.g., by year, month, or day) is to use a dedicated **date dimension table** that pre‑computes the desired granularities. This approach is consistent with star‑schema best practices and works seamlessly with FlowerBI’s grouping, aggregation, and filtering mechanisms.

## Using a Date Dimension Table

Define a `Date` table in your YAML schema with columns for each granularity you need:

```yaml
Date:
    id:
        Id: [datetime]
    columns:
        CalendarYearNumber: [short]
        FirstDayOfQuarter: [datetime]
        FirstDayOfMonth: [datetime]
        FirstDayOfWeek: [datetime]
        DayOfMonth: [short]
        MonthName: [string]
        # ... other useful date parts
```

This physical table should contain one row per calendar date, with pre‑filled values for each column. Your fact tables then reference this table via foreign keys:

```yaml
WorkItem:
    id:
        Id: [int]
    columns:
        RequestedDate: [Date]
        CompletedDate: [Date]
```

### Querying by Date Bucket

To get a count of work items per month, use the `FirstDayOfMonth` column in the `select`:

```ts
const query = {
    select: {
        monthStart: Date.FirstDayOfMonth,
        count: WorkItem.Id.count()
    },
    filters: []
};
```

The result will contain one row per distinct first day of the month, with the number of work items for that month. The same pattern works for any date‑part column: `CalendarYearNumber` for yearly aggregation, `FirstDayOfWeek` for weekly, etc.

## Virtual Tables for Multiple Date Roles

If your schema uses the same date table for different date roles (e.g., `RequestedDate` and `CompletedDate`), you can create **virtual tables** to avoid cross‑contamination of filters and to allow independent bucketing:

```yaml
DateRequested:
    extends: Date

DateCompleted:
    extends: Date

WorkItem:
    id:
        Id: [int]
    columns:
        RequestedDate: [DateRequested]
        CompletedDate: [DateCompleted]
```

Now you can filter or group by `DateRequested.FirstDayOfMonth` without affecting any other date role. Virtual tables are documented in the [Virtual Tables](virtual-tables.md) guide.

## Time‑Series Aggregation with Filters

FlowerBI’s [Filtered Aggregations](filtered-aggregations.md) let you compute different metrics for the same date bucket. For example, to see total work items and completed work items side by side per month:

```ts
const query = {
    select: {
        month: DateRequested.FirstDayOfMonth,
        total: WorkItem.Id.count(),
        completed: WorkItem.Id.count([WorkItem.CompletedDate.notNull()])
    },
    filters: []
};
```

## The `flowerbi‑dates` Package (Experimental)

The client‑side package [`flowerbi-dates`](https://github.com/danielearwicker/flowerbi/tree/master/client/packages/flowerbi-dates) (included in the FlowerBI repository) provides utility functions for working with dates in queries. While its primary purpose is to supply core query helpers, you can use it to define computed date‑part columns in your client code when you prefer not to rely on the SQL for truncation. For example, you could create a `year(column)` function that maps to a SQL expression. However, using a date dimension table remains the most straightforward and performant approach for most applications.

## Best Practices

- **Prefer a dimension table** – It avoids SQL dialect differences and keeps your schema explicit.
- **Use virtual tables** when the same physical date table serves multiple roles.
- **Keep granularities as physical columns** in the dimension table for simplicity.
- **Always group by discrete columns** (e.g., `FirstDayOfMonth`) rather than trying to truncate in SQL, because FlowerBI’s grouping relies on the values of selected columns.

By following these patterns, you can easily build time‑series charts and date‑based reports with FlowerBI.
