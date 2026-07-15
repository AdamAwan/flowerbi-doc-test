---
title: Full Joins for Multi-Aggregation Queries
status: draft
---

# Full Joins for Multi-Aggregation Queries

When a query contains multiple aggregations, the way non-aggregated columns are matched between the aggregations can produce asymmetrical results. The `fullJoins` flag addresses this by generating symmetrical `FULL OUTER JOIN` behaviour.

## The asymmetry problem

Each aggregation in a query may carry its own set of filters, meaning different aggregations can operate on different subsets of the underlying records. The non-aggregated (grouping) columns are shared across all aggregations. When the results are combined, rows present in one aggregation's output may be absent from another's.

By default, the first aggregation listed in the query controls which rows appear in the final output. Rows that exist only in a later aggregation are discarded. This is because the default join type used to combine the aggregation results is `LEFT JOIN` — only the first aggregation's row set drives the output. The order of aggregations therefore matters: a row that appears in a later aggregation but not in the first is silently dropped.

For example, if the first aggregation produces rows for "green / chili" and "purple / chocolate" while the second aggregation produces rows for "purple / chocolate" and "orange / chili", the combined result includes "purple / chocolate" (present in both) and "green / chili" (first only), but "orange / chili" (second only) is lost. The output is asymmetrical.

## The `fullJoins` flag

To produce symmetrical results, the query can set `fullJoins: true`. This changes the join type used in the generated SQL from `LEFT JOIN` to `FULL OUTER JOIN`. A full outer join preserves rows from both sides, so a row present only in a later aggregation is retained in the output with null values for the other aggregation's columns.

The `fullJoins` flag is specified in the top-level query JSON:

```json
{
  "select": ["Vendor.VendorName"],
  "aggregations": [
    { "column": "Invoice.Amount", "function": "Sum" }
  ],
  "fullJoins": true
}
```

On the server, the `QueryJson` model includes the nullable `FullJoins` property. The query object reads it as `json.FullJoins ?? false`, defaulting to `false` to preserve backward compatibility with existing queries.

## SQL changes with full joins

When `fullJoins` is `true`, the generated SQL uses `full join` instead of the default `join` (which is an inner join). This affects all table joins in the query, not just a final combination step.

For non-aggregated columns that may now come from either side of a full outer join, the generated SQL must handle nulls. The documented approach wraps the non-aggregated column references with `COALESCE` so that a value is picked from whichever side provides it. For example, when two aggregation contexts produce separate aliases for the same set of non-aggregated columns:

```sql
select coalesce(a0.Select0, a1.Select0) Select0,
       coalesce(a0.Select1, a1.Select1) Select1,
       a0.Value0 Value0,
       a1.Value0 Value1
```

This ensures that a row which exists only in the second aggregation still contributes its grouping column values to the output.

## Concrete example: the Vendor test schema

The test data includes a vendor named "Acme Ltd" (Id = 1) in the `Supplier` table. This vendor has no invoices in the `Invoice` table. With a standard inner join between `Supplier` and `Invoice`, Acme Ltd is excluded from results because the join match fails.

With a query that selects `Vendor.VendorName` and sums `Invoice.Amount`, setting `fullJoins: true` generates a `FULL OUTER JOIN` between `Supplier` and `Invoice`. This causes Acme Ltd to appear in the results with a null aggregated value:

| Vendor.VendorName    | Sum of Invoice.Amount |
|----------------------|----------------------|
| United Cheese        | 406.84               |
| Handbags-a-Plenty    | 252.48               |
| …                   | …                    |
| Stationary Stationery| 28.12                |
| **Acme Ltd**         | **null**             |

This demonstrates that the full join preserves rows that exist on one side of a relationship but not the other.

## Effect on single-aggregation queries

The `fullJoins` flag applies to all table joins within the query. Even with a single aggregation, setting `fullJoins: true` changes the join type. However, the feature is primarily relevant for multi-aggregation queries where the asymmetry between differently-filtered aggregations can lose data. For queries with only one aggregation, there is no asymmetry to resolve between aggregations, though the join type change may still affect which rows appear in the output.

## Recommendations

It is recommended that `fullJoins` be enabled on any query that has multiple aggregations. The default of `false` exists solely to avoid changing the results of existing queries when upgrading an application. A future major version of FlowerBI is likely to make full joins the default.

## No effect on queries without aggregations

For queries that select only plain columns (no aggregations), the `fullJoins` flag has no meaningful effect because there is no asymmetry between aggregation outputs to resolve.