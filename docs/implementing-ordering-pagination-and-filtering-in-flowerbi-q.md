---
title: Implementing Ordering, Pagination, and Filtering in FlowerBI Queries
status: draft
---

# Implementing Ordering, Pagination, and Filtering in FlowerBI Queries

FlowerBI queries are expressed as JSON objects sent via POST to a single API endpoint. The query structure supports filtering, ordering, and pagination through dedicated properties. This article explains the exact syntax for each, and covers how the schema influences ordering capabilities.

## Query JSON Structure

A full query JSON object can include these top-level keys:

- `select` (required): array of column references (e.g., `["Customer.CustomerName"]`)
- `aggregations` (optional): array of aggregation objects, each with `column` and `function` (`"Count"`, `"Sum"`, `"Average"`, etc.). When present, `select` columns become GROUP BY columns.
- `filters` (optional): array of filter objects (see below).
- `orderBy` (optional): array of sort specifications (see below).
- `offset` (optional): integer, number of rows to skip.
- `limit` (optional): integer, maximum number of rows to return.

Example query selecting customer names and bug counts, filtered by resolved bugs, ordered by customer name ascending, limited to 10 records:

```json
{
    "select": ["Customer.CustomerName"],
    "aggregations": [
        {
            "column": "Bug.Id",
            "function": "Count"
        }
    ],
    "filters": [
        {
            "column": "Workflow.Resolved",
            "operator": "=",
            "value": true
        }
    ],
    "orderBy": [
        {
            "column": "Customer.CustomerName",
            "direction": "asc"
        }
    ],
    "offset": 0,
    "limit": 10
}
```

## Filter Syntax

Filters use the following structure:

```json
{
    "column": "Table.Column",
    "operator": "=",
    "value": <value>
}
```

Operators include `=`, `<>`, `<`, `>`, `<=`, `>=`, `in`, `contains`, etc. (exact set depends on server implementation). For `in`, provide an array of values.

## Order By Syntax

The `orderBy` array contains objects with:
- `column`: fully qualified column name (e.g., `"Customer.CustomerName"`).
- `direction`: `"asc"` or `"desc"` (case-insensitive).

Only columns that appear in the `select` (or in aggregations, if present) can be used for ordering. Attempting to order by a column not present in the query will result in an error at the server level.

## Pagination via Offset and Limit

Pagination is implemented with `offset` (rows to skip) and `limit` (rows to return). The first page typically uses `offset: 0` and `limit: <pageSize>`. Subsequent pages increment offset by pageSize.

## Configuring the Schema to Enable Ordering

Ordering is **not** configured per column in the schema. Instead, the schema defines which tables and columns exist. Any column that is queryable (i.e., defined in the YAML schema and exposed via generated TypeScript/C# constants) can be used in `orderBy` **as long as it is included in the query’s select or aggregation group**.

There are two important restrictions enforced by the FlowerBI engine:

1. **Non-aggregated queries**: If no `aggregations` are specified, `orderBy` may reference any column in the `select` array.
2. **Aggregated queries**: When aggregations are present, `orderBy` must reference either a column in `select` (GROUP BY column) or an aggregation alias. The default alias for an aggregation is the aggregated column name followed by the aggregation function, e.g., `Bug.Id.Count`. However, the exact alias format depends on the client library. The safest approach is to use the same column reference as in the `select` or as the `column` in the aggregation.

The schema YAML does not require any special flags to allow ordering. Simply ensure the column is defined in your YAML, and the generated code will make it available for querying.

## Example with Ordering by an Aggregated Value

To order by the aggregated value (e.g., bug count descending) use the aggregation's default alias. For instance:

```json
{
    "select": ["Customer.CustomerName"],
    "aggregations": [
        {
            "column": "Bug.Id",
            "function": "Count"
        }
    ],
    "orderBy": [
        {
            "column": "Bug.Id",
            "direction": "desc"
        }
    ]
}
```

This generally works because the server implicitly uses the aggregated value for ordering. If your server version requires explicit alias, you may need to check the generated SQL or server documentation.

## Using FlowerBITable with Pagination and Sorting

The `FlowerBITable` component (from `flowerbi-react`) does not handle pagination or sorting internally. You must manage state yourself and re-query with updated `orderBy`, `offset`, and `limit` parameters. For simple cases, consider using a library like `react-table` alongside FlowerBI. The query structure described above is the foundation for implementing any custom pagination or sorting UI.

## Summary

- Order by is specified via an `orderBy` array of `{column, direction}` objects.
- Pagination uses `offset` and `limit`.
- Filters follow a `{column, operator, value}` pattern.
- Schema YAML does not need special configuration for ordering; any queryable column can be ordered as long as it’s part of the query result.
- FlowerBITable does not provide built-in sorting/pagination; you implement it by re-querying with the appropriate parameters.

For more details on query construction, see [the FlowerBI core client documentation](https://earwicker.com/flowerbi/typedoc/flowerbi).