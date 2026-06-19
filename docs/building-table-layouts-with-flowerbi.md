---
title: Building Table Layouts with FlowerBI
status: draft
---

# Building Table Layouts with FlowerBI

FlowerBI provides several ways to render query results: the `FlowerBITable` component for a quick HTML table, `Row` and `Column` components for flexible grid layouts, and the ability to directly iterate over the `records` array for complete control. This guide covers all approaches with code samples.

## Prerequisites

- Ensure you have `flowerbi-react` installed:
  ```bash
  npm install flowerbi-react
  ```
- Your backend must expose a query endpoint that accepts the JSON query and returns `QueryResultJson`.

## Using `FlowerBITable`

`FlowerBITable` is a simple component that takes a query and optional configuration and renders a `<table>`.

### Basic Example

```tsx
import { useQuery, FlowerBITable } from 'flowerbi-react';
import { fetch } from './api';
import { Customer, Bug } from './generated-schema'; // generated from your YAML schema

function BugTable() {
  const { records, loading } = useQuery(fetch, {
    select: {
      customer: Customer.CustomerName,
      bugCount: Bug.Id.count(),
    },
    filters: [Workflow.Resolved.equalTo(true)],
  });

  if (loading) return <div>Loading...</div>;

  return <FlowerBITable records={records} />;
}
```

The component automatically uses the keys of the `select` object as column headers. Each record becomes a row.

### Custom Column Headers

You can override the header labels:
```tsx
<FlowerBITable
  records={records}
  headers={['Customer Name', 'Resolved Bugs']}
/>
```

## Using `Row` and `Column` Components

The `Row` and `Column` components from `flowerbi-react` help you create flexible grid-based layouts without a `<table>` element. They work well for dashboard-style views.

```tsx
import { Row, Column } from 'flowerbi-react';

function Dashboard() {
  const { records } = useQuery(fetch, { /* ... */ });

  return (
    <Row>
      <Column size={6}>
        <FlowerBITable records={records} />
      </Column>
      <Column size={6}>
        {/* Other content like a chart */}
      </Column>
    </Row>
  );
}
```

`Column` accepts a `size` prop (1-12) to define the width. By default, `Row` lays out columns in a flex container.

## Custom Table Rendering with Records

If you need full control over the table markup (e.g., custom formatting, conditional styling, derived columns), map over the `records` array directly. This approach works well when you need:

- Custom column ordering or headers
- Conditional formatting (e.g., highlighting cells based on values)
- Mixed content (e.g., embedding charts or links within cells)
- Non-standard table structures (e.g., grouped rows, matrix layouts)

### Worked Example: A Styled Custom Table

Assume you have the following query:

```tsx
import { useQuery } from 'flowerbi-react';
import { Customer, Bug } from './generated/schema';
import { localFetch } from './fetch';

function CustomTableExample() {
    const { records } = useQuery(localFetch, {
        select: {
            customer: Customer.CustomerName,
            bugCount: Bug.Id.count(),
            resolvedCount: Bug.Id.count([Bug.Resolved.equalTo(true)]),
        },
    });

    // `records` is an array of objects like:
    // { customer: string, bugCount: number, resolvedCount: number }

    return (
        <table style={{ borderCollapse: 'collapse', width: '100%' }}>
            <thead>
                <tr>
                    <th>Customer</th>
                    <th>Total Bugs</th>
                    <th>Resolved</th>
                    <th>Resolution Rate</th>
                </tr>
            </thead>
            <tbody>
                {records.map((record) => (
                    <tr key={record.customer}>
                        <td>{record.customer}</td>
                        <td>{record.bugCount}</td>
                        <td>{record.resolvedCount}</td>
                        <td>
                            {record.bugCount > 0
                                ? `${Math.round(
                                      (record.resolvedCount / record.bugCount) * 100
                                  )}%`
                                : 'N/A'}
                        </td>
                    </tr>
                ))}
            </tbody>
        </table>
    );
}
```

### What’s happening:

1. **`useQuery`** returns an object with `records` – an array strongly typed to match your `select` shape.
2. **We map over `records` directly**, rendering a `<tr>` for each item.
3. We can **compute derived values** (here, the resolution rate) that aren't in the raw query result.
4. The **key** attribute uses a unique value per row (here, `customer` – adjust if names aren’t unique).

The result is a fully controlled HTML table with custom styling and logic.

### Handling Empty Records

Always handle the case when the query hasn’t returned data yet or yields no results:

```tsx
if (!records) {
    return <p>Loading…</p>;
}

if (records.length === 0) {
    return <p>No results found.</p>;
}
```

### Using Row/Column Components vs Direct Mapping

The `Row` and `Column` components are designed for **flexible grid layouts** – they don’t produce `<table>` elements. If you need a genuine HTML table (for accessibility, sorting libraries, or column-based alignment), direct mapping is the better choice.

Use `Row`/`Column` when:
- You want a dashboard-like arrangement of independent blocks.
- You don’t need column headers or row/column alignment.

Use direct mapping over `records` when:
- You need a traditional table with headers and cells.
- You want to apply per-cell formatting or calculations.
- You plan to integrate with a table library (e.g., `react-table`, `ag-grid`).

## Pagination and Sorting

`FlowerBITable` itself does not handle pagination or sorting. You can implement these by managing state and re-querying with offset/limit and order-by clauses (if your schema supports them). For simple cases, consider using a library like `react-table` with the records array.

## Summary

- Use `FlowerBITable` for quick, no-frills tables.
- Use `Row` and `Column` for responsive grid layouts.
- For total control, iterate over the records array directly.
- Always check for loading and empty states.

All components work with the strongly typed results from `useQuery`, ensuring type safety in your templates.