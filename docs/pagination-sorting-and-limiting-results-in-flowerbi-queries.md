---
title: Pagination, Sorting, and Limiting Results in FlowerBI Queries
status: draft
---

# Pagination, Sorting, and Limiting Results in FlowerBI Queries

FlowerBI queries are sent as JSON objects to a single POST endpoint. The query structure includes `select`, `filters`, `aggregations`, and optional metadata for controlling the result set size and ordering. This article explains how to add **offset**, **limit**, and **order-by** clauses to a FlowerBI query to implement pagination and sorting.

> **Note:** The `FlowerBITable` component does not handle pagination or sorting automatically. You must manage state (e.g., current page, sort column) in your application and re-query the API with updated `offset`, `limit`, and `orderBy` parameters.

## Query Structure Overview

A basic FlowerBI query looks like this:

```json
{
  "select": ["Customer.CustomerName"],
  "aggregations": [
    {
      "column": "Bug.Id",
      "function": "Count"
    }
  ],
  "filters": []
}
```

The server returns a `QueryResultJson` containing `columns` and `rows`. To paginate or sort, you add extra top-level fields.

## Adding `offset` and `limit`

Include `offset` and `limit` as top-level numeric fields in the query JSON:

```json
{
  "select": ["Customer.CustomerName"],
  "aggregations": [
    {
      "column": "Bug.Id",
      "function": "Count"
    }
  ],
  "filters": [],
  "offset": 0,
  "limit": 20
}
```

- **`offset`**: Number of rows to skip (0-based). Use for page navigation.
- **`limit`**: Maximum number of rows to return. Commonly the page size.

For the next page, increment `offset` by the page size (e.g., `offset: 20`).

## Adding `orderBy` (sorting)

Sorting is specified by the `orderBy` field. Its value is an array of objects, each containing a `column` and an optional `desc` boolean (default `false` for ascending).

```json
{
  "select": ["Customer.CustomerName"],
  "aggregations": [
    {
      "column": "Bug.Id",
      "function": "Count"
    }
  ],
  "filters": [],
  "orderBy": [
    { "column": "Customer.CustomerName", "desc": false },
    { "column": "Bug.Id", "desc": true }
  ]
}
```

- **`column`**: Fully qualified column name as it appears in the schema (e.g., `Customer.CustomerName`). Aggregated columns can also be sorted (e.g., `Bug.Id` when an aggregation is applied).
- **`desc`**: Set to `true` for descending order; omit or set `false` for ascending.

You can combine multiple sort criteria. The order of entries determines precedence.

## Combining Pagination and Sorting

Simply include both `offset`, `limit`, and `orderBy` in the same query:

```json
{
  "select": ["Customer.CustomerName"],
  "aggregations": [
    {
      "column": "Bug.Id",
      "function": "Count"
    }
  ],
  "filters": [],
  "orderBy": [
    { "column": "Customer.CustomerName", "desc": false }
  ],
  "offset": 10,
  "limit": 10
}
```

This returns rows 11–20 sorted alphabetically by customer name.

## Considerations

- **Schema support**: The columns used in `orderBy` must be present in the `select` list or be valid columns from the schema that can be referenced (the engine will add them to the GROUP BY if needed). Refer to your schema definition for available columns.
- **Performance**: Using large offsets can be inefficient. Consider keyset pagination (where supported) by filtering on a column instead of skipping rows.
- **Consistency**: Sorting should be deterministic (include a unique column like ID in `orderBy`) to avoid missing or duplicate rows across pages.

## Example: React Hook Integration

Using the `useFlowerBI` hook from `flowerbi-react`, you can manage pagination state and re-query:

```tsx
function PaginatedTable() {
  const [page, setPage] = useState(0);
  const pageSize = 20;

  const query = {
    select: { customer: Customer.CustomerName, bugCount: Bug.Id.count() },
    filters: [],
    orderBy: [{ column: "Customer.CustomerName", desc: false }],
    offset: page * pageSize,
    limit: pageSize,
  };

  const { records, loading } = useFlowerBI(fetch, query);

  return (
    <div>
      {loading && <p>Loading...</p>}
      <table>
        <thead>
          <tr>
            <th>Customer</th>
            <th>Bug Count</th>
          </tr>
        </thead>
        <tbody>
          {records.map((r) => (
            <tr key={r.customer}>
              <td>{r.customer}</td>
              <td>{r.bugCount}</td>
            </tr>
          ))}
        </tbody>
      </table>
      <button onClick={() => setPage(page - 1)} disabled={page === 0}>
        Previous
      </button>
      <button onClick={() => setPage(page + 1)}>Next</button>
    </div>
  );
}
```

For a more comprehensive table with built-in sorting and pagination, consider using a library like `react-table` alongside FlowerBI to manage state and trigger new queries.

## References

- [Building Table Layouts with FlowerBI](./building-table-layouts-with-flowerbi.md) – section on pagination and sorting.
- [React Hooks: useFlowerBI and usePageFilters](./react-hooks-useflowerbi-and-usepagefilters.md)
- [FlowerBI Overview](./flowerbi-overview-what-is-flowerbi.md)