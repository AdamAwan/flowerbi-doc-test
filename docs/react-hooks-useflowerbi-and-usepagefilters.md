---
title: React Hooks: useFlowerBI and usePageFilters
status: draft
---

# React Hooks: useFlowerBI and usePageFilters

The `flowerbi-react` package provides two main hooks for integrating FlowerBI queries into React applications.

## `useFlowerBI`

This hook manages the state of a query and the data it produces.

### Signature

```ts
function useFlowerBI(
  fetch: QueryFetch,
  query: QueryDefinition,
  options?: UseFlowerBIOptions
): UseFlowerBIResult;
```

- `fetch`: A function that accepts a `QueryJson` object and returns a `Promise<QueryResultJson>`. This is typically a wrapper around `window.fetch` that POSTs the query JSON to your server endpoint.
- `query`: A `QueryDefinition` object describing the select, aggregations, filters, etc. (as defined in the `flowerbi` core package).
- `options` (optional): An object that may include:
  - `onError`: callback for handling errors.
  - `debounceMs`: debounce delay for re-fetching when query changes.

### Return value (`UseFlowerBIResult`)

```ts
interface UseFlowerBIResult {
  records: readonly Record[];
  loading: boolean;
  error: Error | undefined;
  refresh: () => void;
}
```

- `records`: The array of result rows. Each row is a strongly-typed object inferred from the query's `select` and `aggregations`.
- `loading`: `true` while a fetch is in progress.
- `error`: Any error that occurred during the last fetch.
- `refresh`: Manually triggers a re-fetch with the same query.

### Example

```tsx
import { useFlowerBI } from 'flowerbi-react';
import { Customer, Invoice } from './generated-schema';

const fetchData = async (queryJson) => {
  const response = await fetch('/api/query', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(queryJson),
  });
  return response.json();
};

const MyChart = () => {
  const { records, loading, error } = useFlowerBI(fetchData, {
    select: {
      customerName: Customer.CustomerName,
      totalAmount: Invoice.Amount.sum(),
    },
    filters: [Invoice.Paid.equalTo(true)],
  });

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <table>
      {records.map((row) => (
        <tr key={row.customerName}>
          <td>{row.customerName}</td>
          <td>{row.totalAmount}</td>
        </tr>
      ))}
    </table>
  );
};
```

## `usePageFilters`

This hook provides a shared set of filters that multiple chart components can read and modify, enabling coordinated filtering across a page.

### Signature

```ts
function usePageFilters(): UsePageFiltersResult;
```

No parameters.

### Return value (`UsePageFiltersResult`)

```ts
interface UsePageFiltersResult {
  filters: Filter[];
  setFilters: (filters: Filter[]) => void;
  addFilter: (filter: Filter) => void;
  removeFilter: (filter: Filter) => void;
  clearFilters: () => void;
}
```

- `filters`: The current array of active filters.
- `setFilters`: Replaces the entire filter array.
- `addFilter`: Appends a new filter to the array.
- `removeFilter`: Removes a specific filter instance (by reference or identity).
- `clearFilters`: Removes all filters.

Filters are of type `Filter` from the `flowerbi` core package, such as `Column.equalTo(value)` or `Column.greaterThan(value)`.

### Example

```tsx
import { usePageFilters } from 'flowerbi-react';
import { Invoice } from './generated-schema';

const FilterPanel = () => {
  const { filters, addFilter, clearFilters } = usePageFilters();

  const handlePaidOnly = () => {
    addFilter(Invoice.Paid.equalTo(true));
  };

  return (
    <div>
      <button onClick={handlePaidOnly}>Show paid only</button>
      <button onClick={clearFilters}>Reset all filters</button>
      <p>Active filters: {filters.length}</p>
    </div>
  );
};
```

When used in multiple components, all components sharing the same `usePageFilters` call (within the same provider context) will see the same filter array.

## Context Provider

`usePageFilters` relies on a `PageFiltersProvider` that should wrap the components that need shared filters. This provider is exported from `flowerbi-react`.

```tsx
import { PageFiltersProvider } from 'flowerbi-react';

const App = () => (
  <PageFiltersProvider>
    <FilterPanel />
    <Chart />
  </PageFiltersProvider>
);
```

If no provider is present, `usePageFilters` will throw an error.

## Notes

- Both hooks are TypeScript-friendly and provide full type inference from your generated schema.
- The `useFlowerBI` hook automatically re-fetches when the query definition changes (based on deep equality).
- For more complex state management, consider combining `usePageFilters` with `useFlowerBI` by passing the filters into the query.
