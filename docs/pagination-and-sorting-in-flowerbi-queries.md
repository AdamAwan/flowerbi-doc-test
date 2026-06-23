---
title: Pagination and Sorting
status: draft
---

# Pagination and Sorting in FlowerBI Queries

FlowerBI supports pagination and sorting at the query level, enabling you to control which rows are returned and in what order. This is useful for building interactive tables or lists that limit results to a manageable page size, and let users sort by a specific column.

## Pagination: Offset and Limit

To request a subset of results, include `offset` and `limit` properties in your `QueryJson` object:

```json
{
  "select": ["Customer.CustomerName", "Bug.Id.count()"],
  "filters": [],
  "offset": 10,
  "limit": 25
}
```

- **`offset`** – The number of rows to skip (0‑based). Default: `0`.
- **`limit`** – The maximum number of rows to return. If omitted, all rows matching the query are returned.

When you use offset/limit, the server applies them to the final result set **after** grouping and aggregation. This means the offset and limit control the number of **result rows** (one per group), not raw database rows.

## Sorting: Order‑By Clauses

To order results, add an `orderBy` array to your query. Each element specifies a column and a direction (`asc` or `desc`):

```json
{
  "select": ["Customer.CustomerName", "Bug.Id.count()"],
  "filters": [],
  "orderBy": [
    {"column": "Bug.Id", "direction": "desc"},
    {"column": "Customer.CustomerName", "direction": "asc"}
  ],
  "limit": 10
}
```

- `column` – The full column path (e.g., `"Customer.CustomerName"` or `"Bug.Id"`).
- `direction` – Either `"asc"` (ascending) or `"desc"` (descending).

You can sort by any column that appears in the `select` list or that is part of the grouping columns. The order‑by clauses are applied **before** the offset/limit, so pagination works on the sorted result set.

## Client‑Side Usage with React

When using FlowerBI from a React application via `useFlowerBI` or `useQuery`, you manage pagination state and pass updated queries to the hook. For example:

```tsx
import { useQuery } from 'flowerbi-react';

const [page, setPage] = useState(0);
const pageSize = 25;

const query = {
  select: { customer: Customer.CustomerName, bugCount: Bug.Id.count() },
  filters: [],
  offset: page * pageSize,
  limit: pageSize,
  orderBy: [{ column: 'Bug.Id', direction: 'desc' }]
};

const { records } = useQuery(fetch, query);
```

Whenever `page` changes (e.g., via a “Next” button), the query object changes and `useQuery` re‑fetches data from the server.

## Important Notes

- Not all schemas support `orderBy` on every column. The server enforces that the column exists in the schema and is allowed for ordering.
- `offset` and `limit` have no effect if the underlying database engine or API does not support them (most modern SQL databases do).
- For complex queries with multiple aggregations, offset/limit apply to the outer result set after joins; see [Full Joins](./full-joins.md) for details on the join behaviour.

## Example: Simple Paginated Table

Below is a complete example of a paginated query using the `flowerbi-react` hooks (see [React Hooks documentation](./react-hooks-useflowerbi-and-usepagefilters.md) for more on `useFlowerBI`):

```tsx
const pageSize = 10;
const [currentPage, setCurrentPage] = useState(0);

const { records, loading } = useFlowerBI<MyRecord>(
  fetch,
  {
    select: {
      name: Customer.CustomerName,
      totalBugs: Bug.Id.count()
    },
    offset: currentPage * pageSize,
    limit: pageSize,
    orderBy: [{ column: 'Bug.Id', direction: 'desc' }]
  }
);

return (
  <div>
    <table>
      <thead>
        <tr><th>Customer</th><th>Bugs</th></tr>
      </thead>
      <tbody>
        {records.map(r => (
          <tr key={r.name}><td>{r.name}</td><td>{r.totalBugs}</td></tr>
        ))}
      </tbody>
    </table>
    <button onClick={() => setCurrentPage(p => Math.max(0, p - 1))}>Previous</button>
    <span>Page {currentPage + 1}</span>
    <button onClick={() => setCurrentPage(p => p + 1)}>Next</button>
  </div>
);
```

## References

- [Building Table Layouts with FlowerBI](./building-table-layouts-with-flowerbi.md) – discusses `FlowerBITable` and integration with libraries like `react-table`.
- [React Hooks: useFlowerBI and usePageFilters](./react-hooks-useflowerbi-and-usepagefilters.md) – details on using the hooks.
- [FlowerBI Query JSON Structure (README)](https://github.com/danielearwicker/flowerbi#flexible-and-creative-querying-at-the-client) – shows the core query syntax.
- [Full Joins](./full-joins.md) – explains multi‑aggregation join behaviour that can interact with pagination.